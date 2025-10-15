# serverless-measurement-doc
Documentation for Serverless Setup and Measurement

## Table of contents

- [Testbed design](#testbed-design)
- [Setting up a product ready Kubernetes cluster](#setting-up-a-product-ready-kubernetes-cluster)
- [Setting up a serverless cluster](#setting-up-a-serverless-cluster)
  - [Installing Knative Serving using YAML files](#installing-knative-serving-using-yaml-files)
    - [Install the Knative Serving component](#1-install-the-knative-serving-component)
    - [Install a networking layer](#2-install-a-networking-layer)
    - [Verify the installation](#3-verify-the-installation)
  - [Advance configuration](#advance-configuration)
    - [Kourier Gateway](#kourier-gateway)
    - [Activator](#activator)
    - [Check your configuration](#check-your-configuration)
- [Network configuration](#network-configuration)
- [Setting up Prometheus for measuring](#setting-up-prometheus-for-measuring)
  - [Install Prometheus](#install-prometheus)
  - [Running Prometheus](#running-prometheus)

## Testbed design

![testbed design](./images/testbed_des.png)

In this testbed, we build up three nodes which is `master-node`, and two worker nodes (`cloud-node` and `edge-node`). `mater-node` will be the `control-plane` for `Kubernetes` cluster. It will generate requests and the measurement will be run right there. We will emulate latency and bandwidth between `master-node` and two other worker nodes. Functions will be served in two worker nodes and we will run `Prometheus` server in these nodes for measurement.

## Setting up a product ready Kubernetes cluster
You can follow [this guild](https://github.com/kenphunggg/kubespray.git) to build your own K8s cluster using `Kubespray`

## Setting up a serverless cluster
In this testbed, we're using [Knative](https://knative.dev/docs/) for building a `Serverless cluster`

### Installing Knative Serving using YAML files

#### 1. Install the Knative Serving component

To install the Knative Serving component:

```shell
# Install the required custom resources by running the command
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.18.1/serving-crds.yaml

# Install the core components of Knative Serving by running the command
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.18.1/serving-core.yaml
```

#### 2. Install a networking layer

```shell
# Install the Knative Kourier controller by running the command
kubectl apply -f https://github.com/knative/net-kourier/releases/download/knative-v1.18.0/kourier.yaml

# Configure Knative Serving to use Kourier by default by running the command:
kubectl patch configmap/config-network \
  --namespace knative-serving \
  --type merge \
  --patch '{"data":{"ingress-class":"kourier.ingress.networking.knative.dev"}}'
```

Fetch the External IP address or CNAME by running the command:

```shell
kubectl --namespace kourier-system get service kourier
```

#### 3. Verify the installation

Monitor the Knative components until all of the components show a STATUS of Running or Completed. You can do this by running the following command and inspecting the output:

```shell
kubectl get pods -n knative-serving
```

Example output:

```shell
NAME                                      READY   STATUS    RESTARTS   AGE
3scale-kourier-control-54cc54cc58-mmdgq   1/1     Running   0          81s
activator-67656dcbbb-8mftq                1/1     Running   0          97s
autoscaler-df6856b64-5h4lc                1/1     Running   0          97s
controller-788796f49d-4x6pm               1/1     Running   0          97s
webhook-859796bc7-8n5g2                   1/1     Running   0          96s
```

### Advance configuration

#### Kourier Gateway

```shell
# replicate 3scale-gateway pod to 3 replicas
kubectl -n kourier-system patch deploy 3scale-kourier-gateway --patch '{"spec":{"replicas":1}}'
kubectl -n kourier-system patch deploy 3scale-kourier-gateway --patch '{"spec":{"template":{"spec":{"affinity":{"nodeAffinity":{"requiredDuringSchedulingIgnoredDuringExecution":{"nodeSelectorTerms":[{"matchExpressions":[{"key":"kubernetes.io/hostname","operator":"In","values":["master-node"]}]}]}}}}}}}'

# use local gateway for every request
kubectl -n kourier-system patch service kourier --patch '{"spec":{"internalTrafficPolicy":"Local","externalTrafficPolicy":"Local"}}'
kubectl -n kourier-system patch service kourier-internal --patch '{"spec":{"internalTrafficPolicy":"Local"}}'
```

#### Activator

```shell
# replicate activator pod to 3 replicas
kubectl -n knative-serving patch deploy activator --patch '{"spec":{"replicas":1}}'
kubectl -n knative-serving patch deploy activator --patch '{"spec":{"template":{"spec":{"affinity":{"nodeAffinity":{"requiredDuringSchedulingIgnoredDuringExecution":{"nodeSelectorTerms":[{"matchExpressions":[{"key":"kubernetes.io/hostname","operator":"In","values":["master-node"]}]}]}}}}}}}'
```

#### Config-map

```shell
kubectl patch configmap config-features \
  -n knative-serving \
  --type merge \
  -p '{"data":{"kubernetes.podspec-nodeselector":"enabled"}}'

kubectl patch configmap config-features \
  -n knative-serving \
  --type merge \
  -p '{"data":{"kubernetes.podspec-fieldref":"enabled"}}' 

```

#### Systemd-resolved

It is recommended to configure `/etc/systemd/resolved.conf` to get access to pods without a `curl_pod`. Look at the field `Domains` and adjust it like below

```bash
Domains=default.svc.cluster.local svc.cluster.local cluster.local
```

Then you can apply changes

```bash
systemctl restart systemd-resolved
```

#### Check your configuration

```shell
thai@master-node ~/ken/serverless-measurement-doc
 $ kubectl -n knative-serving get pod -o wide | grep activator
activator-6f5448b646-9sfzp                1/1     Running     0          4m15s   10.233.113.141   master-node   <none>           <none>
```

```shell
thai@master-node ~/ken/serverless-measurement-doc
 $ kubectl -n kourier-system get pod -o wide
NAME                                      READY   STATUS    RESTARTS   AGE     IP               NODE          NOMINATED NODE   READINESS GATES
3scale-kourier-gateway-5886fc6dbd-tcf8v   1/1     Running   0          8m31s   10.233.113.139   master-node   <none>           <none>
```

> [!IMPORTANT]
> Make sure your cluster have exactly one activator and one gateway each node

## Network configuration

In this work, we use [traffic-control](https://github.com/kenphunggg/traffic-control.git) for emulating latency and bandwidth between nodes.

## Setting up Prometheus for measuring

### 1. Install prometheus

First, you need to install `helm`

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

Now you can install `Prometheus` using `helm`

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm repo add stable https://charts.helm.sh/stable

helm repo update
```

Search for `kube-prometheus-stack `
```bash
helm search repo prometheus |egrep "stack|CHART"

NAME                                              	CHART VERSION	APP VERSION	DESCRIPTION                                       
prometheus-community/kube-prometheus-stack        	78.2.1       	v0.86.0    	kube-prometheus-stack collects Kubernetes manif...
prometheus-community/prometheus-stackdriver-exp...	4.12.1       	v0.18.0    	Stackdriver exporter for Prometheus               
stable/stackdriver-exporter                       	1.3.2        	0.6.0      	DEPRECATED - Stackdriver exporter for Prometheus  
```

Pull version you have found above
```bash
helm pull prometheus-community/kube-prometheus-stack --version 78.2.1
tar -xzf kube-prometheus-stack-78.2.1.tgz
cp kube-prometheus-stack/values.yaml values-prometheus.yaml
```

You can change default password when logging in

```bash
adminPassword: pass
```
Install `Prometheus` components

```bash
kubectl create ns monitoring
helm -n monitoring install prometheus-grafana-stack -f values-prometheus-clusterIP.yaml kube-prometheus-stack
kubectl -n monitoring get all
```

Config network for access in localhost
```bash
kubectl port-forward -n monitoring svc/prometheus-grafana-stack-k-prometheus 9090:9090

kubectl get configmap kube-proxy -n kube-system -o yaml | \
  sed 's/metricsBindAddress: 127.0.0.1:10249/metricsBindAddress: 0.0.0.0:10249/' | \
  kubectl apply -f -

kubectl rollout restart daemonset -n kube-system kube-proxy
```


## Measurement

Follow our our guild for measuring [here](https://github.com/kenphunggg/serverless-measurement.git)









