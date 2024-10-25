# Flux_Demo


## Create a kubernetes cluster

In this guide we will need a Kubernetes cluster for testing. Let's create one using [kind](https://kind.sigs.k8s.io/) </br>

```
kind create cluster --name fluxcd --image kindest/node:v1.31.1
```
### Install kubectl

```
curl -sLO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
mv ./kubectl /usr/local/bin/kubectl

```

### Test cluster access:
```
kubectl get nodes

NAME                    STATUS   ROLES    AGE   VERSION
fluxcd-control-plane   Ready    control-plane   48s   v1.31.1
```
## Install the Flux CLI

Let's download the `flux` command-line utility. </br>
We can get this utility from the GitHub [Releases page](https://github.com/fluxcd/flux2/releases) </br>

You will want to make sure you are getting a compatible version of flux that supports your version of Kubernetes. Checkout the [prerequisites](https://fluxcd.io/flux/installation/#prerequisites) page. </br>

```
curl -o /tmp/flux.tar.gz -sLO https://github.com/fluxcd/flux2/releases/download/v2.4.0/flux_2.4.0_linux_amd64.tar.gz
tar -C /tmp/ -zxvf /tmp/flux.tar.gz
mv /tmp/flux /usr/local/bin/flux
chmod +x /usr/local/bin/flux
```

Now we can run `flux --help` to see its installed

## Check our test cluster

```
flux check --pre

► checking prerequisites
✔ Kubernetes 1.31.1 >=1.28.0-0
✔ prerequisites checks passed
```

## Flux documentation

The [Core Concepts](https://fluxcd.io/flux/concepts/) is a good place to start. </br>

We start from the steps under the [bootstrap](https://fluxcd.io/flux/installation/#bootstrap) section for GitHub </br>

We will need to generate a [personal access token (PAT)](https://github.com/settings/tokens/new) that can create repositories by checking all permissions under `repo`.  </br>

Once we have a token, we can set it:

```
export GITHUB_TOKEN=<your-token>
```

Then we can bootstrap it using the GitHub bootstrap method

```
flux bootstrap github \
  --token-auth \
  --owner=Eugentk \
  --repository=flux-demo \
  --path=clusters/demo-cluster \
  --personal \
  --branch demo

flux check

kubectl -n flux-system get GitRepository
kubectl -n flux-system get Kustomization
```

# GitOps Repository structures

In GitOps we have a dedicated repo for infrastructure templates. </br>
Your infrastructure will "sync" from the this repo </br>

```
                                                    
 developer    +-----------+     +-----------------+           
              |           |     |                 |           
  ----------> | REPO(code)|---> |    CI PIPELINE  |           
              +-----------+     +-----------------+           
                                         |  commit     
                                         v             
           +----------+  sync   +-----------------+           
           |  INFRA   |-------> |                 |           
           |  (k8s)   |         |INFRA REPO(yaml) |           
           +----------+         +-----------------+           
                                                            
```

Flux repository structure [documentation](https://fluxcd.io/flux/guides/repository-structure/)

* Mono Repo (all k8s YAML in same "infra repo")
* Repo per team
* Repo per app

## Build our demo apps

We have a microservice called `demo-app-1` and it has its own GitHub repo somewhere. </br>
For demo, it's code is under `apps/demo-app-1/`

```
# go to our demo-app-1 

cd apps/demo-app-1

docker build -t demo-app-1:1.0.1 .

#load the image to our demo cluster so we dont need to push to a registry

kind load docker-image demo-app-1:1.0.1 --name fluxcd 
```

## Setup gitops pipeline

Now we will also have an infrastructure configuration files for GitOps.

```
cd 

# tell flux where our Git repo is and where the YAML is
# flux will monitor the demo-app-1 Git repo for when any infrastructure changes, it will sync

kubectl -n app-1 apply -f apps/demo-app-1/gitrepository.yaml
kubectl -n app-1 apply -f apps/demo-app-1/kustomization.yaml

# check our flux resources
 
kubectl -n app-1 describe gitrepository demo-app-1
kubectl -n app-1 describe kustomization demo-app-1

# check deployed resources

kubectl get all -n app-1

kubectl port-forward svc/demo-app-1 8080:8080

```
## Change demo-app-1 version

Once we make changes to our apps/demo-app-1/index.html we can build a new image with a new tag

```
docker build . -t demo-app-1:1.0.2

kind load docker-image demo-app-1:1.0.2 --name fluxcd 

```
Update our kubernetes deployment flux_demo/infrastructure/apps/demo-app-1/deploy/deployment.yaml image tag and push it to our git registry

If we wait a minute or so we can ` kubectl port-forward svc/demo-app-1 8080:8080` again and see the changes
