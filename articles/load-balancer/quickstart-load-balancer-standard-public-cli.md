---
title: "Quickstart: Create a public load balancer - Azure CLI"
titleSuffix: Azure Load Balancer
description: This quickstart shows how to create a public load balancer using the Azure CLI
services: load-balancer
documentationcenter: na
author: asudbring
manager: KumudD
tags: azure-resource-manager
Customer intent: I want to create a load balancer so that I can load balance internet traffic to VMs.
ms.service: load-balancer
ms.devlang: na
ms.topic: quickstart
ms.tgt_pltfrm: na
ms.workload: infrastructure-services
ms.date: 07/20/2020
ms.author: allensu
ms.custom: mvc
---
# Quickstart: Create a public load balancer to load balance VMs using Azure CLI

Get started with Azure Load Balancer by using Azure CLI to create a public load balancer and three virtual machines.

## Prerequisites

- An Azure account with an active subscription. [Create an account for free](https://azure.microsoft.com/free/?WT.mc_id=A261C142F).
- Azure CLI installed locally or Azure Cloud Shell

[!INCLUDE [cloud-shell-try-it.md](../../includes/cloud-shell-try-it.md)] 

If you choose to install and use the CLI locally, this quickstart requires Azure CLI version 2.0.28 or later. To find the version, run `az --version`. If you need to install or upgrade, see [Install the Azure CLI]( /cli/azure/install-azure-cli).

## Create a resource group

An Azure resource group is a logical container into which Azure resources are deployed and managed.

Create a resource group with [az group create](https://docs.microsoft.com/cli/azure/group?view=azure-cli-latest#az-group-create):

* Named **myResourceGroupLB**. 
* In the **eastus** location.

```azurecli-interactive
  az group create \
    --name myResourceGroupLB \
    --location eastus
```

## Create a public IP address

To access your web app on the Internet, you need a public IP address for the load balancer. 

Use [az network public-ip create](https://docs.microsoft.com/cli/azure/network/public-ip?view=azure-cli-latest#az-network-public-ip-create) to:

* Create a standard zone redundant public IP address named **myPublicIP**.
* In **myResourceGroupLB**.

```azurecli-interactive
  az network public-ip create \
    --resource-group myResourceGroupLB \
    --name myPublicIP \
    --sku Standard
```

To create a zonal redundant public IP address in Zone 1:

```azurecli-interactive
  az network public-ip create \
    --resource-group myResourceGroupLB \
    --name myPublicIP \
    --sku Standard \
    --zone 1
```

## Create Azure Load balancer

This section details how you can create and configure the following components of the load balancer:

  * A frontend IP pool that receives the incoming network traffic on the load balancer.
  * A backend IP pool where the frontend pool sends the load balanced network traffic.
  * A health probe that determines health of the backend VM instances.
  * A load balancer rule that defines how traffic is distributed to the VMs.

### Create the load balancer

Create a public load balancer with [az network lb create](https://docs.microsoft.com/cli/azure/network/lb?view=azure-cli-latest#az-network-lb-create):

* Named **myLoadBalancer**.
* A frontend pool named **myFrontEnd**.
* A backend pool named **myBackEndPool**.
* Associated with the public IP address **myPublicIP** that you created in the preceding step. 

```azurecli-interactive
  az network lb create \
    --resource-group myResourceGroupLB \
    --name myLoadBalancer \
    --sku Standard \
    --public-ip-address myPublicIP \
    --frontend-ip-name myFrontEnd \
    --backend-pool-name myBackEndPool       
```

### Create the health probe

A health probe checks all virtual machine instances to ensure they can send network traffic. 

A virtual machine with a failed probe check is removed from the load balancer. The virtual machine is added back into the load balancer when the failure is resolved.

Create a health probe with [az network lb probe create](https://docs.microsoft.com/cli/azure/network/lb/probe?view=azure-cli-latest#az-network-lb-probe-create):

* Monitors the health of the virtual machines.
* Named **myHealthProbe**.
* Protocol **TCP**.
* Monitoring **Port 80**.

```azurecli-interactive
  az network lb probe create \
    --resource-group myResourceGroupLB \
    --lb-name myLoadBalancer \
    --name myHealthProbe \
    --protocol tcp \
    --port 80   
```

### Create the load balancer rule

A load balancer rule defines:

* Frontend IP configuration for the incoming traffic.
* The backend IP pool to receive the traffic.
* The required source and destination port. 

Create a load balancer rule with [az network lb rule create](https://docs.microsoft.com/cli/azure/network/lb/rule?view=azure-cli-latest#az-network-lb-rule-create):

* Named **myHTTPRule**
* Listening on **Port 80** in the frontend pool **myFrontEnd**.
* Sending load-balanced network traffic to the backend address pool **myBackEndPool** using **Port 80**. 
* Using health probe **myHealthProbe**.
* Protocol **TCP**.
* Enable outbound source network address translation (SNAT) using the frontend IP address.

```azurecli-interactive
  az network lb rule create \
    --resource-group myResourceGroupLB \
    --lb-name myLoadBalancer \
    --name myHTTPRule \
    --protocol tcp \
    --frontend-port 80 \
    --backend-port 80 \
    --frontend-ip-name myFrontEnd \
    --backend-pool-name myBackEndPool \
    --probe-name myHealthProbe \
    --disable-outbound-snat false 
```
> [!NOTE]
> The command above enables outbound connectivity for the resources in the backend pool of the load balancer. For advanced outbound connectivity configuration, omit **--disable-outbound-snat false** and refer to **[Outbound connections in Azure](load-balancer-outbound-connections.md)** and **[Configure load balancing and outbound rules in Standard Load Balancer by using Azure CLI](configure-load-balancer-outbound-cli.md)**.


## Configure virtual network

Before you deploy VMs and test your load balancer, create the supporting virtual network resources.

### Create a virtual network

Create a virtual network using [az network vnet create](https://docs.microsoft.com/cli/azure/network/vnet?view=azure-cli-latest#az-network-vnet-createt):

* Named **myVNet**.
* Subnet named **myBackendSubnet**.
* In the **myResourceGroupLB** resource group.
* Location of **eastus**.

```azurecli-interactive
  az network vnet create \
    --resource-group myResourceGroupLB \
    --location eastus \
    --name myVNet \
    --subnet-name myBackendSubnet
```

### Create a network security group

For a standard load balancer, the VMs in the backend address for are required to have network interfaces that belong to a network security group. 

Create a network security group using [az network nsg create](https://docs.microsoft.com/cli/azure/network/nsg?view=azure-cli-latest#az-network-nsg-create):

* Named **myNSG**.
* In resource group **myResourceGroupLB**.

```azurecli-interactive
  az network nsg create \
    --resource-group myResourceGroupLB \
    --name myNSG
```

### Create a network security group rule

Create a network security group rule using [az network nsg rule create](https://docs.microsoft.com/cli/azure/network/nsg/rule?view=azure-cli-latest#az-network-nsg-rule-create):

* Named **myNSGRuleHTTP**.
* In the network security group you created in the previous step, **myNSG**.
* In resource group **myResourceGroupLB**.
* Protocol **TCP**.
* Direction **Inbound**.
* Source **(*)**.
* Destination **(*)**.
* Destination port **Port 80**.
* Access **Allow**.
* Priority **200**.

```azurecli-interactive
  az network nsg rule create \
    --resource-group myResourceGroupLB \
    --nsg-name myNSG \
    --name myNSGRuleHTTP \
    --protocol tcp \
    --direction inbound \
    --source-address-prefix '*' \
    --source-port-range '*' \
    --destination-address-prefix '*' \
    --destination-port-range 80 \
    --access allow \
    --priority 200
```

### Create network interfaces for the virtual machines

Create three network interfaces with [az network nic create](https://docs.microsoft.com/cli/azure/network/nic?view=azure-cli-latest#az-network-nic-create):

#### VM1

* Named **myNicVM1**.
* In resource group **myResourceGroupLB**.
* In virtual network **myVNet**.
* In subnet **myBackendSubnet**.
* In network security group **myNSG**.
* Attached to load balancer **myLoadBalancer** in **myBackEndPool**.

```azurecli-interactive

  az network nic create \
    --resource-group myResourceGroupLB \
    --name myNicVM1 \
    --vnet-name myVNet \
    --subnet myBackEndSubnet \
    --network-security-group myNSG \
    --lb-name myLoadBalancer \
    --lb-address-pools myBackEndPool
```
#### VM2

* Named **myNicVM2**.
* In resource group **myResourceGroupLB**.
* In virtual network **myVNet**.
* In subnet **myBackendSubnet**.
* In network security group **myNSG**.
* Attached to load balancer **myLoadBalancer** in **myBackEndPool**.

```azurecli-interactive
  az network nic create \
    --resource-group myResourceGroupLB \
    --name myNicVM2 \
    --vnet-name myVnet \
    --subnet myBackEndSubnet \
    --network-security-group myNSG \
    --lb-name myLoadBalancer \
    --lb-address-pools myBackEndPool
```
#### VM3

* Named **myNicVM3**.
* In resource group **myResourceGroupLB**.
* In virtual network **myVNet**.
* In subnet **myBackendSubnet**.
* In network security group **myNSG**.
* Attached to load balancer **myLoadBalancer** in **myBackEndPool**.

```azurecli-interactive
  az network nic create \
    --resource-group myResourceGroupLB \
    --name myNicVM3 \
    --vnet-name myVnet \
    --subnet myBackEndSubnet \
    --network-security-group myNSG \
    --lb-name myLoadBalancer \
    --lb-address-pools myBackEndPool
```

## Create backend servers

In this section, you create three virtual machines to be used as backend servers for the load balancer. 

To verify that the load balancer was successfully created, you install NGINX on the virtual machines.

### Create cloud-init configuration file

Use a cloud-init configuration file to install NGINX and run a 'Hello World' Node.js app on a Linux virtual machine. 

In your current shell, create a file named cloud-init.txt. Copy and paste the following configuration into the shell. Ensure that you copy the whole cloud-init file correctly, especially the first line:

```yaml
#cloud-config
package_upgrade: true
packages:
  - nginx
  - nodejs
  - npm
write_files:
  - owner: www-data:www-data
  - path: /etc/nginx/sites-available/default
    content: |
      server {
        listen 80;
        location / {
          proxy_pass http://localhost:3000;
          proxy_http_version 1.1;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection keep-alive;
          proxy_set_header Host $host;
          proxy_cache_bypass $http_upgrade;
        }
      }
  - owner: azureuser:azureuser
  - path: /home/azureuser/myapp/index.js
    content: |
      var express = require('express')
      var app = express()
      var os = require('os');
      app.get('/', function (req, res) {
        res.send('Hello World from host ' + os.hostname() + '!')
      })
      app.listen(3000, function () {
        console.log('Hello world app listening on port 3000!')
      })
runcmd:
  - service nginx restart
  - cd "/home/azureuser/myapp"
  - npm init
  - npm install express -y
  - nodejs index.js
```
### Create virtual machines

Create the virtual machines with [az vm create](https://docs.microsoft.com/cli/azure/vm?view=azure-cli-latest#az-vm-create):

#### VM1
* Named **myVM1**.
* In resource group **myResourceGroupLB**.
* Attached to network interface **myNicVM1**.
* Virtual machine image **UbuntuLTS**.
* Configuration file **cloud-init.txt** you created in step above.
* In **Zone 1**.

```azurecli-interactive
  az vm create \
    --resource-group myResourceGroupLB \
    --name myVM1 \
    --nics myNicVM1 \
    --image UbuntuLTS \
    --generate-ssh-keys \
    --custom-data cloud-init.txt \
    --zone 1 \
    --no-wait
    
```
#### VM2
* Named **myVM2**.
* In resource group **myResourceGroupLB**.
* Attached to network interface **myNicVM2**.
* Virtual machine image **UbuntuLTS**.
* Configuration file **cloud-init.txt** you created in step above.
* In **Zone 2**.

```azurecli-interactive
  az vm create \
    --resource-group myResourceGroupLB \
    --name myVM2 \
    --nics myNicVM2 \
    --image UbuntuLTS \
    --generate-ssh-keys \
    --custom-data cloud-init.txt \
    --zone 2 \
    --no-wait
```

#### VM3
* Named **myVM3**.
* In resource group **myResourceGroupLB**.
* Attached to network interface **myNicVM3**.
* Virtual machine image **UbuntuLTS**.
* Configuration file **cloud-init.txt** you created in step above.
* In **Zone 3**.

```azurecli-interactive
   az vm create \
    --resource-group myResourceGroupLB \
    --name myVM3 \
    --nics myNicVM3 \
    --image UbuntuLTS \
    --generate-ssh-keys \
    --custom-data cloud-init.txt \
    --zone 3 \
    --no-wait
```
It may take a few minutes for the VMs to deploy.

## Test the load balancer

To get the public IP address of the load balancer, use [az network public-ip show](https://docs.microsoft.com/cli/azure/network/public-ip?view=azure-cli-latest#az-network-public-ip-show). 

Copy the public IP address, and then paste it into the address bar of your browser.

```azurecli-interactive
  az network public-ip show \
    --resource-group myResourceGroupLB \
    --name myPublicIP \
    --query [ipAddress] \
    --output tsv
```
:::image type="content" source="./media/load-balancer-standard-public-cli/running-nodejs-app.png" alt-text="Test the load balancer" border="true":::

## Clean up resources

When no longer needed, use the [az group delete](https://docs.microsoft.com/cli/azure/group?view=azure-cli-latest#az-group-delete) command to remove the resource group, load balancer, and all related resources.

```azurecli-interactive
  az group delete 
    --name myResourceGroupLB
```

## Next steps
In this quickstart

* You created a standard public load balancer
* Attached virtual machines. 
* Configured the load balancer traffic rule and health probe.
* Tested the load balancer.

To learn more about Azure Load Balancer, continue to [What is Azure Load Balancer?](load-balancer-overview.md) and [Load Balancer frequently asked questions](load-balancer-faqs.md).

Learn more about [Load Balancer and Availability zones](load-balancer-standard-availability-zones.md).