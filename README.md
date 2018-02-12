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
rc$ az login
To sign in, use a web browser to open the page https://aka.ms/devicelogin and enter the code [foo] to authenticate.
`

Verify you can access your subscription information by running the following:

`
$ az account list --output=table
`

Note the value in the name field: you'll use that later. If you have multiple accounts and want to change the default or current active subscriptions (designated by the __IsDefault__ column)

`
$ az account set -s "[your subscription Name here]"
`

## Create Your Kubernetes Cluster

To c

Get the node info with 

`
[localhost]: kubectl get nodes
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