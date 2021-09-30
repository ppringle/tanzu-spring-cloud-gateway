# tanzu-spring-cloud-gateway

This details how to install the Spring Cloud Gateway (SCGW)  into a developer instance of k8s, specifically
Minikube.

## Prerequisites
- kubectl
- minikube
- kustomize
- helm
- Tanzu Network Account (required for access to the download artifacts for Spring Cloud Gateway for k8s)

These installation instructions are specifically for Spring Cloud Gateway (v.1.0.0), and therefore
only valid for a k8s cluster running versions **1.17 - 1.20**.

## Minikube
To create an instance of minikube running k8s 1.20, run the following command:

```
minikube start --memory 8192 --cpus 4 --vm-driver=hyperkit --profile spring-cloud-gateway --kubernetes-version=v1.20.0
```
The above creates a cluster called **spring-cloud-gateway** running **v.1.20.0** of k8s.

You can check the version of the k8s server with the following command:

```
kubectl version
```

## Download Spring Cloud Gateway for Kubernetes

1. Download Spring Cloud Gateway (v.1.0.0) from the [Tanzu Network](https://network.pivotal.io/products/spring-cloud-gateway-for-kubernetes#/releases/835308).

2. Extracting the downloaded tar.gz file `tar zxf spring-cloud-gateway-k8s-1.0.0.tgz` you will find the files shown 
   below.
```text
.
├── helm
│   └── spring-cloud-gateway-1.0.0.tgz
├── images
│   ├── gateway-1.0.0.tar
│   └── scg-operator-1.0.0.tar
└── scripts
    ├── install-spring-cloud-gateway.sh
    └── relocate-images.sh

3 directories, 5 files
```
3. Inspect the contents of the `images` folder. Notice it contains two container images packaged as `.tar` files.
   The script located in `scripts/relocate-images.sh` is used to publish `gateway-1.0.0.tar` and
   `scg-operator-1.0.0.tar` images into a private container registry accessible from the kubernetes cluster you
   deploy the spring cloud gateway on.

4. Inspect the `helm` chart folder and notice that it contains a gzipped helm chart. This will
   be used to install the spring cloud gateway operator.

5. Inspect the `scripts` folder notice that it contains two shell scripts, one for publishing the private
   container images to a registry, and the second scripts runs a helm to install the operator. 

## Load the SCGW container images into Minikube 

In a production deployment, we would run the scripts/relocate-images.sh to publish the SCGW container images 
to a private container registry accessible from the k8s cluster that will be running SCGW. 
Here, since we are using Minikube, we only need to load the container images into the docker environment being 
used  by Minikube.

1. To do the above, run the following commands:

```
eval $(minikube docker-env -p spring-cloud-gateway)
minikube image load ./images/scg-operator-1.0.0.tar -p spring-cloud-gateway
minikube image load ./images/gateway-1.0.0.tar -p spring-cloud-gateway
```

2. Check your local docker images `docker images` and look for spring cloud gateway images. You should output similar
to below.

```text
docker images 
REPOSITORY                                                  TAG        IMAGE ID       CREATED         SIZE
dev.registry.pivotal.io/spring-cloud-gateway/scg-operator   1.0.0      35abc5be0038   41 years ago    219MB
dev.registry.pivotal.io/spring-cloud-gateway/gateway        1.0.0      0c2293e5647c   41 years ago    289M
```

## Install Spring Cloud Gateway Operator

1. Create a namespace to install SCGW into using the command:
```
kubectl create namespace spring-cloud-gateway
```
   
2. The SCGW operator is installed using Helm. We will need to create a `scg-image-values.yaml`file pointing at the 
registry containing the SCGW container images. The `scg-image-values.yaml` file is normally generated when the 
`scripts/relocate-images.sh` is run. We did not run the script, as we won't be opting to use a private container 
registry (as we have already loaded the images into the Minikube docker environment manually). Instead, we will 
generate the `scg-image-values.yaml` manually. In the `helm` folder, create a file called `scg-image-values.yaml` 
with the content below:

```yaml
gateway:
   image: "dev.registry.pivotal.io/spring-cloud-gateway/gateway:1.0.0"
scg-operator:
   image: "dev.registry.pivotal.io/spring-cloud-gateway/scg-operator:1.0.0"
```

In the previous section, we loaded the SCGW container images to Minikube which makes the images available to our
k8s cluster.

4. Execute script: `./scripts/install-spring-cloud-gateway.sh`. We should see output similar to that outlined below:

```text
Installing Spring Cloud Gateway...
chart tarball: spring-cloud-gateway-1.0.0.tgz
chart name: spring-cloud-gateway
Release "spring-cloud-gateway" does not exist. Installing it now.
NAME: spring-cloud-gateway
LAST DEPLOYED: Thu Mar  4 14:18:52 2021
NAMESPACE: spring-cloud-gateway
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
This chart contains the Kubernetes operator for Spring Cloud Gateway.
Install the chart spring-cloud-gateway-crds before installing this chart
Finished installing Spring Cloud Gateway
```

5. Execute command:  `kubectl get all -n spring-cloud-gateway` should output content similar to that shown below:

```text
NAME                               READY   STATUS    RESTARTS   AGE
pod/scg-operator-679f77dbb-8w9bt   1/1     Running   0          3m42s

NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/scg-operator   ClusterIP   10.96.32.243   <none>        80/TCP    3m42s

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/scg-operator   1/1     1            1           3m42s

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/scg-operator-679f77dbb   1         1         1       3m42s
```

6. The SCGW operator installed three k8s CRDs. Execute the command:
   `kubectl get crds` and you will see all the created custom CRDs shown below. These CRDs will be used to configure
   the gateway.

```text
 NAME                                              CREATED AT
springcloudgatewaymappings.tanzu.vmware.com       2021-03-04T05:47:20Z
springcloudgatewayrouteconfigs.tanzu.vmware.com   2021-03-04T05:47:20Z
springcloudgateways.tanzu.vmware.com              2021-03-04T05:47:20Z
```

## Define a Gateway Instance

1. Inspect the `resources/my-gateway.yml` file. It contains the YAML shown below which defines a Spring Cloud Gateway 
instance.

```yaml
apiVersion: "tanzu.vmware.com/v1"
kind: SpringCloudGateway
metadata:
  name: my-gateway
```

2. Execute the command `kubectl apply -f resources/gateway-config.yaml` which will submit a request to the cluster to 
deploy an instance of Spring Cloud Gateway.

3. Execute the command `kubectl get all`. This should make visible a pod of the Spring Cloud Gateway running or being 
launched in the cluster's default namespace as shown in the output below.

```text
NAME               READY   STATUS    RESTARTS   AGE
pod/my-gateway-0   1/1     Running   0          6h49m

NAME                          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/kubernetes            ClusterIP   10.96.0.1      <none>        443/TCP    6h59m
service/my-gateway            ClusterIP   10.96.222.12   <none>        80/TCP     6h49m
service/my-gateway-headless   ClusterIP   None           <none>        5701/TCP   6h49m

NAME                          READY   AGE
statefulset.apps/my-gateway   1/1     6h49m
```

4. Inspect the `resources/gateway-routes-config.yaml` file. It contains the YAML shown below which defines the routes:

```yaml
apiVersion: tanzu.vmware.com/v1
kind: SpringCloudGatewayRouteConfig
metadata:
  name: my-gateway-routes
spec:
  routes:
    - id: test-route
      uri: https://github.com
      predicates:
        - Path=/github/**
      filters:
        - StripPrefix=1
```

5. Execute the command `kubectl apply -f resources/gateway-routes-config.yaml`. This will submit a request to define 
new routes.

6. Inspect the `resources/gateway-mapping-config.yaml`. It contains the YAML shown below which maps the routes to the
gateway instance:

```yaml
apiVersion: tanzu.vmware.com/v1
kind: SpringCloudGatewayMapping
metadata:
  name: test-gateway-mapping
spec:
  gatewayRef:
    name: my-gateway
  routeConfigRef:
    name: my-gateway-routes
```

7. Execute the command `kubectl apply -f resources/gateway-mapping-config.yaml`. This will submit a request to 
define new mappings which map specific gateway routes to a given gateway instance.

8. As there is no LoadBalancer configured within the cluster, we will use port forwarding to test the gateway. Run 
the command `kubectl port-forward service/my-gateway 8667:80`. This should display output similar to that shown 
below: 

```
Forwarding from 127.0.0.1:8667 -> 8080
Forwarding from [::1]:8667 -> 8080
Handling connection for 8667
Handling connection for 8667
```

9. Using a browser, navigate to `http://localhost:8667/github`. You should then be forwarded to the Github website. 
In this example, the request is going through SCGW, which is then forwarding the request to github.com based on the 
configured routing rules.