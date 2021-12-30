# GitOps sample for running Oracle WebLogic Server on Azure Kubernetes Service

This sample is using Flux and Kustomize to create and manage Oracle WebLogic Cluster on Azure Kubernetes Service cluster automatically.

We will configure Flux:

 - To install Oracle WebLogic Kubernetes Operator using HelmRepository and HelmRelease custom resources. Flux will monitor the Helm repository, and it will automatically upgrade the Helm releases to their latest chart version based on semver ranges.

 - To install NGINX controller using HelmRepository and HelmRelease custom resources. Flux will monitor the Helm repository, and it will automatically upgrade the Helm releases to their latest chart version based on semver ranges.

 - To deploy Oracle WebLogic Server with model in image, create ingresses for the Administration Server and the cluster. Flux will monitor this repository, and it will automatically upgrade all the resource based on source code changes in clusters folder.

 ## Contents

 - [Use Flux CLI](#prerequisites)
   - [Prerequisites](#prerequisites)
   - [Bootstrap](#bootstrap)
   - [Demo Video](#demo-video)
 - [Use GitOps in Azure Acr-enabled Kubernetes](resources/docs/azure-arc-gitops.md)
 - [Encrypt Kubernetes using SOPS and Azure Key Vault](#encrypt-kubernetes-using-sops-and-azure-key-vault)


## Prerequisites

- Azure CLI. Follow this [guide](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) to install Azure CLI.

  Install Azure CLI on Linux with one command:

  ```bash
  curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
  ```
- kubectl. Follow this [guide](https://kubernetes.io/docs/tasks/tools/) to install kubectl.

  Install latest kubectl on Linux:

  ```bash
  curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

  curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"

  sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

  kubectl version --client
  ```

- You will need an Azure Kubernetes Service cluster version 1.21.7 or newer version.

  Follow this [guide](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough) to create an AKS cluster.

  Create AKS cluster using Azure CLI:

  ```bash
  AKS_NAME="acr"$(date +%s)
  REAOURCE_GROUP_NAME='wlsgitops'$(date +%s)
  az group create --name ${REAOURCE_GROUP_NAME} --location eastus

  az aks create --resource-group ${REAOURCE_GROUP_NAME} --name ${AKS_NAME} --node-count 3 --generate-ssh-keys
  ```

  Connect to AKS cluster:

  ```bash
  az aks install-cli

  az aks get-credentials --resource-group ${REAOURCE_GROUP_NAME} --name ${AKS_NAME}
  ```

- You will need an Azure Container Registry to manage your Oracle WebLogic Server image. The sample uses image `docker.io/sleepycat2/weblogic-samples:wlsonaks`. If you are bringing your own image, follow the steps to push image to ACR and enable AKS to access ACR.
  
  Follow this [guide](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-get-started-azure-cli) to create an ACR.

  Create ACR with Azure CLI:

  ```bash
  ACR_NAME="acr"$(date +%s)
  az acr create --resource-group ${REAOURCE_GROUP_NAME} \
  --name ${ACR_NAME} --sku Basic
  ```

  Follow [Create Docker image](https://oracle.github.io/weblogic-kubernetes-operator/samples/azure-kubernetes-service/model-in-image/#create-docker-image) to build image that includes your application and WebLogic configuration. Push the image to ACR. If you want to use the sample image, you are able to import the image using `az acr import`
  Import image from `docker.io/sleepycat2/weblogic-samples:wlsonaks`.

  ```bash
  az acr import --name ${ACR_NAME} --source docker.io/sleepycat2/weblogic-samples:wlsonaks --image wlsonaks:v1.0.0
  ```

  Enable AKS to access ACR, which is very important, otherwise, the AKS will fail to pull image from ACR.

  ```bash
  az aks update --name ${AKS_NAME} --resource-group ${REAOURCE_GROUP_NAME} --attach-acr ${ACR_NAME}
  ```

- In order to follow the guide you'll need a GitHub account and a personal access token that can create repositories (check all permissions under repo).

- Flux CLI.

  ```bash
  curl -s https://fluxcd.io/install.sh | sudo bash
  ```

## Repository structure

The Git repository contains the following top directories:

- **operator** dir contains Helm releases for Oracle WebLogic Kubernetes Operator.
- **ingress** dir contains Helm release for NGINX ingress controller.
- **weblogic** dir contains custom resource definition of Oracle WebLogic Server cluster and the ingress.
- **clusters**  dir contains the Flux configuration in AKS cluster
- **azure-acr-gitops** dir contains documents that guides you to set up GitOps in Azure Arc-enable AKS cluster based on this sample.

```text
├── operator
│   ├── kustomization.yaml
│   ├── namespace.yaml 
│   |── serviceaccount.yaml
|   └── release.yaml
├── infrastructure
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   └── release.yaml
├── clusters
│   ├── wko-kustomization.yaml
│   ├── ingress-kustomization.yaml 
│   └── wls-kustomization
└── azure-acr-gitops
```

In **clusters/wko-kustomization.yaml**, we have a HelmRelease for Oracle WebLogic Kubernetes Operator.

```YAML
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: weblogic-operator
  namespace: flux-system
spec:
  interval: 10m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./operator
  prune: true
  wait: true
  timeout: 30m
```

In **clusters/ingress-kustomization.yaml**, we have a HelmRelease for NGINX ingress controller.

```YAML
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: ingress-nginx
  namespace: flux-system
spec:
  interval: 10m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./ingress
  prune: true
  wait: true
  timeout: 30m
```

In **clusters/wls-kustomization.yaml**, we have a HelmRelease for NGINX ingress controller.

```YAML
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: weblgic-sample-domain1
  namespace: flux-system
spec:
  dependsOn:
    - name: weblogic-operator
    - name: ingress-nginx
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./weblogic
  prune: true
  wait: true
```

Note that the WebLogic Server resources depend on the WebLogic operator and NGINX ingress controller.

## Bootstrap 

Fork this repository on your personal GitHub account and export your GitHub access token, username and repo name:

```bash
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
export GITHUB_REPO=<repository-name>
```

Verify that your staging cluster satisfies the prerequisites with:

```bash
flux check --pre
```

Note that this sample uses image `docker.io/sleepycat2/weblogic-samples:wlsonaks`.  
If you are bringing your own image, you have to update `spec.image` in weblogic/domain.yaml with the image path in the ACR, then commit and push the change to your fork. Make sure your AKS cluster can access the ACR, see [Prerequisites](#prerequisites).

Set the kubectl context to your cluster and bootstrap Flux:

```bash
flux bootstrap github \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --path=clusters
```

The bootstrap command commits the manifests for the Flux components in clusters/flux-system dir and creates a deploy key with read-only access on GitHub, so it can pull changes inside the cluster.

Watch for the Helm releases being install:

```
$ watch flux get helmreleases --all-namespaces

NAMESPACE                       NAME                    READY   MESSAGE                                 REVISION        SUSPENDED
ingress-basic                   ingress-nginx           True    Release reconciliation succeeded        4.0.13          False
sample-weblogic-operator-ns     weblogic-operator       True    Release reconciliation succeeded        3.3.6           False

```

Watch for resources being deployed:

```
$ kubectl get pod -A -w

<... kube-system ..>
<... flux-system ..>
sample-weblogic-operator-ns   weblogic-operator-744cfc8694-262cz         0/1     Pending             0          0s
sample-weblogic-operator-ns   weblogic-operator-744cfc8694-262cz         0/1     Pending             0          0s
sample-weblogic-operator-ns   weblogic-operator-744cfc8694-262cz         0/1     ContainerCreating   0          0s
ingress-basic                 ingress-nginx-admission-create-5fgdr       0/1     Pending             0          0s
ingress-basic                 ingress-nginx-admission-create-5fgdr       0/1     Pending             0          0s
ingress-basic                 ingress-nginx-admission-create-5fgdr       0/1     ContainerCreating   0          0s
ingress-basic                 ingress-nginx-admission-create-5fgdr       0/1     Completed           0          3s
ingress-basic                 ingress-nginx-admission-create-5fgdr       0/1     Terminating         0          3s
ingress-basic                 ingress-nginx-admission-create-5fgdr       0/1     Terminating         0          3s
ingress-basic                 ingress-nginx-controller-54bfb9bb-l8k2n    0/1     Pending             0          0s
ingress-basic                 ingress-nginx-controller-54bfb9bb-l8k2n    0/1     Pending             0          0s
ingress-basic                 ingress-nginx-controller-54bfb9bb-l8k2n    0/1     ContainerCreating   0          1s
ingress-basic                 ingress-nginx-controller-54bfb9bb-l8k2n    0/1     Running             0          6s
sample-weblogic-operator-ns   weblogic-operator-744cfc8694-262cz         0/1     Running             0          12s
sample-weblogic-operator-ns   weblogic-operator-744cfc8694-262cz         1/1     Running             0          20s
ingress-basic                 ingress-nginx-controller-54bfb9bb-l8k2n    1/1     Running             0          21s
ingress-basic                 ingress-nginx-admission-patch-9vdxt        0/1     Pending             0          0s
ingress-basic                 ingress-nginx-admission-patch-9vdxt        0/1     Pending             0          0s
ingress-basic                 ingress-nginx-admission-patch-9vdxt        0/1     ContainerCreating   0          0s
ingress-basic                 ingress-nginx-admission-patch-9vdxt        0/1     Completed           0          4s
ingress-basic                 ingress-nginx-admission-patch-9vdxt        0/1     Terminating         0          4s
ingress-basic                 ingress-nginx-admission-patch-9vdxt        0/1     Terminating         0          4s
ingress-basic                 ingress-nginx-admission-create-n2sdj       0/1     Pending             0          0s
ingress-basic                 ingress-nginx-admission-create-n2sdj       0/1     Pending             0          0s
ingress-basic                 ingress-nginx-admission-create-n2sdj       0/1     ContainerCreating   0          0s
ingress-basic                 ingress-nginx-admission-create-n2sdj       0/1     Completed           0          2s
ingress-basic                 ingress-nginx-admission-create-n2sdj       0/1     Terminating         0          2s
ingress-basic                 ingress-nginx-admission-create-n2sdj       0/1     Terminating         0          2s
ingress-basic                 ingress-nginx-admission-patch-8fntp        0/1     Pending             0          0s
ingress-basic                 ingress-nginx-admission-patch-8fntp        0/1     Pending             0          0s
ingress-basic                 ingress-nginx-admission-patch-8fntp        0/1     ContainerCreating   0          0s
ingress-basic                 ingress-nginx-admission-patch-8fntp        1/1     Running             0          1s
ingress-basic                 ingress-nginx-admission-patch-8fntp        0/1     Completed           0          2s
ingress-basic                 ingress-nginx-admission-patch-8fntp        0/1     Terminating         0          2s
ingress-basic                 ingress-nginx-admission-patch-8fntp        0/1     Terminating         0          2s
sample-domain1-ns             sample-domain1-introspector-v7j7c          0/1     Pending             0          0s
sample-domain1-ns             sample-domain1-introspector-v7j7c          0/1     Pending             0          0s
sample-domain1-ns             sample-domain1-introspector-v7j7c          0/1     ContainerCreating   0          0s
sample-domain1-ns             sample-domain1-introspector-v7j7c          1/1     Running             0          29s
sample-domain1-ns             sample-domain1-introspector-v7j7c          0/1     Completed           0          101s
sample-domain1-ns             sample-domain1-introspector-v7j7c          0/1     Terminating         0          101s
sample-domain1-ns             sample-domain1-introspector-v7j7c          0/1     Terminating         0          101s
sample-domain1-ns             sample-domain1-admin-server                0/1     Pending             0          0s
sample-domain1-ns             sample-domain1-admin-server                0/1     Pending             0          0s
sample-domain1-ns             sample-domain1-admin-server                0/1     ContainerCreating   0          0s
sample-domain1-ns             sample-domain1-admin-server                0/1     Running             0          2s
sample-domain1-ns             sample-domain1-admin-server                1/1     Running             0          36s
sample-domain1-ns             sample-domain1-managed-server1             0/1     Pending             0          0s
sample-domain1-ns             sample-domain1-managed-server1             0/1     Pending             0          0s
sample-domain1-ns             sample-domain1-managed-server1             0/1     ContainerCreating   0          0s
sample-domain1-ns             sample-domain1-managed-server2             0/1     Pending             0          0s
sample-domain1-ns             sample-domain1-managed-server2             0/1     Pending             0          0s
sample-domain1-ns             sample-domain1-managed-server2             0/1     ContainerCreating   0          0s
sample-domain1-ns             sample-domain1-managed-server1             0/1     Running             0          29s
sample-domain1-ns             sample-domain1-managed-server2             0/1     Running             0          30s
sample-domain1-ns             sample-domain1-managed-server1             1/1     Running             0          63s
sample-domain1-ns             sample-domain1-managed-server2             1/1     Running             0          71s
```

Verify that the Administration Server and the demo app can be accessed via ingress.


You should find two ingresses, for the Administration Server and applications in the cluster.

```
$ kubectl get ingress -n sample-domain1-ns
NAME                                       CLASS    HOSTS   ADDRESS        PORTS   AGE
ingress-sample-domain1-admin-server        <none>   *       20.124.73.73   80      4m44s
ingress-sample-domain1-cluster-cluster-1   <none>   *       20.124.73.73   80      4m44s
```

Access the Administration Server with the ADDRESS of ingress:

```
$ curl http://<ingress-address>/console/ --verbose
*   Trying 20.124.73.73:80...
* TCP_NODELAY set
* Connected to 20.124.73.73 (20.124.73.73) port 80 (#0)
> GET /console/ HTTP/1.1
> Host: 20.124.73.73
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 302 Moved Temporarily
< Date: Wed, 29 Dec 2021 07:54:18 GMT
< Content-Type: text/html; charset=UTF-8
< Content-Length: 291
< Connection: keep-alive
< Location: http://20.124.73.73/console/login/LoginForm.jsp
< Set-Cookie: ADMINCONSOLESESSION=7CMFLsWE4y1qWH1WsumEgYnjZpB72I-j6w4B9z8VOcAGdG4Sjx7f!371331241; path=/console/; HttpOnly
<
<html><head><title>302 Moved Temporarily</title></head>
<body bgcolor="#FFFFFF">
<p>This document you requested has moved
temporarily.</p>
<p>It's now at <a href="http://20.124.73.73/console/login/LoginForm.jsp">http://20.124.73.73/console/login/LoginForm.jsp</a>.</p>
</body></html>
* Connection #0 to host 20.124.73.73 left intact
```

Access demo app:

```
$ curl http://<ingress-address>/applications/testwebapp/ --verbose
*   Trying 20.124.73.73:80...
* TCP_NODELAY set
* Connected to 20.124.73.73 (20.124.73.73) port 80 (#0)
> GET /applications/testwebapp/ HTTP/1.1
> Host: 20.124.73.73
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Date: Wed, 29 Dec 2021 07:55:27 GMT
< Content-Type: text/html; charset=UTF-8
< Content-Length: 484
< Connection: keep-alive
< Set-Cookie: JSESSIONID=tQUFL9MfwfObfooBrLei6RhNXepl28lojsCFLvrOaaKhP_3iXwaj!28580866; path=/; HttpOnly
<




<!DOCTYPE html>
<html>
  <head>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8">

    <link rel="stylesheet" href="/testwebapp/res/styles.css;jsessionid=tQUFL9MfwfObfooBrLei6RhNXepl28lojsCFLvrOaaKhP_3iXwaj!28580866" type="text/css">
    <title>Test WebApp</title>
  </head>
  <body>


    <li>InetAddress: sample-domain1-managed-server2/10.244.0.5
    <li>InetAddress.hostname: sample-domain1-managed-server2

  </body>
</html>
* Connection #0 to host 20.124.73.73 left intact
```

## Demo Video

[![Run this sample using Flux CLI]]({https://github.com/galiacheng/weblogic-aks-flux2-kustomize/resources/medias/flux-cli.mp4} "Run this sample using Flux CLI")

## Encrypt Kubernetes Secret using SOPS and Azure Key Vault

If you want to enhance the sample for development/test/production senarios, 
you are able to [encrypt Kubernetes Secret using SOPS and Azure Key Vault](https://techcommunity.microsoft.com/t5/azure-global/gitops-and-secret-management-with-aks-flux-cd-sops-and-azure-key/ba-p/2280068).
