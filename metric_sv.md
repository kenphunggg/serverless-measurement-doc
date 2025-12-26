# Metrics Server
Kubernetes Metrics Server: Installation and Configuration Guide

## Install Metrics Server using helm-chart
Create folder for installation

```bash
mkdir metric-server
cd metric-server/
```

Install Metrics Server

```bash
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm search repo metrics-server
helm pull metrics-server/metrics-server --version 3.13.0
tar -xzf metrics-server-3.13.0.tgz
helm install metric-server metrics-server -n kube-system
```

Metrics Server can be crashed. [See mroe](https://github.com/kubernetes-sigs/metrics-server/issues/278)

```bash
 $ kubectl get pods -n kube-system | grep metrics-server
metric-server-metrics-server-994798949-dh6bh   0/1     Running   0                3m40s
```

You can fix it by making some changes in `metric-server-metrics-server` deployment

```bash
kubectl patch deployment metric-server-metrics-server -n kube-system --type='json' -p='[
  {
    "op": "replace",
    "path": "/spec/template/spec/containers/0/args",
    "value": [
      "--secure-port=4443",
      "--cert-dir=/tmp",
      "--kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname",
      "--kubelet-insecure-tls=true",
      "--kubelet-use-node-status-port",
      "--metric-resolution=15s"
    ]
  },
  {
    "op": "replace",
    "path": "/spec/template/spec/containers/0/readinessProbe/httpGet/port",
    "value": 4443
  },
  {
    "op": "replace",
    "path": "/spec/template/spec/containers/0/livenessProbe/httpGet/port",
    "value": 4443
  }
]'
```

You also have to update service of `metric-server`
```bash
kubectl patch service metric-server-metrics-server -n kube-system --type='json' -p='[
  {
    "op": "replace",
    "path": "/spec/ports/0/targetPort",
    "value": 4443
  }
]'
```
