# How to use docker with swarm
This topic shows a very simple way to use [docker engine with in swarm mode](https://docs.docker.com/engine/swarm/) to create a swarm-managed cluster on Azure. It creates three virtual machines in Azure, one to act as the swarm manager, and two as part of the cluster of docker hosts. When you are finished, you can use swarm to see the cluster and then begin to use docker on it. 

> This topic uses docker with swarm and the Azure CLI *without* using swarm features built into **docker-machine** in order to show how the different tools work together but remain independent. **docker-machine** has **--swarm** switches that enable you to use **docker-machine** to directly add nodes to a swarm. For an example, see the [docker-machine](https://github.com/docker/machine) documentation. However, we will be using the non-swarm related **docker-machine** commands to deploy the machines to Azure and to connect to them remotely to manage the swarm.

For the differences between Docker Swarm and Docker Swarm Mode, there is a great explaination [here](http://stackoverflow.com/questions/38474424/the-relation-between-docker-swarm-and-docker-swarmkit/38483429).

## Install Azure CLI 2
You will need to install [Azure CLI 2.0](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) for this lab.

> Note for Windows users: Please run all the commands from the ```Git Bash``` prompt that is installed with [Git For Windows](https://git-for-windows.github.io/).  You can download it from the page if you do not have it.  Use all the defaults when installing.

## Create docker hosts with Azure Virtual Machines

This topic creates three VMs, but you can use any number you want. Before you run the script login into the Azure CLI and get your subscription Id.

**Login to the Azure CLI**

    az login

**Get your subscription Id**

    az account list

**Build Swarm VMs**
Use the environment variables we set previously or put your Subscription ID from above in the code before running these commands.  You may also want to change the azure location, resource group name and machine names to your liking by modifying the deployment script.

Clone this repository and move to the deployment folder:

    $ git clone https://github.com/billpratt/docker-azure-workshop.git
    $ cd deployment

Run the deployment script and you will be prompted for your ```Subscription Id or Name```, ```Resource Group Name```, and ```Admin Password```:

    $ sh ./deploy.sh
    Subscription Id:
    youraccountidfromabove
    ResourceGroupName:
    dockerswarm
    Enter an admin password for vms
    Admin Password:
    yourpassword

This will take 5-10 minutes.  

When you're done you should be able to use **az vm list** to see your Azure VMs:

    $ az vm list --resource-group $AZ_RESOURCE_GROUP --output table

    Name         ResourceGroup     Location
    -----------  -------------     --------
    swarm-leader  DOCKERSWARM2     eastus
    swarm-node-1  DOCKERSWARM2     eastus
    swarm-node-2  DOCKERSWARM2     eastus
    swarm-node-3  DOCKERSWARM2     eastus

**More environment variables!**

NEED DIRECTIONS ON HOW TO GET IP ADDRESS 

Since we will be typing the VM names a bit, lets set them as environment variables

    export MASTER_VM_NAME=swarm-leader-bp-demo01
    export WKR_VM_NAME1=swarm-node-bp-demo01
    export WKR_VM_NAME2=swarm-node-bp-demo02

## Installing swarm on the swarm master VM

For this step, you will be using SSH to send commands to the **swarm-master** from your Laptop using Docker. Unlike the standalone version of Docker Swarm, we do not need a respostitory/discovery service or a cluster ID to support the formation of the cluster.

Note for Windows users please use the ```Git Bash``` prompt that is installed with [Git For Windows](https://git-for-windows.github.io/).

You will need to initialize the swarm on the master node and tell it to listen on its private address.

First, let's get the private IP of the master node.

    $ ssh $MASTER_VM_NAME 
    $ ifconfig | grep eth0 -A 2
    
This will return the internal IP on the network card. You should see the following:

    eth0      Link encap:Ethernet  HWaddr 00:0d:3a:17:82:b2  
              inet addr:192.168.0.4  Bcast:192.168.255.255  Mask:255.255.0.0
              inet6 addr: fe80::20d:3aff:fe17:82b2/64 Scope:Link

The `inet addr` is the internal/private IP you will use in this next step. Now, initialize the swarm that private IP.

    $ docker swarm init --advertise-addr <PRIVATE IP>

    Swarm initialized: current node (1i8hnn4v7msygj2z7nrk2p7zu) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-20sy0fq77wiu8pqx5dosb8xb2o6pf1o4j97bxmp5w6d0e9mn73-0hxovzt3xf7p0yu6n7bs9f61n \
    192.168.0.5:2377

Copy this command for use in the next step.

## Join the node VMs to your docker cluster

For each node you need to add to the swarm, run the command provided from the manager. 

    $ ssh $WKR_VM_NAME1 
    $ docker swarm join --token SWMTKN-1-20sy0fq77wiu8pqx5dosb8xb2o6pf1o4j97bxmp5w6d0e9mn73-0hxovzt3xf7p0yu6n7bs9f61n <MASTER PRIVATE IP>:2377
    This node joined a swarm as a worker.
    $ exit
    
    $ ssh $WKR_VM_NAME2 
    $ docker swarm join --token SWMTKN-1-20sy0fq77wiu8pqx5dosb8xb2o6pf1o4j97bxmp5w6d0e9mn73-0hxovzt3xf7p0yu6n7bs9f61n <MASTER PRIVATE IP>:2377
    This node joined a swarm as a worker.
    $ exit

To check your work, run:

    $ ssh $MASTER_VM_NAME 
    $ docker info

Look for the following information within the resulting output:

    Swarm: active
     NodeID: 1i8hnn4v7msygj2z7nrk2p7zu
     Is Manager: true
     ClusterID: 78f3x9oea40piz6rai37srgdv
     Managers: 1
     Nodes: 3

## Begin managing the swarm cluster

While ssh'd into leader, To list out your nodes in your cluster:

    $ docker node ls

    ID                           HOSTNAME                STATUS  AVAILABILITY  MANAGER STATUS
    upsysx8lv4ow54fz6d33mc5m4 *  swarm-leader            Ready   Active        Leader
    x5rf6xgo1j3o69hneejv7noao    swarm-node-1            Ready   Active        
    yc8smnoayl85pvc43287jn47d    swarm-node-2            Ready   Active

Note: The * shows which VM you ran the command on        

## Deploy a demo web app container ##
While ssh'd into leader, deploy containers run the following command:

    $ docker service create --replicas 1 --name my_web --publish 8080:5000 chrch/docker-pets

What we just did:
 
 - created a new [Docker service](https://docs.docker.com/engine/swarm/how-swarm-mode-works/services/) named **my_web**
 - deployed 1 container replica using the Docker image chrch/docker-pets from [Docker Hub](hub.docker.com)
 - mapped the internal port that the web app is listening on (5000) to the external port 8080

Check the service is running:

    $ docker service ls

    ID                  NAME                MODE                REPLICAS            IMAGE
    51hckk26443q        my_web              replicated          1/1                 chrch/docker-pets:latest

See what node the container is running on:

    $ docker service ps my_web

Scale the service up to three nodes:

    $ docker service scale my_web=3

Inspect the details of the service. If you leave off the "pretty" switch, you'll get a response in JSON:

    $ docker service inspect --pretty my_web

Check again which nodes the container is running on after scaling:

    $ docker service ps my_web

Check that your website is accessible via the external IP address of the leader/master node. (The other nodes are inaccessible via port 8080 unless the port is opened on the firewall in Azure.) 

Open a web browser to the returned IP address with the leader web server's exposed port (e.g. `http://40.117.195.87:8080`, in this case).

Then, delete the service:

    $ docker service rm my_web

## Next steps

1. Learn more about Docker Swarm Mode - [https://docs.docker.com/engine/swarm/](https://docs.docker.com/engine/swarm/) , or perhaps a [video](https://www.youtube.com/watch?v=KC4Ad1DS8xU)
2. Run through more labs from DockerCon 2017 - [https://github.com/docker/dcus-hol-2017](https://github.com/docker/dcus-hol-2017)

## Notes

In order to save credits in your Azure subscription or reduce costs, it's recommended that you STOP or DELETE the VMs after the completion of the lab.  If you stop the VMs and turn them back on at a future time, the external IP addresses will likely change.  You will need to use docker-machine regenerate-certs to update you SSH access.
