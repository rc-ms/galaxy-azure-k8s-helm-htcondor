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
`

We are now ready to begin interacting with our newly-created cluster.  First, let's make sure we're all connected up. To do that we'll start using `kubectl`, the [Kubernetes command-line interface for interacting with k8s clusters](https://kubernetes.io/docs/reference/kubectl/overview/). We'll start with `kubectl get nodes`.

```
kubectl get nodes
NAME                       STATUS    ROLES     AGE       VERSION
aks-nodepool1-37476279-0   Ready     agent     1h        v1.7.7
aks-nodepool1-37476279-1   Ready     agent     1h        v1.7.7
aks-nodepool1-37476279-2   Ready     agent     1h        v1.7.7
```

## Post-create cluster configuration

For this Galaxy implementations, we're going to directly connect to the Kubernetes agents. One of the agent nodes will be configured as a storage node with an [NFS server](https://help.ubuntu.com/lts/serverguide/network-file-system.html). Shared files will be hosted in an `/export` directory, and the remaining nodes will mount that `/export` directory as NFS clients.

First, use `kubectl` to designate one node (we'll use the '-0' node) for storage (remember to replace the commands below with the __Name__ of the nodes in your k8s cluster).

 ```
 kubectl label nodes aks-nodepool1-37476279-0 type=store
 node "aks-nodepool1-37476279-0" labeled
 ```

### Find your agent pool

Since it's possible, even likely, that the resource group AKS created to house your cluster is *different* than the one you specified earlier using the Azure CLI, let's first list our resource groups

`
az group list -o table
`
You should see the resource group you created initially in the results. Check as well for another resource group with a concatenation of your group and k8s cluster, as happened with me this go-round.

```
az group list -o table
Name                                    Location        Status
--------------------------------------  --------------  ---------

k8sGalaxy                               centralus       Succeeded
MC_k8sGalaxy_ansAlsGalaxy_centralus     centralus       Succeeded
```
My suspicion is that the agents are in the concatenated version. :)

`
az resource list -g MC_k8sGalaxy_ansAlsGalaxy_centralus -o table
`
Look for your agent pool VMs above in the list that results.  Are they there?  Good.  No? You sure your cluster creation was successful?  Maybe look in another resource group. Once you do find them, let's get some additional info about them:

`
az vm show -g MC_k8sGalaxy_ansAlsGalaxy_centralus -n aks-nodepool1-37476279-0
`

I'll spare you the full dump of data; suffice to say it's a lot. 

### Create public IPs for SSH access

Cool. Now that we've verified our agents are there and accessible; lets start creating a public IP for them so we can SSH into them.

`
az network public-ip create -g MC_k8sGalaxy_ansAlsGalaxy_centralus -n galaxy-ip
`

We can get the list of IPs available now with:

```
az network nic list -g MC_k8sGalaxy_ansAlsGalaxy_centralus -o table
az network nic list -g MC_k8sGalaxy_ansAlsGalaxy_centralus -o table
EnableIpForwarding    Location    MacAddress         Name                          Primary    ProvisioningState    ResourceGroup                        ResourceGuid
--------------------  ----------  -----------------  ----------------------------  ---------  -------------------  -----------------------------------  ------------------------------------
True                  centralus   00-0D-3A-92-21-9A  aks-nodepool1-37476279-nic-0  True       Succeeded            MC_k8sGalaxy_ansAlsGalaxy_centralus  f7bb5530-5ca2-435a-b5b3-609d728bf435
True                  centralus   00-0D-3A-91-EC-9A  aks-nodepool1-37476279-nic-1  True       Succeeded            MC_k8sGalaxy_ansAlsGalaxy_centralus  277dc614-0b0e-41c0-8372-0383b6798554
True                  centralus   00-0D-3A-92-2A-4C  aks-nodepool1-37476279-nic-2  True       Succeeded            MC_k8sGalaxy_ansAlsGalaxy_centralus  ed39e5a1-93ae-445f-89b3-2827c8bcb52d
```

And now we can take the **Name** from the nic list and ask about its associated ipconfig information.

`
az network nic ip-config list --nic-name aks-nodepool1-37476279-nic-0 -g MC_k8sGalaxy_ansAlsGalaxy_centralus
`

Okay. Just a little longer.

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