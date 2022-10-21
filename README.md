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
brew install colima kubectl helm helmfile kustomize

# needed for helmfile apply and helmfile diff
helm plugin install https://github.com/databus23/helm-diff

# start virtual machine with k8s
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
# enable port forwarding between localhost port 8080 and grafana service on port 80
kubectl -n monitoring port-forward svc/prom-grafana 8080:80
```

Open browser under `http://localhost:8080` and enter credentials `admin:prom-operator`.

kuard
-----

- Deployment is implemented with `kustomize`
  - This documentation assumes that the standalone tool is installed instead of kubectl subcommand
- It consists of:
  - `Deployment`
  - `Service`
  - `HorizontalPodAutoscaler`
  - `ServiceMonitor`
- There are no overlays because there is only one environment

### Deployment

```sh
cd apps/kuard

# create namespace
kubectl create ns kuard

# deploy kustomize output
kustomize build | kubectl -n kuard apply -f -
```

Tasks
=====

## Logging solution within a Kubernetes environment?

## How to achieve application scalability caused by a lot of requests?

The simplest method is to increase the `Pod`'s resource limits or to remove them completely. This vertical scaling approach is highly discouraged as it's ignoring the benefits of using a distributed orchestrator like Kubernetes is. 

Instead of deploying a single `Pod` for an app to serve traffic it's better to embed the `Pod` within a `ReplicaSet`. Typically you use a `Deployment` instead of a `ReplicaSet` directly which itself creates and manages `ReplicaSets` underneath.

The benefit of a `ReplicaSet` over a single `Pod` is that it's able to create multiple instances of the same `Pod`. A `Service` object then abstracts the load distribution via a set of `EndPoints` which are connected to a `Pod`. Depending on the amount of requests or load you could manually increase the number of replicas and the `Service` is responsible to distribute the traffic automatically.

Increasing (or decreasing) the amount of replicas can be done by executing the following command.

```sh
# set the number of replicas of the kuard deployment
kubectl scale -n kuard deployment kuard --replicas 2
```

It's also possible to do that with the underlying `ReplicaSet` instead of the `Deployment`.

Since manual scaling is error prone and can take a lot of time it might be better to look for an automated way. For this reason there is the so called `HorizontalPodAutoscaler` (HPA) object within Kubernetes. In short HPAs can be used to track specific metrics of e.g. a `Deployment` and change the numbers of replicase accordingly. The metrics are scraped via the metrics-server which usually runs as part of every Kubernetes installation.

The demo application includes an HPA (`apps/kuard/hpa.yaml`) that will scale the application in case the average CPU utilization is higher than 70% of the CPU requests. The scale up can be simulated by running a load test tool like `ApacheBench` with the following command.

```sh
# enable port forwarding between localhost port 8080 and kuard service on port 80
kubectl port-forward -n kuard svc/kuard 8080:80

# start load test with 10000 requests in total and a concurrency of 100
ab -n 10000 -c 100 http://localhost:8080/
```

This should increase the number of replicas automatically after some time.

## How to detect errors in templating before an actual deployment?

In order to reduce the errors that can happen either within manifests, templating or values it's advisable to validate or lint the output before applying it. There are 3rd party tools that can check the manifests and integrated options to achieve it. All the options below can either be used in combination with helm or kustomize or even other templating tools. It's important to send the final output to the tool of choice. Since `kuard` uses `Kustomize` only those examples are shown below. To generate the helm output the `helm template ...` command can be used. 

> In general it's a good idea to put those checks into continuous delivery / deployment pipelines before they're finally pushed to Kubernetes.

