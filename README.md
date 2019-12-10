# eks-hpa-profile

An [eksctl](https://eksctl.io) GitOps profile for horizontal pod autoscaling with Prometheus metrics.

### Prerequisites

Install eksctl and fluxctl for macOS with [Homebrew](https://brew.sh/):

```sh
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
brew install fluxctl
```

For Windows you can use [Chocolatey](http://chocolatey.org):

```sh
choco install eksctl
choco install fluxctl
```

### Create an EKS cluster

Create an EKS cluster with two EC2 managed nodes and a Fargate profile:

```sh
cat << EOF | eksctl create cluster -f -
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: eks-fargate-hpa
  region: eu-west-1

managedNodeGroups:
  - name: default
    instanceType: m5.large
    desiredCapacity: 2
    volumeSize: 120
    iam:
      withAddonPolicies:
        appMesh: true
        albIngress: true

fargateProfiles:
  - name: default
    selectors:
      - namespace: demo
        labels:
          scheduler: fargate
EOF
```

You'll use the managed nodes for cluster add-ons(CoreDNS, KubeProxy) and for the horizontal pod autoscaler metrics
add-ons(Metrics server, Prometheus, Prometheus metrics adapter).

You'll use Fargate for the demo application [podinfo](https://github.com/stefanprodan/eks-hpa-profile/tree/master/demo/podinfo),
note that only the pods deployed in the `demo` namespace with a `scheduler: fargate` label will be running as Fargate tasks.

### Create a GitHub repository

Create a GitHub [repository](https://github.com/new) and clone it locally.
Replace `GH_USER/GH_REPO` value with your GitHub username and new repo.
Use these variables to clone your repo and setup GitOps for your cluster.

```sh
export GH_USER=YOUR_GITHUB_USERNAME
export GH_REPO=YOUR_GITHUB_REPOSITORY

git clone https://github.com/${GH_USER}/${GH_REPO}
cd ${GH_REPO}
```

Run the eksctl repo command:

```sh
export EKSCTL_EXPERIMENTAL=true

eksctl enable repo \
--cluster=eks-fargate-hpa \
--region=eu-west-1 \
--git-url=git@github.com:${GH_USER}/${GH_REPO} \
--git-user=fluxcd \
--git-email=${GH_USER}@users.noreply.github.com
```

The command `eksctl enable repo` takes an existing EKS cluster and an empty repository 
and sets up a GitOps pipeline.

After the command finishes installing [FluxCD](https://github.com/fluxcd/flux) and [Helm Operator](https://github.com/fluxcd/flux),
you will be asked to add Flux's deploy key to your GitHub repository.

Copy the public key and create a deploy key with write access on your GitHub repository.
Go to `Settings > Deploy keys` click on `Add deploy key`, check `Allow write access`,
paste the Flux public key and click `Add key`.

Once that is done, Flux will be able to pick up changes in the repository and deploy them to the cluster.

### Install the metrics add-ons

Run the eksctl profile command:

```sh
eksctl enable profile \
--name=https://github.com/stefanprodan/eks-hpa-profile \
--cluster=eks-fargate-hpa \
--region=eu-west-1 \
--git-url=git@github.com:${GH_USER}/${GH_REPO} \
--git-user=fluxcd \
--git-email=${GH_USER}@users.noreply.github.com
```

The command `eksctl enable profile` adds the HPA metrics add-ons and
the demo app manifests to the configured repository.

Sync your local repository with:

```sh
git pull origin master
```

Run the fluxctl sync command to apply the manifests on your cluster:

```sh
fluxctl sync --k8s-fwd-ns flux
```

Flux does a git-cluster reconciliation every five minutes,
the above command can be used to speed up the synchronization.

List the installed components:

```
$ kubectl -n monitoring-system get helmreleases

NAME                 RELEASE              STATUS
metrics-server       metrics-server       DEPLOYED
prometheus           prometheus           DEPLOYED
prometheus-adapter   prometheus-adapter   DEPLOYED
```

### Install podinfo

You'll use a Go web app named [podinfo](https://github.com/stefanprodan/podinfo) to test
the Horizontal Pod Autoscaler (HPA).
The app is instrumented with Prometheus and exposes the `http_requests_total` [counter](https://prometheus.io/docs/concepts/metric_types/#counter).
The HPA controller will scale the Fargate tasks based on the number of requests per second.

Install podinfo by setting `fluxcd.io/ignore` to `false` in base/demo/namespace.yaml:

```sh
cat << EOF | tee base/demo/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo
  annotations:
    fluxcd.io/ignore: "false"
EOF
```

Apply changes via git:

```sh
git add -A && \
git commit -m "init demo" && \
git push origin master && \
fluxctl sync --k8s-fwd-ns flux
```

Wait for Fargate to schedule and start podinfo:

```sh
watch kubectl -n demo get po -l scheduler=fargate
```

When podinfo starts, Prometheus will scrape the metrics endpoint and the Prometheus adapter will export the HTTP 
requests per second metrics to the Kubernetes custom metrics API:

```
$ watch kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 | jq .

{
  "kind": "APIResourceList",
  "apiVersion": "v1",
  "groupVersion": "custom.metrics.k8s.io/v1beta1",
  "resources": [
    {
      "name": "namespaces/http_requests_per_second",
      "singularName": "",
      "namespaced": false,
      "kind": "MetricValueList",
      "verbs": [
        "get"
      ]
    },
    {
      "name": "pods/http_requests_per_second",
      "singularName": "",
      "namespaced": true,
      "kind": "MetricValueList",
      "verbs": [
        "get"
      ]
    }
  ]
}
``` 

### Configure autoscaling based on HTTP traffic

To configure auto-scaling you can set up a HPA definition that uses the `http_requests_per_second` metric. 
The HPA manifest is in `base/demo/podinfo.hpa.yaml` and it's set to scale up podinfo
when the average req/sec per pod goes over 10:

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: podinfo
  namespace: demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: podinfo
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: Pods
      pods:
        metricName: http_requests_per_second
        targetAverageValue: 10
```

Once the metric is available to the metrics API, the HPA controller will display the current value:

```
$ watch kubectl -n demo get hpa

NAME      REFERENCE            TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
podinfo   Deployment/podinfo   200m/10   1         10        1          8m58s
```

The m in `200m` represents milli-units, 200m means 0.2 req/sec.
The traffic is generated by Prometheus that scrapes the `/metrics` endpoint every five seconds.

Exec into the loadtester pod with:

```sh
kubectl -n demo exec -it loadtester-xxxx-xxxx
```

Generate traffic with `hey`:

```sh
hey -z 10m -c 5 -q 5 -disable-keepalive http://podinfo.demo
```

After a few minutes the HPA begins to scale up the deployment:

```
$ kubectl -n demo describe hpa podinfo

Events:
  Type    Reason             Age   From                       Message
  ----    ------             ----  ----                       -------
  Normal  SuccessfulRescale  2m    horizontal-pod-autoscaler  New size: 3; reason: pods metric http_requests_per_second above target
```

After the load tests finishes, the HPA down scales the deployment to it's initial replicas:

```
Events:
  Type    Reason             Age   From                       Message
  ----    ------             ----  ----                       -------
  Normal  SuccessfulRescale  21s   horizontal-pod-autoscaler  New size: 1; reason: All metrics below target
```

You may have noticed that the autoscaler doesn't react immediately to usage spikes.
By default the metrics sync happens once every 30 seconds and scaling up/down can only happen
if there was no rescaling within the last 3-5 minutes.
In this way, the HPA prevents rapid execution of conflicting decisions.
