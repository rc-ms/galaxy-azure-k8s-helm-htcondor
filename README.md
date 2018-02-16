<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Galaxy Kubernetes Cluster on Azure Using Helm, Charts, NFS, and HTCondor](#galaxy-kubernetes-cluster-on-azure-using-helm-charts-nfs-and-htcondor)
  - [Overview](#overview)
  - [Setup](#setup)
    - [Install the Azure CLI Tools](#install-the-azure-cli-tools)
      - [Mac](#mac)
      - [Windows](#windows)
      - [Linux](#linux)
    - [Verify your Azure setup](#verify-your-azure-setup)
  - [Create Your AKS Kubernetes Cluster](#create-your-aks-kubernetes-cluster)
    - [Install the AKS CLI](#install-the-aks-cli)
    - [Check out your cluster](#check-out-your-cluster)
  - [Post-create cluster configuration](#post-create-cluster-configuration)
    - [Find your agent pool](#find-your-agent-pool)
    - [Create public IPs for SSH access](#create-public-ips-for-ssh-access)
    - [Configure storage node and set up NFS Server](#configure-storage-node-and-set-up-nfs-server)
    - [Set remaining nodes as NFS clients](#set-remaining-nodes-as-nfs-clients)
  - [Install Galaxy via a Helm Chart](#install-galaxy-via-a-helm-chart)
    - [Install Helm and Tiller](#install-helm-and-tiller)
    - [Choose Your Galaxy Flavor](#choose-your-galaxy-flavor)
    - [Post-install: Fix your permissions](#post-install-fix-your-permissions)
    - [Configure HTCondor](#configure-htcondor)
    - [Configure a Public Static IP](#configure-a-public-static-ip)
  - [Troubleshooting](#troubleshooting)
    - [Stopping and Restarting Galaxy](#stopping-and-restarting-galaxy)
    - [Updating Tools](#updating-tools)
    - [Upgrading Kubernetes](#upgrading-kubernetes)
  - [Additional Resources and Links](#additional-resources-and-links)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Galaxy Kubernetes Cluster on Azure Using a Helm Chart, NFS Server, and HTCondor

## Overview

This readme describes how to build and run a Galaxy server cluster on Azure using Kubernetes, HTCondor, Helm, and some post install incantations and configurations. It was built to support the [Neurolincs](http://neurolincs.org/) epigenome workflow of the [Fraenkel Lab at MIT](http://fraenkel.mit.edu/) in support of the [AnswerALS Foundation research plan](http://research.answerals.org/) seeking a cure for [Amyotrophic Lateral Sclerosis](https://en.wikipedia.org/wiki/Amyotrophic_lateral_sclerosis).

The acceptance criteria for this configuration included the following:

- At least 60 TB of accessible storage for genome data and algorithm output
- FTP, Galaxy web UI import, and direct file upload options
- Sufficient peformance for uploaded data
- Dynamic scaling
- [Kubernetes](https://kubernetes.io/) orchestration
- Package management via [Helm](https://helm.sh/) chart for external configurability, consistency with Galaxy Cloudman next-generation support
- NFS server/client configuration for Galaxy file-management support

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

- You don't pay for the Kubernetes master VMs because the AKS control plane is free
- You can scale and upgrade your Kubernetes clusters easily and in place
- Full "upstream" Kubernetes API support
- Configurations that use AKS spin up and consume fewer compute resources than previous ACS-based versions

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

``` bash
kubectl get nodes
NAME                       STATUS    ROLES     AGE       VERSION
aks-nodepool1-37476279-0   Ready     agent     1h        v1.7.7
aks-nodepool1-37476279-1   Ready     agent     1h        v1.7.7
aks-nodepool1-37476279-2   Ready     agent     1h        v1.7.7
```

## Post-create cluster configuration

For this Galaxy implementations, we're going to directly connect to the Kubernetes agents. One of the agent nodes will be configured as a storage node with an [NFS server](https://help.ubuntu.com/lts/serverguide/network-file-system.html). Shared files will be hosted in an `/export` directory, and the remaining nodes will mount that `/export` directory as NFS clients.

First, use `kubectl` to designate one node (we'll use the '-0' node) for storage (remember to replace the commands below with the __Name__ of the nodes in your k8s cluster).

 ``` bash
 kubectl label nodes aks-nodepool1-37476279-0 type=store
 node "aks-nodepool1-37476279-0" labeled
 ```

### Find your agent pool

Since it's possible, even likely, that the resource group AKS created to house your cluster is *different* than the one you specified earlier using the Azure CLI, let's first list our resource groups

`
az group list -o table
`
You should see the resource group you created initially in the results. Check as well for another resource group with a concatenation of your group and k8s cluster, as happened with me this go-round.

``` bash
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

Look for your agent pool VMs in the list that results.  Are they there?  Good.  No? You sure your cluster creation was successful?  Maybe look in another resource group. Once you do find them, let's get some additional info about them:

`
az vm show -g MC_k8sGalaxy_ansAlsGalaxy_centralus -n aks-nodepool1-37476279-0
`

I'll spare you the full dump of data; suffice to say it's a lot. But we're good, right?

### Create public IPs for SSH access

(Note that this section plagiarizes pretty much entirely from [this great how-to](https://gist.github.com/tsaarni/624d5406e442f08fe11083169c059a68).)

Okay. Now that we've verified our agents are there and accessible; lets start creating a public IP *for each node* so we can enable SSH access.

`
az network public-ip create -g MC_k8sGalaxy_ansAlsGalaxy_centralus -n node0-ip
`

We can also get the list of IPs available:

``` bash
az network nic list -g MC_k8sGalaxy_ansAlsGalaxy_centralus -o table
az network nic list -g MC_k8sGalaxy_ansAlsGalaxy_centralus -o table
EnableIpForwarding    Location    MacAddress         Name                          Primary    ProvisioningState    ResourceGroup                        ResourceGuid
--------------------  ----------  -----------------  ----------------------------  ---------  -------------------  -----------------------------------  ------------------------------------
True                  centralus   00-0D-3A-92-21-9A  aks-nodepool1-37476279-nic-0  True       Succeeded            MC_k8sGalaxy_ansAlsGalaxy_centralus  f7bb5530-5ca2-435a-b5b3-609d728bf435
True                  centralus   00-0D-3A-91-EC-9A  aks-nodepool1-37476279-nic-1  True       Succeeded            MC_k8sGalaxy_ansAlsGalaxy_centralus  277dc614-0b0e-41c0-8372-0383b6798554
True                  centralus   00-0D-3A-92-2A-4C  aks-nodepool1-37476279-nic-2  True       Succeeded            MC_k8sGalaxy_ansAlsGalaxy_centralus  ed39e5a1-93ae-445f-89b3-2827c8bcb52d
```

Almost there! Now we take the **Name** from the nic list and ask about its associated ipconfig information.

`
az network nic ip-config list --nic-name aks-nodepool1-37476279-nic-0 -g MC_k8sGalaxy_ansAlsGalaxy_centralus
`

Now we can add the public IP to the ipconfig file. (Crikey. ipconfig? Yes. But not forever, okay?) Note that the `--name ipconfig1` parameter is in the response above; it *should* be ipconfig1 but if for some reason it isn't, check the **Name** field.

``` bash
az network nic ip-config update -g MC_k8sGalaxy_ansAlsGalaxy_centralus --nic-name aks-nodepool1-37476279-nic-0 --name ipconfig1 --public-ip-address node0-ip
```

Assuming this update is successful (which it should be), you can now ask about the public ip that was created for this node.

`
az network public-ip show -g MC_k8sGalaxy_ansAlsGalaxy_centralus -n node0-ip
`

Woo hoo!  There it is. Our very own IP address. Now we can SSH into node 0.

Before leaving, let's set up SSH access for the other two nodes that were created. Thanks to command history we can just circle back and swap out -1 and then -2 for each command. Remember that you will need to create a new public IP for each node, as well as update each ipconfig file to incorporate the new IP. (And yes, we really should script this.) Now remember those IP addresses for our next set of work.

### Configure storage node and set up NFS Server

What did we want to do again? Right: SSH into the nodes. We'll start with our storage node.

`
ssh [your_username]@[your.node0.IP.address]
`

Remember, the username and password are the accounts you created in the Azure Portal.

Let's get some proper priviledges.

``` bash
$ sudo su
[sudo] password for rc:
root@aks-nodepool1-37476279-0:/home/rc#
```

First, we're going to create an `/export` directory.

``` bash
root@aks-nodepool1-37476279-0:/home/rc# mkdir /export
root@aks-nodepool1-37476279-0:/home/rc# chown nobody:nogroup /export
```

Now we'll install and start NFS server.

``` bash
root@aks-nodepool1-37476279-0:/home/rc# sudo apt install nfs-kernel-server
```

After install, add the `/export` directory to the list of directories eligible for nfs mount with both read and write privileges. We'll do it by adding the following entries to the `/etc/exports` file using vi (if your vi is rusty, [i find this page helpful](http://www.lagmonster.org/docs/vi.html)).

``` bash
root@aks-nodepool1-37476279-0:/home/rc# vi /etc/exports
```

Once you've placed your cursor where you want (don't forget to press the 'i' key :) ), copy the below and then save (`Esc`, `wq:`):

``` vi
      /ubuntu  *(ro,sync,no_root_squash)
      /export  *(rw,sync,no_root_squash)
```

And now for the `hosts.allow` file

``` bash
vi /etc/hosts.allow
```

Copy and paste the following:

``` vi
   ALL: ALL
```

Ok. Now we're ready to start the service.

``` bash
root@aks-nodepool1-37476279-0:/etc# sudo systemctl start nfs-kernel-server.service
```

### Set remaining nodes as NFS clients

We should know the SSH dance pretty well now. For the remaining nodes, we're going to also create an `/export` directory, but instead of NFS Server incantations, they'll be NFS clients and mount the server's `/export` directory.

Why do we have to do this, you ask? (Or maybe you don't.) Regardless, we're going through these steps because of the way Galaxy handles files in its current state. Our experience shows the most performant configuration is enabling NFS support, especially if we want to have files uploaded through the Galaxy UI be available for scalable compute jobs.

(FYI, I find it helpful to run these commands in a new terminal window, keeping the storage node session open in case I need to stop and restart the NFS server.)

Once connected to one of your remaining nodes, `sudo su` and create an `/export` directory again:

``` bash
root@aks-nodepool1-37476279-1:/home/rc# mkdir /export
```

Now, though, we're going to mount the storage node's `/export` directory.

``` bash
sudo mount aks-nodepool1-37476279-0:/export /export
```

If you get a `permission denied` message, return to your storage node and stop and restart the NFS service. I don't know why this works, but it does.

``` bash
root@aks-nodepool1-37476279-0:/etc# sudo systemctl stop nfs-kernel-server.service
root@aks-nodepool1-37476279-0:/etc# sudo systemctl start nfs-kernel-server.service
```

## Install Galaxy via a Helm Chart

Bet you thought we'd never get here. I know I did. Now we're going to get ready to build and deploy Galaxy via a Helm Chart.

### Install Helm and Tiller

Ok but first let's [install Helm](https://docs.helm.sh/using_helm/#installing-helm). For Mac, we'll use good old `brew`.

``` bash
brew install kubernetes-helm
```

With Helm installed, we'll install Tiller by running `helm init`.

``` bash
helm init
```

### Choose Your Galaxy Flavor

Now you're (finally!) ready to start working with Galaxy! This is a critical moment. Which Galaxy do you want to install into your cluster? If you have cloned this repository and you navigated to its directory in Terminal, you can install this repo into your new cluster with the following command.

``` bash
helm install galaxy
```

[Other installation options obtain as well](https://docs.helm.sh/using_helm/#more-installation-methods).

### Post-install: Fix your permissions

While your Galaxy may be installed and running, it will also likely throw some permission issues if you tried to load it in your browser. So let's nip that in the bud.

If you haven't kept your storage node SSH session open by chance, let's connect again and run these commands on your `/export` directory

``` bash
root@aks-nodepool1-37476279-0: cd /export
root@aks-nodepool1-37476279-0: chmod 777 -R *
```

To set up your k8s cluster to load the Galaxy web UI in your local browser, run this command on your local computer (not one of the agent nodes).

``` bash
kubectl port-forward galaxy 8080:80
```

Now you can open your browser and point it at the URL you specified above (in this case you are forwarding the Galaxy response to port 8080 so enter the URL [http://localhost:8080](http://localhost:8080) in your browser; if for some reason you get an error on port 8080 feel free to try another port such as 8090 or 13080).

### Configure HTCondor

Now we'll  shell to galaxy-htcondor container

``` bash
[localhost]:kubectl exec -it galaxy-htcondor -- /bin/bash
```

Edit file `/etc/hosts`

``` bash
  [root@galaxy-htcondor]: vi /etc/hosts
```

Insert the following line

``` vi
  127.0.0.1   galaxy-htcondor
```

Shell to galaxy-htcondor-executor container

``` bash
kubectl exec -it galaxy-htcondor-executor -- /bin/bash
```

Edit file /etc/hosts

``` bash
  [root@galaxy-htcondor-executor]: vi /etc/hosts
```

``` vi
  127.0.0.1   galaxy-htcondor-executor
```

Get shell to galaxy container

``` bash
kubectl exec -it galaxy --container=galaxy  -- /bin/bash
```

Edit  `/etc/condor/condor_config.local`

  ``` bash
  [root@galaxy-htcondor]: vi /etc/condor/condor_config.local
  ```

Copy and paste the following

``` vi
  HOSTALLOW_READ = *
  HOSTALLOW_WRITE = *
  HOSTALLOW_NEGOTIATOR = *
  HOSTALLOW_ADMINISTRATOR = *
```

Restart condor.

``` bash
    [root@galaxy]:condor_restart
```

### Configure a Public Static IP

We would imagine that given you've gone to set up this awesome Galaxy server, you don't want people to have to port-forward from `kubectl` to access it. To set one up in Azure, you can do so from the portal or  To assign a static public IP to galaxy and galaxy-proftpd server run

``` bash
 [localhost]: kubectl expose pod galaxy --type=LoadBalancer
 [localhost]: kubectl expose pod galaxy-proftpd --type=LoadBalancer
```

[This article gives a nice summary on your options](http://www.techdiction.com/2017/11/22/deploying-a-kubernetes-service-on-azure-with-a-specific-ip-addresses/).

## Troubleshooting

So many places for it all to go wrong. Hopefully you've been familiarizing yourself with the different to-ing and fro-ing along the way.

### Stopping and Restarting Galaxy

If you come back to your Galaxy after some time away and you find the URL isn't loading or whatnot, here are the steps you can do to perform a "reboot" Kubernetes-style. Open your terminal and run the following commands.

``` bash
kubectl delete --all pods --all --force --grace-period=0
kubectl delete --all services --all --force --grace-period=0
kubectl delete --all deployments --all --force --grace-period=0
kubectl delete --all rc --all --force --grace-period=0
```

### Updating Tools

### Upgrading Kubernetes

## Additional Resources and Links

- [Deploy Kubernetes clusters for Linux containers](https://docs.microsoft.com/en-us/azure/container-service/kubernetes/container-service-kubernetes-walkthrough)
- [Using the Kubernetes web UI with Azure Container Service](https://docs.microsoft.com/en-us/azure/container-service/kubernetes/container-service-kubernetes-ui)