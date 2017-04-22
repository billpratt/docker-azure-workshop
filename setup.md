# Setup Prerequisites

You'll need to follow the steps below to setup your machine for the workshop.

## Steps

* [1. Install Azure CLI 2.0](#1)
* [2. Get Docker](#2)
* [3. Install Git](#3)
* [4. Login to Azure CLI](#4)

## <a name="1"></a>Step 1: Install Azure CLI 2.0

We will be using the Azure CLI 2.0 tool

- Install it from [here](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)

## <a name="2"></a>Step 2: Get Docker

Mac

- Install [Docker for Mac](https://www.docker.com/docker-mac)

Windows 10 Professional

- Windows 10 Professional/Enterprise: Install [Docker for Windows](https://www.docker.com/docker-windows)
- Docker for Windows Requires Microsoft Windows 10 Professional or Enterprise 64-bit - (install a trial in a VM if you need to) 
  - If you don't have a supported version of Windows 10, check the previous versions of [Docker Toolbox](https://www.docker.com/products/docker-toolbox)

## <a name="3"></a>Step 3: Install Git

Mac

- Install [Git for Mac](https://www.atlassian.com/git/tutorials/install-git)

Windows

- Install [Git for Windows](https://git-for-windows.github.io/)

## <a name="4"></a>Step 4: Login to Azure CLI

Open a command prompt and use the `az login` command -- without any arguments -- to authenticate interactively with either:

- a work or school account identity that requires multi-factor authentication, or
- a Microsoft account identity when you want to access Resource Manager deployment mode functionality

Interactively logging in is easy: type `az login` and follow the prompts as shown below:

	az login                                                                                                                                              
	
	To sign in, use a web browser to open the page https://aka.ms/devicelogin and enter the code [...] to authenticate.                                   

Copy the code offered to you, above, and open a browser to http://aka.ms/devicelogin. Enter the code, and then you are prompted to enter the username and password for the identity you want to use. When that process completes, the command shell completes the log in process. It might look something like:

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




