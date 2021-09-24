# Kubernetes 101
An effort to brush up kubernetes skills. This repo contains the all the manifest files.

## Tasks

* Create an infra repo where all this work will be contained
    - Add access to the whole team (for reviewing purposes)
* Create a 3-node cluster
* Deploy 3 distinct webservers into the cluster (e.g. Nginx)
    - Each of the webservers should display different content (e.g. different `index.html` static page)
* Expose these webservers, make each of them accessible from outside the cluster
    - via NodePort
    - via Ingress
    - via LoadBalancer
* Expose a service, that will forward the requests to the existing webservers
    - in 30% of the requests target webserver-1
    - in 45% of the requests target webserver-2
    - in 25% of the requests target webserver-3
* Deploy Prometheus and Grafana for monitoring the workload
   - Ensure you collect pod metrics like pod CPU/memory utilization

### Creating 3-node cluster 

I'll be using `minikube` in this example. To initialize a kube cluster with 3 nodes, we run the following command.

```bash
$ minikube start --nodes 3 
```

### Deploying three pods to the cluster 
We start by creating the 3 `index.html` files for each nginx server. You can find them under /webserver directory. Once things are in place, we proceed with Dockerfiles. Next up is the `manifest` file, where we define our deployments and services.

you can deploy this manifest file using the following `kubectl` command
```bash
$ kubectl apply -f ./webserver/webserver.yaml
```

once the recourses are created, you can check the status using `kubectl`.
```bash
$ kubectl get pods            # list out all the pods deployed
```
Since we have used a `Nodeport` type of service for this section of the, we need to get the IP address of the node to access the our service. Since we are on minikube, we can take help of minikube's cli.
```bash
$ minikube ip                 # returns the ip of the cluster
```
We have exposed the service on port `30001`, thus we can access the service on `$(minikube ip):30001`
```bash
$ curl $(minikube ip):30001   # hits one of the pods and returns HTML
```


### Trying out different ways of exposing service

In previous section we used `NodePort` type of service to expose our service to the node. But in the production, it's advice to use a ingress to expose your cluster to outside world. Let's take a look at how to use an ingress in the kubernetes.

We start by enabling the _ingress addon_ on minikube. Next we define the ingress manifest file under `/ingress`. We have a very simple config here, just routing the request on `kubernetes-101.info` to the service `webserver-service`. One the manifest file is applied to the cluster and ingress resource is created, we'll have to make a change in out `/etc/hosts` file to add the ip address of the cluster and bind it to domain `kubernetes-101.info`. Now you can hit `kubernetes-101.info` and serve the HTML content from one of the pod.

### Traffic Management 

For our next task we need to divide the traffic between the pods and manage load on each pod. This kind of scenario is helpful in __canary deployments__. For this, we take help from a Service mesh called [Istio](https://istio.io/latest/). We need to spin a cluster with memory allocation of about `8192mb` and `4` units of cpu. To do so in minikube, you can run the following command

```bash
$ minikube start --nodes 3 --memory=8192 --cpus=4 
```
Next we'll need to install the `istioctl` to install isto on our cluster. You can follow the installation guide available [here](https://istio.io/latest/docs/setup/getting-started/). This will create a namespace of `istio-system` and install default gateway. 

Once istio is installed on the cluster, nest step is to create `gateway`,`virtual Service` and define `destination rules` for this `virtual service` 

Before deploying our services, we'll need to add a label to our `default` namespace so that istio and add `envoy sidecars` to all our pods. Here is the command 
```bash
$ kubectl label namespace default istio-injection=enabled
```

Now we are ready to deploy our services and create other resources 
```bash
$ kubectl apply -f webservers/webserver-clusterIP.yaml 
$ kubectl apply -f istio/gateway.yaml
$ kubectl apply -f istio/virtualService.yaml 
```

To expose the service, we'll have top grab the IP of cluster and port on which the gateway is available.

```bash
$ export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')

$ export INGRESS_HOST=$(minikube ip)

$ curl http://$INGRESS_HOST:$INGRESS_PORT
```


### Monitoring
we need to deploy Prometheus and Grafana for to setup monitoring. Since we need basic monitoring, we'll use the prometheus helm charts. 

```bash
$ helm install prometheus-stack prometheus-community/kube-prometheus-stack
```

To access the Grafana dashboard, we can port forward the grafan server which is running on port 80 and map to a free port, e.g. 3000.
```bah
$ kubectl port-forward service/prometheus-stack-grafana 3000:80
```

The prometheus-stack come with predefined dashboards which includes basic CPU, Memory and network monitoring.