---
title: Implementing different types of load balancers in GKE
description: Deploying different kinds of load balancers like HTTPS-Internal and External LB, TCP- Internal and External LB using GKE Ingress 
author: valavan007
tags: GKE Ingress, Load Balancers, Internal Load Balancers
date_published: 2020-04-16
---

This guide provides a walk through of deploying a simple helloworld application on GKE and using the differnet types of Load Balancers which cater to different use cases. GKE uses ingress controller  to deploy Load Balancers on the GKE Platform. GKE ingress controllers currrently supports multiple load balancers like External HTTPS, Internal HTTPS, TCP/UDP External , TCP/UDP Internal and Container Native Load Balancer which are based on Network Endpoint Groups(NEG). More details about the benefits of using Ccontainer Native Load Balancing and NEG can be found [here](https://cloud.google.com/kubernetes-engine/docs/concepts/container-native-load-balancing)

This guide shows how to build a simple hello world application running in a single GKE cluster and exposing the service through the following load balancer option. Each type of load balancer will be implemented under a different namespace 

- Deploy an external HTTPS load balancer through standard GKE Ingress
- Deploy an external HTTPS load balancer using GKE ingress & Container Native Load balancing 
- Deploy an internal HTTPS load balancer using GKE ingress & Container Native Load balancing 
- Deploy an external TCP Load balancer 
- Deploy an internal TCP Load balancer 

### Prerequsites 

To perform the steps in the tutorials you will need a 

- A GKE cluster with version 1.15 and above 
- Clone the git repository(https://github.com/GoogleCloudPlatform/community)
- Navigate to `cd tutorials/implemeting-diffferent-types-of-loadbalancers-in-GKE` to use the K8S Manifest files 

### Preparing GCP environment 

1.  Before the next step, run `gcloud auth list` to ensure that your shell session is authenticated.

    If not you can use the `gcloud auth login` command to authenticate.
    
2.  In your shell, run the following commands to set the configuration context for `gcloud`:
```
export PROJECT=$(gcloud info --format='value(config.project)')
export ZONE=us-central1-f
gcloud config set compute/zone [ZONE_NAME]
```
### Creating a Test GKE Cluster 

* Open Cloudshell to create a GKE Cluster 
```
gcloud beta container clusters create hellow --release-channel=rapid --enable-ip-alias --network=default
```
Note: Internal HTTPS LB is currently in Beta and is supported on Rapid Release GKE versions 

### Sample Application 
The sample application that is shown here consists of a simple Deployment that deploys a HelloWorld page  and exposes the service on port 8080. This application is not accessible outside the cluster. 

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: basic-ingress
spec:
  selector:
    matchLabels:
      run: web
  template:
    metadata:
      labels:
        run: web
    spec:
      containers:
      - image: gcr.io/google-samples/hello-app:1.0
        imagePullPolicy: IfNotPresent
        name: web
        ports:
        - containerPort: 8080
          protocol: TCP
```
```
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: basic-ingress
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    run: web
  type: ClusterIP
```
###  Deploying an external HTTPS load balancer through standard GKE Ingress
This application can be exposed through standard Ingress controller on GKE by deploying the following Ingress Resource.

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: basic-ingress
  namespace: basic-ingress
spec:
  backend:
    serviceName: web
    servicePort: 8080
```
All of the related files for deploying are available in their separate folder and can be implemented using the following commands in a dedicated namespace 

```
cd basic-ingress
kubectl create ns basic-ingress
kubectl ns basic-ingress
kubectl apply -f web-deployment.yaml
kubectl apply -f web-service.yaml
kubectl apply -f basic-ingress.yaml
```

Verify that the deployment is complete once the external IP address is visible
```
kubectl get ingress basic-ingress --watch
NAME            HOSTS   ADDRESS         PORTS   AGE
basic-ingress   *       <ext-ip-addr>   80      174m

curl <ext-ip-addr>
```

### Deploy an external HTTPS load balancer using GKE ingress & Container Native Load balancing 
In the next step, the same application can be exposed through Ingress controller on GKE but as a Network Endpoint Group instead of using Managed Instance Group type. Network endpoint groups provide Improved network performance, Increased visibility with Pods acting as direct backends instead of GKE Nodes. The Deployment YAML and the ingress YAML aren't changed, only the service definition is changed with an added annotation  
```
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: cnlb
  annotations:
    cloud.google.com/neg: '{"ingress": true}'
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    run: web
  type: ClusterIP
```
To deploy the entire setup in a new namespace called `cnlb` on the same cluster, follow th steps below
```
cd cnlb-ext-ingress
kubectl create namespace cnlb
kubectl ns cnlb
kubectl apply -f web-deployment.yaml
kubectl apply -f web-service.yaml
kubectl apply -f cnlb-ingress.yaml
```
Verify that the deployment is complete once the external IP address is visible
```
kubectl get ingress cnlb-ingress 
NAME            HOSTS   ADDRESS         PORTS   AGE
basic-ingress   *       <ext-ip-addr>   80      174m

curl <ext-ip-addr>
```

###  Deploy an internal HTTPS load balancer using GKE ingress & Container Native Load balancing
Google Cloud recently announced the availability of Internal HTTPS Load balancer in beta. Prior to the announcement, internal TCP and 
UDP load balancer served more as a transparent proxy forwarding packets received from the internal network. As a result, the incoming traffic will be proxied through the load balancer while the return traffic was sent to the network gateway instead of the load balancer proxy. This introduced added complexity where in more changes were required to the underlying operating system if you are using a non-GCP OS image.

Few important considerations before using Internal HTTPS Load Balancer 
- Creating a proxy-subnet
- Allowing health check from GCP keep-alive IP ranges

The additional steps of creating a proxy subnet which will be used by the HTTPS load balancer to pass it to the region VPC network
```
  gcloud compute networks subnets create proxy-only-subnet \
  --purpose=INTERNAL_HTTPS_LOAD_BALANCER \
  --role=ACTIVE \
  --region=us-central1 \
  --network=default \
  --range=10.124.0.0/23

  gcloud compute firewall-rules create fw-allow-proxies \
  --network=default \
  --action=allow \
  --direction=ingress \
  --source-ranges=10.124.0.0/23 \
  --target-tags=load-balanced-backend \
  --rules=tcp:80,tcp:443,tcp:8000
```

To make the service available through Internal HTTPS load balancer, changes are required for both service and Ingress resources in the form of additonal annotations
```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: ilb-ingress
  namespace: ilb-ingress
  annotations:
    kubernetes.io/ingress.class: "gce-internal"
spec:
  backend:
    serviceName: web
    servicePort: 8080
```
```
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: ilb-ingress
  annotations:
    cloud.google.com/neg: '{"ingress": true}'
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    run: web
  type: NodePort
```
The internal https load balancer relies on Network Endpoint Groups and doesnt support the Managed Instance Group method of connecting to the pods. To deploy from the repo, follow the steps below.
```
cd ../ilb-ingress
kubectl create namespace ilb-ingress 
kubectl ns ilb-ingress
kubectl apply -f web-deployment.yaml
kubectl apply -f web-service.yaml
kubectl apply -f ilb-ingress.yaml
```
Verify that the deployment is complete once the internal IP address is visible
``` 
kubectl get ingress ilb-ingress  (doesnt show the IP)
NAME          HOSTS   ADDRESS        PORTS   AGE
lb-ingress   *       <internal-ip>   80      68s
```
To verify that you can reach the service, you need a VM deployed on the same VPC to test 
```
gcloud instance create temp-vm 
gcloud ssh temp-vm 
$ curl <internal-ip>
```

### Deploy an external TCP Load balancer 
Deploying a service through external TCP Load Balancer is quite straightforward with just naming the service type as `LoadBalancer`
```
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: lb
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    run: web
  type: LoadBalancer
  ```

To deploy the TCP Load balancer through the repo, follow the steps below
```
cd ../tcp-lb
kubectl create namespace tcp-lb
kubectl ns tcp-lb
kubectl apply -f web-deployment.yaml
kubectl apply -f web-service.yaml
```
Verify that the deployment is complete once the external IP address is visible
```
kubectl get service
NAME   TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)          AGE
web    LoadBalancer   10.15.245.245   104.154.176.232   8080:30665/TCP   6m58s

curl <LB-IP>:8080  
```


### Deploy an internal TCP Load balancer 
This section will focus on deploying the service through Internal LB. There is no ingress resource required, as GCP will provision load balancer based on the spec `type: LoadBalancer` and `annotations` in the Service definition. Only change that will be required is the annotations in the metadata which triggers GCP to deploy and internal TCP Load balancer 
```
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: tcp-ilb
  annotations:
    cloud.google.com/load-balancer-type: "Internal"
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    run: web
  type: LoadBalancer
```
To deploy the TCP Load balancer through the repo, follow the steps below
```
cd ../tcp-ilb
kubectl create namespace tcp-lb
kubectl ns tcp-lb
kubectl apply -f web-deployment.yaml
kubectl apply -f web-service.yaml
```
Verify that the deployment is complete once the external IP address is visible
```
kubectl get service
NAME   TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
web    LoadBalancer   10.15.241.39   <internal-ip>   8080:31721/TCP   102s
```
To verify that you can reach the service, you need a VM deployed on the same VPC to test 
```
gcloud ssh temp-vm 
$ curl <internal-ip>
```    
