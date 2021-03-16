# Fury Distribution Minimal

A minimal deployment of the [Fury Kubernetes Distribution](https://github.com/sighupio/fury-distribution) using the GitOps paradigm.

For a regular deployment in a minikube environment without GitOps go [here](https://github.com/nikever/fury-distribution-minimal).

## Tools

- flux v2 `>= 0.9` ([installation guide](https://toolkit.fluxcd.io/guides/installation/))
- furyctl `>= 0.4` ([installation guide](https://github.com/sighupio/furyctl#install))
- minikube `>= v1.18` ([installation guide](https://minikube.sigs.k8s.io/docs/start/))

There are various others GitOps tools available (e.g. Argo CD), for this demo I choose Flux v2.

## 0. GitHub Token

Create a GitHub personal access token following this [guide](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token).

Remember to grant all `repo` permissions.

Once created save the token in an environment variable:

```bash
export GITHUB_TOKEN=<TOKEN_ID>
export GITHUB_USER=<MY_GITHUB_USER> (e.g. nikever)
```

## 1. Start minikube cluster

At the root level of this repository, execute:

```bash
export REPO_DIR=$(PWD)
```

Start a minikube cluster:

```bash
cd $REPO_DIR/minikube
make setup
```

By default, it will spin up a one node Kubernetes cluster of version `v1.19.4` in a VirtualBox VM (CPUs=4, Memory=8192MB, Disk=20000MB).

You can pass additional parameters to change the default values:

```bash
make setup cpu=2 memory=2048
```

If you reduce the `memory` and `cpu` of the node, remember to adjust resources requests and limits accordingly.

Please referer to this [Makefile](minikube/Makefile) for additional details on the cluster creation.

## 1. Install Flux CD

### Bootstrap Flux v2

We are going to bootstrap flux via the cli:

```bash
flux bootstrap github \
    --owner $GITHUB_USER \
    --repository fury-flux-fleet \
    --branch main \
    --path ./clusters/fury-minimal-cluster \
    --personal
```

The bootstrap command creates a repository if one doesn't exist, commits the manifests for the Flux components to the default branch at the specified path, and installs the Flux components. Then it configures the target cluster to synchronize with the specified path inside the repository.

Wait for the pods to be up and running:

```bash
kubectl -n flux-system get pods -w
```

### Clone the created repository

```bash
mkdir $REPO_DIR/demo
cd $REPO_DIR/demo

git clone https://github.com/$GITHUB_USER/fury-flux-fleet
cd fury-flux-fleet
```

## 2. Deploy Fury Distribution

Create a folder for the Fury distribution:

```bash
mkdir -p $REPO_DIR/clusters/fury-minimal-cluster/fury/
```

### Create Flux GitRepository

Create a `GitRepository` manifest pointing to [*Fury Distribution Minimal*](https://github.com/nikever/fury-distribution-minimal) repository's master branch:

```bash
flux create source git fury-distribution \
    --url https://github.com/nikever/fury-distribution-minimal \
    --branch main \
    --interval 1m \
    --export \
    > ./clusters/fury-minimal-cluster/fury/fury-distribution-source.yaml
```

### Create a Flux Kustomization

Create a Flux `Kustomization` manifest for the *Fury distribution*. This configures Flux to build and apply the kustomize directory located in the *Fury Distribution Minimal* repository under the `fury` folder.

```bash
flux create kustomization fury-distribution \
  --source=fury-distribution \
  --path="./fury" \
  --prune=true \
  --interval=5m \
  --export \
  > ./clusters/fury-minimal-cluster/fury/fury-distribution-kustomization.yaml
```

We don't perform any `validation` due to some race-condition on Custom Resource Definitions (CRDs). This happens because CRDs are referenced before they are created.

### Deploy via GitOps

Commit and push to deploy the *Fury Distribution* in the cluster:

```bash
git add -A && git commit -m "Add Fury Minimal Distribution GitRepository and Kustomization"
git push
```

Wait for Flux to reconcile everything:

```bash
watch flux get sources git
watch flux get kustomizations
```

Wait for pods to be up and running:

```bash
kubectl get pods -A -w
```

### Explore the Fury distribution

Use `minikube ip` to get the external IP of the cluster:

```bash
$ minikube ip     
192.168.99.113
```

Add ingresses to `/etc/hosts` file by adding the following line to the bottom of the file:

```bash
192.168.99.113 directory.fury.info alertmanager.fury.info goldpinger.fury.info grafana.fury.info prometheus.fury.info >> /etc/hosts
```

Replace `192.168.99.113` with your actual IP.
If you are adventurous enough, you might want to use this one-liner:

```bash
sudo bash -c "echo $(minikube ip) directory.fury.info alertmanager.fury.info goldpinger.fury.info grafana.fury.info prometheus.fury.info >> /etc/hosts"
```

Now, open a browser and go to [directory.fury.info](http://directory.fury.info), you should be able to see all the cluster ingresses.

More information of what can you do can be found [here](https://github.com/nikever/fury-distribution-minimal#explore-the-distribution).

## 3. Deploy demo application

To conclude we are going to deploy a simple demo app in our cluster. We are going to deploy a [sample web application](https://github.com/GoogleCloudPlatform/kubernetes-engine-samples/tree/master/hello-app) called `hello-app`. It is a web server written in Go that responds to all requests with the message `Hello, World!` on port `8080`.

The application is available as two Docker images, which respond to requests with different version numbers:

- `gcr.io/google-samples/hello-app:1.0`
- `gcr.io/google-samples/hello-app:2.0`

We are going to deploy `gcr.io/google-samples/hello-app:1.0` first, and then update to `gcr.io/google-samples/hello-app:2.0`.

### Create Flux GitRepository and Kustomization

Let's create a new directory first for our app:

```bash
mkdir -p $REPO_DIR/clusters/fury-minimal-cluster/hello-app/
```

Now, we can create a `GitRepository` manifest pointing to the [*Hello App*](https://github.com/nikever/kubernetes-hello-app). We will point to the branch `hello-app-1.0` that uses the `gcr.io/google-samples/hello-app:1.0` image.

```bash
cd $REPO_DIR

flux create source git hello-app \
    --url https://github.com/nikever/kubernetes-hello-app \
    --branch hello-app-1.0 \
    --interval 1m \
    --export \
    > ./clusters/fury-minimal-cluster/hello-app/hello-app-source.yaml
```

```bash
flux create kustomization hello-app \
  --source=hello-app \
  --path="." \
  --prune=true \
  --interval=5m \
  --export \
  > ./clusters/fury-minimal-cluster/hello-app/hello-app-kustomization.yaml
```

Commit and push to deploy the *Hello App*:

```bash
git add -A && git commit -m "Add hello-app GitRepository and Kustomization"
git push
```

Wait for Flux to reconcile everything:

```bash
watch flux get sources git
watch flux get kustomizations
```

Wait for pods to be up and running:

```bash
kubectl get pods -n logging -w
```

### Test the app

Add the following line to the bottom of the `/etc/hosts` file manually:

```bash
<MINIKUBE_IP> hello-world.info
```

(where `<MINIKUBE_IP>` must be replace with output of `minikube ip`)

or, again, via:

```bash
sudo bash -c "echo $(minikube ip) hello-world.info >> /etc/hosts"
```

Verify that the Ingress controller is directing traffic:

```bash
curl hello-world.info
```

It should output something similar to:

```bash
Hello, world!
Version: 1.0.0
Hostname: hello-app-5c4957dcc4-l4mqz
```

Ensure that the version is `1.0.0`.

### Update the app

To update the app, we simply going to change the branch of the `hello-app` GitRepository from `branch: hello-app-1.0` to `branch: hello-app-2.0`.

```bash
cat >> ./clusters/fury-minimal-cluster/hello-app/hello-app-source.yaml << EOF

---
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: GitRepository
metadata:
  name: hello-app
  namespace: flux-system
spec:
  interval: 1m0s
  ref:
    branch: hello-app-2.0
  url: https://github.com/nikever/kubernetes-hello-app
EOF
```

Commit to apply:

```bash
git add -A && git commit -m "Add hello-app GitRepository and Kustomization"
git push
```

Wait for Flux to reconcile everything:

```bash
watch flux get sources git
watch flux get kustomizations
```

Now you can retry the app:

```bash
curl hello-world.info
```

It should output `Version: 2.0.0`:

```bash
Hello, world!
Version: 2.0.0
Hostname: hello-app-5c4957dcc4-l4mqz
```

## 4. Clean up

To clean up:

- delete the `minikube` cluster:

```bash
cd $REPO_DIR/minikube
make delete
```

- delete the lines you added in the `/etc/hosts`

- delete the `fury-flux-fleet` repository (optional).

## Additions

### Handle Private Repository

As a side note, in case of private repositories you can:

- Generate SSH Credentials
- Add Deploy key in GitHub
- Create a secret containing SSH credentials
- Specify the secret name in the `secret-ref` parameter when creating GitRepository

This approach is quick but not really "GitOps" since it requires `kubectl apply`.
It is okay for a quick demo, consider alternatives in a real setup.

#### Generate ssh credentials

```bash
mkdir secrets

ssh-keygen -q -N "" -f ./secrets/identity
ssh-keyscan github.com > ./secrets/known_hosts
```

#### Add deploy key in GitHub

Copy the public key:

```bash
cat identity.pub
ssh-rsa AAAAB3NzaC1yc2EAA...
```

Add it to deploy keys under `https://github.com/<GITHUB_USER>/<REPOSITORY>/settings/keys`

#### Create secret

```bash
kubectl create secret generic ssh-credentials \
    --from-file=./secrets/identity \
    --from-file=./secrets/identity.pub \
    --from-file=./secrets/known_hosts \
    --namespace=flux-system
```

#### Create GitRepository

```bash
flux create source git fury-distribution \
    --url PRIVATE_REPO/fury-distribution-minimal \
    --branch master \
    --interval 1m \
    --secret-ref ssh-credentials \
    --export \
    | tee ./clusters/fury-minimal-cluster/fury/fury-distribution-source.yaml
```

## Credits

This guide uses:

- [Get started with Flux v2](https://toolkit.fluxcd.io/get-started/)
- [Fury Distribution Minimal](https://github.com/nikever/fury-distribution-minimal)