One of the issues a user can run into with 3rd party applications are custom resources. 3rd party applications like [datree](https://github.com/datreeio/datree), [kubeval](https://kubeval.instrumenta.dev/) and [kubeconform](https://github.com/yannh/kubeconform) usually need to be instructed how to validate custom resources with the help of schema files in JSON format.

If there are no custom resource in use (only default ojects like `Deployment`, `Service`, `ConfigMap`, `Secret` etc.) then the tools mentioned above are a good way to check if the output is valid.

An already integrated approach is to use `kubectl` in combination with the `--dry-run` flag. Here it's possible to let either the client or the server validate the rendered template or manifests.

```sh
# change into kuard directory
cd apps/kuard

# client side validation
kustomize build | kubectl apply -n kuard --dry-run=client -f -

# server side validation
kustomize build | kubectl apply -n kuard --dry-run=server -f -
```

Server side validation is always preferable since it already knows all the `CustomResourceDefinitions`. But client side validation might be needed if the executing system is not able or allowed to run the validation against the Kubernetes API.

## Which options do you have to rollback an errorneous deployment to an earlier version?

If a deployment of an application is not running as expected then there are multiple options how to rollback. It depends on the tooling that is used to do the deployment at all. The following list has three potential approaches but it does not cover all possible solutions.

### Manual rollback

Whenever the `.spec` part within a `Deployment` changes then there will be a new history entry. THe history of a `Deployment` can be checked with the following command.

```sh
# print the history for the kuard deployment
kubectl rollout -n kuard history deployment kuard
```

Unfortunately the `CHANGE-CAUSE`column is not used by default and it might not be in the [future](https://github.com/kubernetes/kubernetes/issues/40422). Anyway it's possible to rollback to the latest or any other previous revision with the `rollout undo` command.

```sh
kubectl rollout undo deployment kuard [--to-revision=8]
```

If the `--to-revision` parameter – which is optional – is omitted then automatically the last revision is used.

### Helm

Every `helm upgrade` command creates a new history entry which is actually just another secret that contains the deployment information. If Helm is used then it's easy to rollback with a single command.

```sh
# show history of a helm release
helm history -n monitoring prom

# rollback to specific revision 1
helm rollback -n monitoring prom 1
```

Here the revision parameter is optional as well. If it's omitted then it will use the previous revision automatically.

### GitOps

Both methodes above have the downside that it's hard to reason about a change and the rollback. With the [GitOps](https://www.gitops.tech/) approach things become more verbose. No matter if a GitOps tool like [ArgoCD](https://argoproj.github.io/cd/), [Flux](https://fluxcd.io/) or [Jenkins X](https://jenkins-x.io/) or a custom implementation is used every change is done _only_ in Git with one or multiple commits.

Let's say there's a new deployment triggered by a commit that changes a container's image to a new tag. First of all we can see the change in the Git history with (hopefully) a meaningful message. When the new container is faulty or doesn't even start and a rollback is needed then the last commit can simply be reverted and pushed. That will also rollback the `Deployment` to the previous, working version.

```sh
# git the commit hash that is faulty (-1 is just the latest)
git log -1

# revert the hash return by previous command and add a meaningful message
git revert <hash>

# push to remote
git push
```

With this approach each and every change is part of the repository and thus easier to comprehend.

## Which metrics does the demo app (kuard) provide?

They can be found under the `/metrics` route. On a local / dev cluster you can access them as follows:

```sh
# enable port forwarding between localhost port 8080 and kuard service on port 80
kubectl port-forward -n kuard svc/kuard 8080:80
curl http://localhost:8080/metrics
```
curl output:

```
# HELP go_gc_duration_seconds A summary of the GC invocation durations.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 0.000195
go_gc_duration_seconds{quantile="0.25"} 0.000463
go_gc_duration_seconds{quantile="0.5"} 0.000641
go_gc_duration_seconds{quantile="0.75"} 0.000865
go_gc_duration_seconds{quantile="1"} 0.046545
go_gc_duration_seconds_sum 0.509364
go_gc_duration_seconds_count 335
# HELP go_goroutines Number of goroutines that currently exist.
# TYPE go_goroutines gauge
go_goroutines 6
[...]
```

There is also a custom resource `ServiceMonitor` within the `kuard` namespace that is collecting the metrics. They can be checked in prometheus with the query `{service="kuard"}`. To expose prometheus you can run the following.

```sh
# enable port forwarding between localhost port 9090 and prometheus service on port 9090
kubectl port-forward -n monitoring svc/prom-kube-prometheus-stack-prometheus 9090:9090
```

Afterwards you can access prometheus in the browser via http://localhost:9090.

## Which one would be the CI/CD tool of your choice?

If the question is related to the demo application then CI can be ignored as the final image was present already.

If it's a general question then I'd look for some widely used solution that has proven its value already, especially in the Cloud Native world. My choice would be either GitHub Actions or something that is running directly inside the cluster (e.g. [Tekton](https://tekton.dev/) with [Kaniko](https://github.com/GoogleContainerTools/kaniko)).
Another tool that looks promising is [Dagger](https://dagger.io/) which can close the gap between a developer machine and the CI system that will really build the final image and run all the other pipeline steps.

Independent of the project I'd choose [Flux](https://fluxcd.io/) for the CD part. It allows the user to work with the GitOps approach, has an alerting system, works with various Git systems out of the box and supports various ways of keeping (unencrpyted) secrets out of the repository.

Additional Questions
====================

## How do I expose an application via an ingress controller and what are ingress controllers in general used for?


## How could alerting in a K8s cluster look like? Which options are available?

## Which options do you have to get data into Promehtheus' TSDB?

## Which options do you have to secure sensitive data (e.g. secrets) in a GIT repository? Which one would you choose?

## Which options do you have to completely remove data that has been leaked into a GIT repository by mistake?