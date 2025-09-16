# GemFire on K8s
Instructions to run GemFire (TGF) in Kubernetes (K8s).  We will be deploying TGF to K8s locally on your laptop using the TGF Operator which, by default, deploys one locator and two servers.  Any compliant K8s distribution should work for this (e.g., Kind, MiniKube, Colima, etc).  I have tested this with a Kind Cluster, running on Docker, on my ARM based MacBook Pro.

# Mac/OSX Setup
The below will work on both ARM and Intel based Macs.  It should also work on Linux or a K8s cluster on Windows but some modifications to the prerequisits or commands may be required.  Also, if you have a K8s deployment running elsewhere (e.g., in a lab) you can use that instead of a local deployment.

## Prerequisits
1. Docker installed and running (e.g., Homebrew or [Docker Desktop](https://www.docker.com/products/docker-desktop/)) (podman *'should'* also work)
```shell
brew install docker
```
2. kubectl installed (Homebrew)
```shell
brew install kubectl
```
3. Install Helm (Homebrew)
```shell
brew install helm
```
4. (optional) Install kubectx and kubens (Homebrew)
```shell
brew install kubectx
```

## Export Your Repository Credentials
1. Login to [support.broadcom.com](http://support.broadcom.com)
2. Select 'My Downloads'
3. Search for GemFire and Select ["VMware Tanzu GemFire"](https://support.broadcom.com/group/ecx/productdownloads?subfamily=VMware%20Tanzu%20GemFire). (**Not** "VMware Tanzu GemFire on Kuberntes"; yes, seriously)
4. Expand (click on the right arrow '**>**') "VMware Tanzu GemFire"
5. Scroll down and click on the "Show All Releases" button
6. Navigate to the row named "Click Green Token for Repository Access" and click on the green shield
![green shield](images/greenshield.png)
7. View your Repository Credentials ('User Name' and 'Password / Access Token')
8. Export your Credentials
```shell
export TGF_REPO_USER='user.email@company.com'
```
```shell
export TGF_REPO_PASSWORD='****'
```

## Start K8s
1. Start Kind Cluster
```shell
kind create cluster
```
2. Kind will set the correct K8s context for you
2. Validate K8s is up and running 
```shell
kubectl get nodes
```

## Deploy the TGF Operator
[Official Docs Here](https://techdocs.broadcom.com/us/en/vmware-tanzu/data-solutions/tanzu-gemfire-on-kubernetes/2-5/gf-k8s/index.html) (for reference)

Simple Lab Instructions:
1. Create a new namespace
```shell
kubectl create ns tgf
```
2. Install Cert Manager
```shell
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.yaml
```
3. Helm Login to the Repo
```shell
helm registry login -u $TGF_REPO_USER -p $TGF_REPO_PASSWORD registry.packages.broadcom.com
```
4. Set Docker Secret for the Repo
```shell
kubectl create secret docker-registry image-pull-secret -n tgf --docker-server=registry.packages.broadcom.com --docker-username=$TGF_REPO_USER --docker-password=$TGF_REPO_PASSWORD
```
5. Helm Deploy the TGF Operator
```shell
helm install gemfire-crd oci://registry.packages.broadcom.com/tanzu-gemfire-for-kubernetes/gemfire-crd --version 2.5.0 --namespace tgf --set operatorReleaseName=gemfire-operator
helm install gemfire-operator oci://registry.packages.broadcom.com/tanzu-gemfire-for-kubernetes/gemfire-operator --version 2.5.0 --namespace tgf
```
6. Wait until the pod is ready (1/1) and RUNNING
```shell
kubectl get po -n tgf #add ‘-w’ to watch
```
  - details
    ```shell
    kubectl describe po gemfire-operator-controller-manager-<guid> -n tgf
    ```
  - logs
    ```shell
    kubectl logs gemfire-operator-controller-manager-<guid> -n tgf #add '-f' to tail
    ```

## Deploy TGF Cluster
5. Clone this Git Repository
```shell
git clone https://github.com/dbeauregard/gemfire-k8s.git
cd gemfire
```
3. Create the GemFire Cluster (uses [gemfire1.yaml](gemfire1.yaml). Take a look!)
```shell
kubectl create -f gemfire1.yaml -n tgf
```
4. Check the status of the GemFire cluster and pods.  By default there will be 1 locator and 2 servers.  Wait for them to be running.
```shell
kubectl get pods -n tgf #add ‘-w’ to watch
```
5. You can also watch the operator logs with `kubectl logs gemfire-operator-controller-manager-<guid> -n tgf` (add ‘-f’ at the end to tail the logs) 
6. Wait until the GemFire instance is ready (1/1 LOCATORS and 2/2 SERVERS)
```shell
kubectl get gemfireclusters -n tgf
```

## Connect with GFSH CLI
1. Get truststore/keystore credential (TLS_PASSWORD below)
```shell
kubectl get secret gemfire1-cert -n tgf -o jsonpath='{.data.password}' | base64 -d #ignore any shell appended % signs
```
2. Exec into the Locator Pod/Container:
```shell
kubectl exec -it gemfire1-locator-0 -n tgf -- gfsh
```
3. Connect to the TGF Cluster (in gfsh, run the following)
```
connect --locator=gemfire1-locator-0.gemfire1-locator.tgf.svc.cluster.local[10334] --trust-store=/certs/truststore.p12 --trust-store-password=TLS_PASSWORD --key-store=/certs/keystore.p12 --key-store-password=TLS_PASSWORD
```
 - Accept the defaults for all the (5) prompts
4. List the TGF Cluster Members (in gfsh, run the following)
```shell
list members
```
5. See [data.md](data.md) for more GFSH commands and a simple example
6. Type `exit` to exit gfsh and the container

## Deploy the GemFire Management Console
[Official Docs Here](https://techdocs.broadcom.com/us/en/vmware-tanzu/data-solutions/tanzu-gemfire-management-console/1-4/gf-mc/index.html) (for reference)

Simple Lab Instructions:
1. Create the TGF Management Console instance (uses [tgfmgmt.yaml](tgfmgmt.yaml). Take a look!)
```shell
kubectl create -f tgfmgmt.yaml -n tgf
```
2. Check the pod status and wait for it to be running
```shell
kubectl get pods -n tgf
```
3. Port-forward to the pod
```shell
kubectl port-forward pod/gmc-0 -n tgf 8080
```
4. In a browser navigate to the url http://127.0.0.1:8080
5. Leave "Management Login Security Provider" as "None" and click "Enable Developer Mode"
6. Instructions to connect to the TGF Cluster coming soon

## Cleanup
1. Stop Kind (pauses Kind and the K8s cluster; can be restarted later)
    - (On OSX) go to the Docker Desktop Dashboard
    - Select 'Containers'
    - In the 'Actions' Column click on the 3 vertical dots and select 'Pause'
    - You can follow this same process to un-pause it later
2. Delete Kind Cluster (perminately deletes the K8s cluster)
```shell
kind delete cluster
```