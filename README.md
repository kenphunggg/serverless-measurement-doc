# serverless-measurement-doc
Documentation for Serverless Setup and Measurement

## Table of contents
- [Setting up a product ready Kubernetes cluster](#setting-up-a-product-ready-kubernetes-cluster)
- [Setting up a serverless cluster](#setting-up-a-serverless-cluster)
  - [Prerequisite](#prerequisite)
  - [Installing Knative Serving using YAML files](#installing-knative-serving-using-yaml-files)
    - [Install the Knative Serving component](#1-install-the-knative-serving-component)
    - [Install a networking layer](#2-install-a-networking-layer)
    - [Verify the installation](#3-verify-the-installation)
    - [Configure DNS](#4-configure-dns)
  - [Advance configuration](#advance-configuration)
    - [Kourier Gateway](#kourier-gateway)
    - [Activator](#activator)
    - [Check your configuration](#check-your-configuration)


## Setting up a product ready Kubernetes cluster
You can follow [this guild](https://github.com/kenphunggg/kubespray.git) to build your own K8s cluster using `Kubespray`

## Setting up a serverless cluster
In this testbed, we're using [Knative](https://knative.dev/docs/) for building a `Serverless cluster`

### Prerequisite
It is recommended to using [Metallb](https://metallb.io/) to expose a IPV4 address for cluster's `Loadbalancer` Service

If you’re using kube-proxy in IPVS mode, since Kubernetes v1.14.2 you have to enable strict ARP mode.

You can achieve this by editing kube-proxy config in current cluster:

```shell
# see what changes would be made, returns nonzero returncode if different
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl diff -f - -n kube-system

# actually apply the changes, returns nonzero returncode on errors only
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl apply -f - -n kube-system
```

Installation by manifest

```shell
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-native.yaml
```

In order to assign an IP to the services, MetalLB must be instructed to do so via the `IPAddressPool` CR.

All the IPs allocated via `IPAddressPools` contribute to the pool of IPs that MetalLB uses to assign IPs to services.

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - <your_ipAddresspool>  # Example: 192.168.17.1-192.168.17.250
```

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

#### 4. Configure DNS

You can configure DNS to prevent the need to run curl commands with a host header.

Knative provides a Kubernetes Job called default-domain that configures Knative Serving to use sslip.io as the default DNS suffix.

```shell
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.18.1/serving-default-domain.yaml
```

### Advance configuration

#### Kourier Gateway

```shell
# replicate 3scale-gateway pod to 3 replicas
kubectl -n kourier-system patch deploy 3scale-kourier-gateway --patch '{"spec":{"replicas":3}}'
kubectl -n kourier-system patch deploy 3scale-kourier-gateway --patch '{"spec":{"template":{"spec":{"affinity":{"nodeAffinity":{"requiredDuringSchedulingIgnoredDuringExecution":{"nodeSelectorTerms":[{"matchExpressions":[{"key":"kubernetes.io/hostname","operator":"In","values":["master-node", "cloud-node", "edge-node"]}]}]}}}}}}}'

# use local gateway for every request
kubectl -n kourier-system patch service kourier --patch '{"spec":{"internalTrafficPolicy":"Local","externalTrafficPolicy":"Local"}}'
kubectl -n kourier-system patch service kourier-internal --patch '{"spec":{"internalTrafficPolicy":"Local"}}'
```

#### Activator

```shell
# replicate activator pod to 3 replicas
kubectl -n knative-serving patch deploy activator --patch '{"spec":{"replicas":3}}'
kubectl -n knative-serving patch deploy activator --patch '{"spec":{"template":{"spec":{"affinity":{"nodeAffinity":{"requiredDuringSchedulingIgnoredDuringExecution":{"nodeSelectorTerms":[{"matchExpressions":[{"key":"kubernetes.io/hostname","operator":"In","values":["master-node", "cloud-node", "edge-node"]}]}]}}}}}}}'
```

#### Check your configuration

```shell
thai@master-node ~/ken/serverless-measurement-doc
 $ kubectl -n knative-serving get pod -o wide | grep activator
activator-6f5448b646-9sfzp                1/1     Running     0          4m15s   10.233.113.141   master-node   <none>           <none>
activator-6f5448b646-bkw8m                1/1     Running     0          3m55s   10.233.77.68     edge-node     <none>           <none>
activator-6f5448b646-gdf8h                1/1     Running     0          4m17s   10.233.99.12     cloud-node    <none>           <none>
```

```shell
thai@master-node ~/ken/serverless-measurement-doc
 $ kubectl -n kourier-system get pod -o wide
NAME                                      READY   STATUS    RESTARTS   AGE     IP               NODE          NOMINATED NODE   READINESS GATES
3scale-kourier-gateway-5886fc6dbd-4tsbz   1/1     Running   0          8m31s   10.233.99.11     cloud-node    <none>           <none>
3scale-kourier-gateway-5886fc6dbd-kq77k   1/1     Running   0          8m31s   10.233.77.66     edge-node     <none>           <none>
3scale-kourier-gateway-5886fc6dbd-tcf8v   1/1     Running   0          8m31s   10.233.113.139   master-node   <none>           <none>
```

> [!IMPORTANT]
> Make sure your cluster have exactly one activator and one gateway each node





