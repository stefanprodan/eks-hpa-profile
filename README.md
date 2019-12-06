# eks-hpa-profile

An [eksctl](https://eksctl.io) GitOps profile for horizontal pod autoscaling with Prometheus metrics.

### Prerequisites

Install eksctl and fluxctl for macOS or Linux with [Homebrew](https://brew.sh/):

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

We'll use the managed nodes for cluster add-ons(CoreDNS, KubeProxy) and for the horizontal pod autoscaler metrics
add-ons(Metrics server, Prometheus, Prometheus metrics adapter).
We'll use Fargate for the demo application [podinfo](https://github.com/stefanprodan/eks-hpa-profile/tree/master/demo/podinfo),
note that only the pods deployed in the `demo` namespace with a `scheduler: fargate` label will be running as Fargate tasks.

### Create a GitHub repository

Create a GitHub [repository](https://github.com/new) and clone it locally.
Replace `GH_USER/GH_REPO` value with your GitHub username and new repo.
We'll use these variables to clone your repo and setup GitOps for your cluster.

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
Once that is done, Flux will be able to pick up changes in the repository and deploy them to the cluster.

### Install the HPA profile

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

