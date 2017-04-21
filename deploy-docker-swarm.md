# Docker Swarm Mode with Azure
In this lab you will play around with the container orchestration features of Docker. You will deploy a simple application to a single host and learn how that works. Then, you will configure Docker Swarm Mode, and learn to deploy the same simple application across multiple hosts. You will then see how to scale the application and move the workload across different hosts easily.

> **Tasks**:
>
> * [Section #0 - Prerequisites/Creating VMs](#prerequisites)
> * [Section #1 - What is Orchestration](#basics)
> * [Section #2 - Configure Swarm Mode](#start-cluster)
> * [Section #3 - Deploy applications across multiple hosts](#multi-application)
> * [Section #4 - Scale the application](#scale-application)
> * [Section #5 - Drain a node and reschedule the containers](#recover-application)
> * [Cleaning Up](#cleanup)

## Document conventions

When you encounter a phrase in between `<` and `>`  you are meant to substitute in a different value. 

For instance if you see `ssh <username>@<hostname>` you would actually type something like `ssh sysadmin@swarm-leader.ivaf2i2atqouppoxund0tvddsa.jx.internal.cloudapp.net`

# <a name="prerequisites"></a>Section 0: Prerequisites/Creating VMs

For this lab we will need to create three virtual machines referred to as __swarm-leader__, __swarm-node-1__ and __swarm-node-2__. We will use Azure and a custom bash script that uses the Azure CLI to create all the VMs in one command.

## Step 0.1 - Install Azure CLI 2.0

You will need to install [Azure CLI 2.0](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) for this lab.

> Note for Windows users: Please run all the commands from the ```Git Bash``` prompt that is installed with [Git For Windows](https://git-for-windows.github.io/).  You can download it from the page if you do not have it.  Use all the defaults when installing.

## Step 0.2 - Login to the Azure CLI

Before you continue please login into the Azure CLI and get your subscription Id.

**Login to the Azure CLI**

    az login

You should see output similar to below. Follow the instructions printed on the command shell.
When that process completes, the command shell completes the log in process. It might look something like:

	[
		{
			"cloudName": "AzureCloud",
			"id": "...",
			"isDefault": true,
			"name": "Free Trial",
			"state": "Enabled",
			"tenantId": "...",
			"user": {
			"name": "...",
			"type": "user"
			}
		}
	] 

**Get your subscription Id**

    az account list

This will print out a JSON block similar to the below one showing the accounts you are currently logged into. The subscription id is found in the **id** field.

    [
        {
            "cloudName": "AzureCloud",
            "id": ".....",
            "isDefault": true,
            "name": "Free Trial",
            "state": "Enabled",
            "tenantId": "....",
            "user": {
                "name": "....",
                "type": "user"
            }
        }
    ]

Copy your subscription id because we will use it a few times.
    
**Confirm your Azure Subscription**

    az account set --subscription <YOUR SUB ID>

## Step 0.3 - Create docker hosts with Azure Virtual Machines

This topic uses three VMs, but you can use any number you want. 

**Build Swarm VMs**

Clone this repository and move to the deployment folder:

    $ git clone https://github.com/billpratt/docker-azure-workshop.git
    $ cd docker-azure-workshop/deployment

Run the deployment script and you will be prompted for your ```Subscription Id or Name```, ```Resource Group Name```, and ```Admin Password```.:

    $ sh ./deploy.sh
    Subscription Id:
    <YOUR SUB ID>
    ResourceGroupName:
    dockerswarm
    Enter an admin password for vms
    Admin Password:
    <YOUR PASSWORD>

**Important** Use a password you will remember because you will use it to SSH into the VMs later.

This will take 5-10 minutes.  

When you're done you should be able to use **az vm list** to see your Azure VMs and get the public IPs to SSH into:

    $ az vm list --resource-group <Resource Group Name Used> --output table --show-details

    Name          ResourceGroup    PowerState    PublicIps      Location
    ------------  ---------------  ------------  -------------  ----------
    swarm-leader  dockerswarm      VM running    11.11.111.111  eastus
    swarm-node-1  dockerswarm      VM running    11.11.111.112  eastus
    swarm-node-2  dockerswarm      VM running    11.11.111.113  eastus
    swarm-node-3  dockerswarm      VM running    11.11.111.114  eastus

# <a name="basics"></a>Section 1: What is Orchestration

So, what is Orchestration anyways? Well, Orchestration is probably best described using an example. Lets say that you have an application that has high traffic along with high-availability requirements. Due to these requirements, you typically want to deploy across at least 3+ machines, so that in the event a host fails, your application will still be accessible from at least two others. Obviously, this is just an example and your use-case will likely have its own requirements, but you get the idea.

Deploying your application without Orchestration is typically very time consuming and error prone, because you would have to manually SSH into each machine, start up your application, and then continually keep tabs on things to make sure it is running as you expect.

But, with Orchestration tooling, you can typically off-load much of this manual work and let automation do the heavy lifting. One cool feature of Orchestration with Docker Swarm, is that you can deploy an application across many hosts with only a single command (once Swarm mode is enabled). Plus, if one of the supporting nodes dies in your Docker Swarm, other nodes will automatically pick up load, and your application will continue to hum along as usual.

If you are typically only using `docker run` to deploy your applications, then you could likely really benefit from using Docker Compose, Docker Swarm mode, and both Docker Compose and Swarm.

# <a name="start-cluster"></a>Section 2: Configure Swarm Mode

Real-world applications are typically deployed across multiple hosts as discussed earlier. This improves application performance and availability, as well as allowing individual application components to scale independently. Docker has powerful native tools to help you do this.

A swarm comprises one or more *Manager Nodes* and one or more *Worker Nodes*. The manager nodes maintain the state of swarm and schedule application containers. The worker nodes run the application containers. As of Docker 1.12, no external backend, or 3rd party components, are required for a fully functioning swarm - everything is built-in!

In this part of the demo you will use all three of the nodes in your lab. __swarm-leader__ will be the Swarm manager, while __swarm-node-1__ and __swarm-node-2__ will be worker nodes. Swarm mode supports a highly available redundant manager nodes, but for the purposes of this lab you will only deploy a single manager node.

## Step 2.1 - Create a Manager node

Note for Windows users please use the ```Git Bash``` prompt that is installed with [Git For Windows](https://git-for-windows.github.io/).

If you haven't already done so, please SSH in to **swarm-leader**.

```
$ ssh sysadmin@<swarm-leader public IP address>
```

In this step you'll initialize a new Swarm, join a single worker node, and verify the operations worked.

Run `docker swarm init` on **swarm-leader**.

```
$ docker swarm init
Swarm initialized: current node (6dlewb50pj2y66q4zi3egnwbi) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-1wxyoueqgpcrc4xk2t3ec7n1poy75g4kowmwz64p7ulqx611ih-68pazn0mj8p4p4lnuf4ctp8xy \
    10.0.0.5:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

You can run the `docker info` command to verify that **swarm-leader** was successfully configured as a swarm manager node.

```
$ docker info
Containers: 2
 Running: 0
 Paused: 0
 Stopped: 2
Images: 2
Server Version: 17.03.1-ee-3
Storage Driver: aufs
 Root Dir: /var/lib/docker/aufs
 Backing Filesystem: extfs
 Dirs: 13
 Dirperm1 Supported: true
Logging Driver: json-file
Cgroup Driver: cgroupfs
Plugins:
 Volume: local
 Network: bridge host macvlan null overlay
Swarm: active
 NodeID: rwezvezez3bg1kqg0y0f4ju22
 Is Manager: true
 ClusterID: qccn5eanox0uctyj6xtfvesy2
 Managers: 1
 Nodes: 1
 Orchestration:
  Task History Retention Limit: 5
 Raft:
  Snapshot Interval: 10000
  Number of Old Snapshots to Retain: 0
  Heartbeat Tick: 1
  Election Tick: 3
 Dispatcher:
  Heartbeat Period: 5 seconds
 CA Configuration:
  Expiry Duration: 3 months
 Node Address: 10.0.0.5
 Manager Addresses:
  10.0.0.5:2377
<Snip>
```

The swarm is now initialized with **swarm-leader** as the only Manager node. In the next section you will add **swarm-node-1** and **swarm-node-2** as *Worker nodes*.

##  Step 2.2 - Join Worker nodes to the Swarm

You will perform the following procedure on **swarm-node-1** and **swarm-node-2**. Towards the end of the procedure you will switch back to **swarm-leader**.

Open a new SSH session to __swarm-node-1__ (Keep your SSH session to **swarm-leader** open in another tab or window).

```
$ ssh sysadmin@<swarm-node-1 public IP address>
```

Now, take that entire `docker swarm join ...` command we copied earlier from `swarm-leader` where it was displayed as terminal output. We need to paste the copied command into the terminal of **swarm-node-1** and **swarm-node-2**.

It should look something like this for **swarm-node-1**. By the way, if the `docker swarm join ...` command scrolled off your screen already, you can run the `docker swarm join-token worker` command on the Manager node to get it again.

```
$ docker swarm join \
    --token SWMTKN-1-1wxyoueqgpcrc4xk2t3ec7n1poy75g4kowmwz64p7ulqx611ih-68pazn0mj8p4p4lnuf4ctp8xy \
    10.0.0.5:2377
```

Again, ssh into **swarm-node-2** and it should look something like this.

```
$ ssh sysadmin@<swarm-node-2 public IP address>
```

```
$ docker swarm join \
    --token SWMTKN-1-1wxyoueqgpcrc4xk2t3ec7n1poy75g4kowmwz64p7ulqx611ih-68pazn0mj8p4p4lnuf4ctp8xy \
    10.0.0.5:2377
```

Once you have run this on **swarm-node-1** and **swarm-node-2**, switch back to **swarm-leader**, and run a `docker node ls` to verify that both nodes are part of the Swarm. You should see three nodes, **swarm-leader** as the Manager node and **swarm-node-1** and **swarm-node-2** both as Worker nodes.

```
$ docker node ls
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
6dlewb50pj2y66q4zi3egnwbi *  swarm-leader   Ready   Active        Leader
ym6sdzrcm08s6ohqmjx9mk3dv    swarm-node-2   Ready   Active
yu3hbegvwsdpy9esh9t2lr431    swarm-node-1   Ready   Active
```

The `docker node ls` command shows you all of the nodes that are in the swarm as well as their roles in the swarm. The `*` identifies the node that you are issuing the command from.

Congratulations! You have configured a swarm with one manager node and two worker nodes.

# <a name="multi-application"></a>Section 3: Deploy applications across multiple hosts


REMOVE BELOW!!!

For this step, you will be using SSH to send commands to the **swarm-leader** from your Laptop using Docker. Unlike the standalone version of Docker Swarm, we do not need a respostitory/discovery service or a cluster ID to support the formation of the cluster.

You will need to initialize the swarm on the leader node and tell it to listen on its private address.

First, let's get the private IP of the leader node.

    $ ssh $LEADER_VM_NAME 
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
    $ docker swarm join --token SWMTKN-1-20sy0fq77wiu8pqx5dosb8xb2o6pf1o4j97bxmp5w6d0e9mn73-0hxovzt3xf7p0yu6n7bs9f61n <LEADER PRIVATE IP>:2377
    This node joined a swarm as a worker.
    $ exit
    
    $ ssh $WKR_VM_NAME2 
    $ docker swarm join --token SWMTKN-1-20sy0fq77wiu8pqx5dosb8xb2o6pf1o4j97bxmp5w6d0e9mn73-0hxovzt3xf7p0yu6n7bs9f61n <LEADER PRIVATE IP>:2377
    This node joined a swarm as a worker.
    $ exit

To check your work, run:

    $ ssh $LEADER_VM_NAME 
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

Check that your website is accessible via the external IP address of the leader node. (The other nodes are inaccessible via port 8080 unless the port is opened on the firewall in Azure.) 

Open a web browser to the returned IP address with the leader web server's exposed port (e.g. `http://40.117.195.87:8080`, in this case).

Then, delete the service:

    $ docker service rm my_web

## Next steps

1. Learn more about Docker Swarm Mode - [https://docs.docker.com/engine/swarm/](https://docs.docker.com/engine/swarm/) , or perhaps a [video](https://www.youtube.com/watch?v=KC4Ad1DS8xU)
2. Run through more labs from DockerCon 2017 - [https://github.com/docker/dcus-hol-2017](https://github.com/docker/dcus-hol-2017)

## Notes

In order to save credits in your Azure subscription or reduce costs, it's recommended that you STOP or DELETE the VMs after the completion of the lab.  If you stop the VMs and turn them back on at a future time, the external IP addresses will likely change.  You will need to use docker-machine regenerate-certs to update you SSH access.
