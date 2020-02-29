# Klib Istio Quickstart
Following this guide you will
1. Set up a kubernetes cluster with minikube on the Hyper-V Hypervisor
2. Inject Istio into the Cluster
3. Deploy our [Aries Example Webservice](https://github.com/klibio/io.klib.aries.example) into the Cluster
4. Watching the traffic with the Kiali Dashboard while consuming the Webservice locally

## Prerequisites

One Windows we recommend the free [Chocolatey package Manager](https://chocolatey.org/), which will make the installations easier.

1. Activate the Hyper-V hypervisor via the windows feature menu
2. Install kubectl via
    ````
    choco install kubernetes-cli
    ````
3. Install minikube via 
    ````
    choco install minikube
    `````
4. Configure minikube to use the Hyper-V hypervisor
    ````
    minikube config set vm-driver hyperv
    ````
5. Install istioctl via
    ````
    choco install istioctl
    ````
6. Start minikube va
    ````
    minikube start --memory=16384 --cpus=4
    ````
    The first start may take a while. If the command failes while configuring kubectl follow steps 7 to 9.
7. Connect to the minikube vm via the Hyper-V Manager and login with root. Execute
    ````
    shutdown 0
    ````
8. Create a new virtual switch with the Hyper-V Manager which shares the internet connection with a bridge.
9. Allocate the reated virtual switch to the Hyper-V VM of minikube.
10. Execute
    ````
    minikube start
    ````

We now have a fully functioning kubernetes cluster inside the minikube environment.

## Inject Istio into the cluster and install the kiali pod

Now we are going to inject Istio into our minikube cluster, as well as visualizing it with Kiali.

1. Inject Istio with kiali via
    ````
    istioctl manifest apply --set values.kiali.enabled=true 
    ````
    Please note that the default manifest does not include any jaeger or grafana configurations. You will need to apply them seperately, or use the demo profile via
    ````
    istioctl manifest apply --set profile=demo
    ````
    The demo profile includes Kiali, Jaeger and Grafana, as well as a Kiali Secret to Login to the Kiali dashboard. If you choose this option can skip step 4.
2. You can Watch the services start up via
    ````
    kubectl get all -n istio-system
    ````
    Please note that the istio-ingressgateway will not get an External-IP, instead  it will be `<pending>`. This is a result of the minikube environment not having support for an external loadbalancer. We will access the cluster with the NodePort the istio-ingressgateway provides.
3. Now we will enable the sidebar injection for our default namespace via
    ````
    kubectl label namespace default istio-injection=enabled
    ````
4. To be able to log in to the Kiali Dashboard we need to provide a Kiali Secret. This repo provides an example Secret with the username and password combination: 
    ````
    usr = admin; pw = admin
    ````
    Apply this secret via
    ````
    kubectl apply -f .\samples\kiali-secret.yaml
    ````

## Deploy and consume the webservice
Deploying the webservice is quite easy. Execute

    kubectl apply -f .\samples\aries\aries.yaml

And

    kubectl apply -f .\samples\aries\aries-gateway.yaml

This will crate a worker pod, as well as an aries-ingressgateway for this pod. To make it reachable from the outside we also need a virtual service which links the istio-ingressgateway with the aries-ingressgateway.

To consume the webservice we need the ip and the port on which our service is exposed.

1. Get the ip via
    ````
    minikube ip
    ````
2. Get the NodePort http2 Port via
    ````
    kubectl get svc istio-ingressgateway -n istio-system -o jsonpath='{.spec.ports[?(@.name==\"http2\")].nodePort}'
    ````
3. Open your browser and connect to `<minikube ip>:<NodePort>/Jon`, which should return `Hello Jon`.
4. Monitor the traffic by logging into Kiali via
    ````
    istioctl dashboard kiali
    ````
    You might need to refresh the webpage to get new traffic.