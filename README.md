MB K8s Challenge
================

Setup
-----

Tested with
```
colima version 0.4.6
git commit: 10377f3a20c2b0f7196ad5944264b69f048a3d40

runtime: containerd
arch: aarch64
client: v0.23.0
server: v1.6.8

kubernetes
Client Version: v1.25.3
Kustomize Version: v4.5.7
Server Version: v1.25.0+k3s1
```
On MacBook Pro with M1

### Install prerequisites & start VM and K8s

```sh
brew install colima kubectl helm helmfile
colima start --runtime containerd --memory 4 --kubernetes
```

Prometheus & Grafana
--------------------

- No alertmanager (not needed according to challenge)
- node exporter with adjustments to work in virtualized environment

### Installation

```sh
cd infra/monitoring

# add custom resource definitions
kubectl create -f crds/

# apply helm chart
helmfile apply
```

### Access Grafana

```sh
kubectl port-forward svc/prom-grafana 8080:80
```

Open browser under `http://localhost:8080` and enter credentials `admin:prom-operator`.