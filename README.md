# galaxy-kubernetes-htc-condor

## Overview

This repo and readme describes how to build and run a Galaxy server cluster on Azure using Kubernetes, HTCondor, Helm, and some post install incantations and configurations.

The acceptance criteria for this configuration included the following:

* At least 60 TB of accessible storage
* blah blah blah

## Setup 

### Install the Azure CLI Tools

Before we can start working on Galaxy itself, we need to set up and configure the Kubernetes environment it will run in.

Start by [creating an Azure account](https://azure.microsoft.com/).

Once you have an account, [install the Azure CLI tools](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest).

#### [Mac](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-macos?view=azure-cli-latest)

[Per this link](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-macos?view=azure-cli-latest), `brew` is the preferred way to install the Azure CLI.

`
$ brew update && brew install azure-cli
`

You can then run the Azure CLI with the `az` command from a Terminal window.

#### Windows

The recommended way to install Azure CLI tools on Windows is to use the [MSI Installer](https://azurecliprod.blob.core.windows.net/msi/azure-cli-latest.msi).

#### Linux

Depending on which version of Linux you're using, you can install using `yum`, `zypper`, `apt`, or script. Best to [check the docs](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) for your particular peccadillo.

### Verify your Azure setup

Once you've installed your tools and set up your account, let's verify everything is working correctly. Open a terminal window or command prompt and enter the following:

`
$ az login
`

You should be [prompted to go to this URL (http://aka.ms/devicelogin) and enter the code](https://aka.ms/devicelogin) referenced in your Terminal response to sign in.

`
To sign in, use a web browser to open the page https://aka.ms/devicelogin and enter the code BPV5A449L to authenticate.
`

Once you've successfully signed in, verify you can access your subscription information by running the following in your Terminal window:

`
$ az account list --output=table
`

Note the value in the name field: you'll use that later. If you have multiple accounts and want to change the default (designated by whichever subscription shows as __True__ in the __IsDefault__ column) or current active subscription, use the `set` command 

`
$ az account set -s "[your subscription Name here]"
`

## Create Your AKS Kubernetes Cluster

To create your Kubernetes cluster, we're going to use the new Amazon Container Service (AKS) optimized for Kubernetes.  Some notes as to why:

* You don't pay for the Kubernetes master VMs because the AKS control plane is free
* You can scale and upgrade your Kubernetes clusters easily and in place
* Full "upstream" Kubernetes API support
* Configurations that use AKS spin up and consume fewer compute resources than previous ACS-based versions

That's no small stuff.

Let's start by creating a resource group:

`
az group create --name k8sGalaxy --location centralus
`

Note that not all regions have all versions of VMs that might be used for Kubernetes clusters. If you're concerned about specific configurations, specifically as relates to VMs, [refer to this product availability chart](https://azure.microsoft.com/en-us/regions/services/); you may generate errors on create but the message will tell you which regions are available (at the time of this writing: eastus,westeurope,centralus,canadacentral,canadaeast).

Now that you've created your resource group, let's create a k8s cluster within it.

`
az aks create --resource-group k8sGalaxy --name ansAlsGalaxy --generate-ssh-keys
`
It may take a few minutes, but if all goes according to plan you'll see a new k8s cluster appear in your portal. Congrats!

### Install the AKS CLI

Now that we've got our shiny new cluster set up, we need to connect to it. To do that, let's install more fun tools.

`
az aks install-cli
`

If you're on a Mac and try to install and get permission issues, run again under `sudo`:

`
sudo az aks install-cli
`

(At the password prompt, enter the login for your Mac.)

### Check out your cluster

For the CLI tool to connect to your cluster, we have to download the credentials and keys necessary to do so securely, etc. Yup: [there's a command for that](https://www.youtube.com/watch?v=yYey8ntlK_E):

`
az aks get-credentials --resource-group k8sGalaxy --name ansAlsGalaxy
'

'
kubectl get nodes
NAME                       STATUS    ROLES     AGE       VERSION
aks-nodepool1-37476279-0   Ready     agent     1h        v1.7.7
aks-nodepool1-37476279-1   Ready     agent     1h        v1.7.7
aks-nodepool1-37476279-2   Ready     agent     1h        v1.7.7
`

- Let the nodes be 
  - Node master-0 
  - Node agent-0 
  - Node agent-1 
  
### Steps

- Designate one node for storage through kubectl label command 
 ```
 [localhost]:kubectl label nodes <Node agent-0> type=store
 ```
- ssh to the storage node
  - To ssh to any node, first ssh to master node with the IP available on the azure portal (create user and add password to all nodes through the azure portal, makes things easier :P) then ssh to any desired node with
    ```
    [Node master-0]:-# ssh username@Node agent-0
    ```
  
- make directory export with
  ```
  [Node agent-0]: mkdir -p /export
  [Node agent-0]: chown nobody:nogroup /export
  ```
  
- install NFS server 
  ```
  [Node agent-0]: sudo apt install nfs-kernel-server
  ```
 
- add "/export" to list of directories eligible for nfs mount with both read and write privileges
    - You can configure the directories to be exported by adding them to the /etc/exports file. For example:
      ```
      /ubuntu  *(ro,sync,no_root_squash)
      /export  *(rw,sync,no_root_squash)
      ```
- edit /etc/hosts.allow
  - add
  ```
  ALL: ALL
  ```
    
- start service with
  ```
  [Node agent-0]: sudo systemctl start nfs-kernel-server.service
  ```
 
- ssh to all other nodes
  ```
  [Node any]:-# ssh username@Node agent-(all except 0)
  [Node agent-(all except 0)]: mkdir -p /export
  [Node agent-(all except 0)]: sudo mount <Node agent-0>:/export /export
  ```
  
- Install helm chart with
  ```
  [localhost]: helm install galaxy
  ```
  
- ssh to storage node using same steps above, then, in order to get rid of the permission denied error:
  ```
  [Node agent-0]: cd /export
  [Node agent-0]: chmod 777 -R * 
  ```
  
- check if galaxy is running with
  ```
  [localhost]: kubectl port-forward galaxy 8080:80
  ```
  and then browse to localhost:8080

#### Setup Htcondor

- get shell to galaxy-htcondor container
```
[localhost]:kubectl exec -it galaxy-htcondor -- /bin/bash
```
  - edit file /etc/hosts
  ```
  [root@galaxy-htcondor]: vi /etc/hosts
  ```
  - add 127.0.0.1   galaxy-htcondor
- get shell to galaxy-htcondor-executor container
```
[localhost]:kubectl exec -it galaxy-htcondor-executor -- /bin/bash
```
  - edit file /etc/hosts
  ```
  [root@galaxy-htcondor-executor]: vi /etc/hosts
  ```
  - add 127.0.0.1   galaxy-htcondor-executor 
- get shell to galaxy container
```
[localhost]:kubectl exec -it galaxy --container=galaxy  -- /bin/bash
```
  - edit file /etc/condor/condor_config.local
  ```
  [root@galaxy-htcondor]: vi /etc/condor/condor_config.local
  ```
  - add the following lines
  ```
  HOSTALLOW_READ = *
  HOSTALLOW_WRITE = *
  HOSTALLOW_NEGOTIATOR = *
  HOSTALLOW_ADMINISTRATOR = *
  ```
  - restart condor with 
  ```
    [root@galaxy]:condor_restart
  ```
 - to assign a static public IP to galaxy and galaxy-proftpd server run 
 ```
 [localhost]: kubectl expose pod galaxy --type=LoadBalancer
 [localhost]: kubectl expose pod galaxy-proftpd --type=LoadBalancer
 ```
## Resources

* [Deploy Kubernetes clusters for Linux containers](https://docs.microsoft.com/en-us/azure/container-service/kubernetes/container-service-kubernetes-walkthrough)
* [Using the Kubernetes web UI with Azure Container Service](https://docs.microsoft.com/en-us/azure/container-service/kubernetes/container-service-kubernetes-ui)