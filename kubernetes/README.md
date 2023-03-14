# Kubernetes Workshop

This is a kubernetes lab for local development

## TOC

* [Setup Pre-Requisites](#setup-prerequisites)
  * [Install Docker](#install-docker)
  * [Install Kubectl](#install-kubectl)
  * [Install Kubectx](#install-kubectx)
  * [Install Helm](#install-helm)
  * [Install Stern](#install-stern)
  * [Install k9s](#install-k9s)
* [Local Kubernetes Cluster](#local-kubernetes-cluster)
  * [Setup a Cluster with KinD](#setup-a-cluster-with-kind)
  * [Communicate to Kubernetes with kubectl](#communicate-to-kubernetes-with-kubectl)
  * [Use kubectx to switch contexts](#use-kubectx-to-switch-contexts)
  * [Use kubens to switch default namespace](#use-kubens-to-switch-default-namespace)
* [Helm 101](#helm-101)
* [Ingress Controller](#setup-ingress-controller)
  * [Setup a Nginx Ingress Controller with Helm](#setup-a-nginx-ingress-controller-with-helm)
  * [View Nginx Ingress Controller Resources](#view-nginx-ingress-controller-resources)
  * [Deploy a Sample Application with Ingress](#deploy-a-sample-application-with-ingress)
* [Deploy a Web Application](#deploy-a-web-application)
* [Create a Helm Chart](#create-a-helm-chart)
## Setup PreRequisites

To follow this kubernetes workshop we will be installing a couple of required applications so that you can follow step-by-step. This is demonstrated for MacOSx, but I will leave resources for Linux and Windows as well.

The tools we will require:
- **Docker**: a container runtime environment, which KinD will use to run "nodes" as containers.
- **Kubectl**: a kubernetes client that interacts with the kubernetes cluster.
- **Helm**: a package manager for kubernetes, enables you to manage kubernetes applications easier.
- **Stern**: a log cli client that allows you to tail kubernetes logs from multiple pods at once
- **k9s**: a kubernetes terminal ui

Resources:
- https://agrimprasad.com/post/supercharge-kubernetes-setup/
- https://lobogit.unm.edu/blue/linux-cfg/-/tree/59176530912c8cf195070bb6149ea6498b5b5008/.oh-my-zsh/plugins/kube-ps1

### Install Docker

We will be using KinD to run a local kubernetes cluster, and KinD (Kubernetes in Docker) requires [Docker](https://docs.docker.com/desktop/install/mac-install/) to be installed.

See the following links to install Docker for your operating system:

- [Mac](https://docs.docker.com/desktop/install/mac-install/)
- [Linux](https://docs.docker.com/desktop/install/linux-install/)
- [Windows](https://docs.docker.com/desktop/install/windows-install/)

Once you have docker installed you should be able to run:

```bash
docker version
```

### Install Kubectl

We will need `kubectl` installed in order to interact with the kubernetes cluster.

If you are on [Mac](https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/) and you have `homebrew` installed you can install `kubectl` with `homebrew`:

```bash
brew install kubectl
```

If you are on [Linux](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/), you can install `kubectl` with `curl`:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

If you are on [Windows](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/) and you have `choco` installed:

```bash
choco install kubernetes-cli
```

Once you have `kubectl` installed, you should be able to run:

```bash
kubectl version --client
```
### Install Helm

Install [helm](https://helm.sh/) using the operating system of your choice:

For [Mac](https://helm.sh/docs/intro/install/#from-homebrew-macos), you can use `brew`:

```bash
brew install helm
```

For [Windows](https://helm.sh/docs/intro/install/#from-chocolatey-windows), you can use `choco`:

```bash
choco install kubernetes-helm
```

For [Linux](https://helm.sh/docs/intro/install/), you can use `wget`:

```bash
wget https://get.helm.sh/helm-canary-linux-amd64.tar.gz
tar -xf helm-canary-linux-amd64.tar.gz
sudo install -o root -g root -m 0755 ./linux-amd64/helm /usr/local/bin/helm
```

Once helm is installed you can test it with:

```bash
helm version
```

### Install Stern

- https://github.com/wercker/stern

### Install k9s

- https://github.com/derailed/k9s

## Local Kubernetes Cluster

### Setup a Local Kubernetes Cluster with KinD

### Communicate to Kubernetes with kubectl

### Use kubectx to switch contexts

### Use kubens to switch default namespace

## Helm 101

## Ingress Controller

### Setup a Nginx Ingress Controller with Helm

View nodes:

```bash
kubectl get nodes
NAME                   STATUS   ROLES           AGE   VERSION
sektor-control-plane   Ready    control-plane   44h   v1.24.0
sektor-worker          Ready    <none>          44h   v1.24.0
```

Add helm repo:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

Install a release:

```bash
helm upgrade --install ingress-nginx ingress-nginx --repo https://kubernetes.github.io/ingress-nginx --namespace ingress-nginx --create-namespace
```

View pods:

```bash
kubectl get pods --namespace=ingress-nginx --selector=app.kubernetes.io/component=controller
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-6bf7bc7f94-g7qps   1/1     Running   0          84s
```

Create deployment:

```bash
kubectl create deployment demo --image=httpd --port=80
deployment.apps/demo created
```

View deployment:

```yaml
# k get deployment/demo -o yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  labels:
    app: demo
  name: demo
  namespace: default
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: demo
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: demo
    spec:
      containers:
      - image: httpd
        imagePullPolicy: Always
        name: httpd
        ports:
        - containerPort: 80
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
```

Create a service:

```bash
kubectl expose deployment demo
service/demo exposed
```

View service:

```yaml
# k get svc/demo -o yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: "2022-08-25T06:29:03Z"
  labels:
    app: demo
  name: demo
  namespace: default
  resourceVersion: "107443"
  uid: 39c59b5c-bc2a-4ec8-9819-7925965d1928
spec:
  clusterIP: 10.96.228.45
  clusterIPs:
  - 10.96.228.45
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: demo
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
```

Create ingress:

```bash
kubectl create ingress demo-localhost --class=nginx --rule="demo.localdev.me/*=demo:80"
```

View ingress:

```yaml
# k get ingress/demo-localhost -o yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  creationTimestamp: "2022-08-25T06:30:06Z"
  generation: 1
  name: demo-localhost
  namespace: default
  resourceVersion: "107544"
  uid: 800bf38f-5631-4f54-b59d-07faf6d8efc0
spec:
  ingressClassName: nginx
  rules:
  - host: demo.localdev.me
    http:
      paths:
      - backend:
          service:
            name: demo
            port:
              number: 80
        path: /
        pathType: Prefix
status:
  loadBalancer: {}
```

Expose a port:

```bash
kubectl port-forward --namespace=ingress-nginx service/ingress-nginx-controller 8080:80
```

Test

```bash
curl -I http://demo.localdev.me:8080/
HTTP/1.1 200 OK
```

- https://kubernetes.github.io/ingress-nginx/deploy/#quick-start

### View Nginx Ingress Controller Resources

### Deploy a Sample Application with Ingress

## Deploy a Web Application

## Create a Helm Chart