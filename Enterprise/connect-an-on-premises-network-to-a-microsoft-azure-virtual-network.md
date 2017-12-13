---
title: "Connect an on-premises network to a Microsoft Azure virtual network"
ms.author: josephd
author: JoeDavies-MSFT
manager: laurawi
ms.date: 12/15/2017
ms.audience: ITPro
ms.topic: article
ms.service: o365-solutions
localization_priority: Normal
ms.collection:
- Ent_O365
- Ent_O365_Hybrid
- Ent_O365_Hybrid_Top
- Ent_O365_Top
ms.custom:
- DecEntMigration
- Strat_O365_Enterprise
- Ent_Solutions
ms.assetid: 81190961-5454-4a5c-8b0e-6ae75b9fb035
description: "Summary: Learn how to configure a cross-premises Azure virtual network for Office server workloads."
---

# Connect an on-premises network to a Microsoft Azure virtual network

 **Summary:** Learn how to configure a cross-premises Azure virtual network for Office server workloads.
  
A cross-premises Azure virtual network is connected to your on-premises network, extending your network to include subnets and virtual machines hosted in Azure infrastructure services. This connection allows computers on your on-premises network to directly access virtual machines in Azure and vice versa. For example, a DirSync server running on an Azure virtual machine needs to query your on-premises domain controllers for changes to accounts and synchronize those changes with your Office 365 subscription. This article shows you how to set up a cross-premises Azure virtual network that is ready to host Azure virtual machines.
  
In this article:
  
- [Overview](connect-an-on-premises-network-to-a-microsoft-azure-virtual-network.md#Overview)
    
- [Plan your Azure virtual network](connect-an-on-premises-network-to-a-microsoft-azure-virtual-network.md#PlanningVirtual)
    
- [Deployment roadmap](connect-an-on-premises-network-to-a-microsoft-azure-virtual-network.md#DeploymentRoadmap)
    
## Overview
<a name="Overview"> </a>

Your virtual machines in Azure don't have to be isolated from your on-premises environment. To connect Azure virtual machines to your on-premises network resources, you must configure a cross-premises Azure virtual network. The following diagram shows the required components to deploy a cross-premises Azure virtual network with a virtual machine in Azure.
  
![On-premises network connected to Microsoft Azure by a site-to-site VPN connection](images/CP_ConnectOnPremisesNetworkToAzureVPN.png)
  
In the diagram, there are two networks connected by a site-to-site virtual private network (VPN) connection: the on-premises network and the Azure virtual network. The site-to-site VPN connection is terminated by a VPN device on the on-premises network and an Azure VPN gateway on the Azure virtual network. The Azure virtual network has virtual machines. Network traffic originating from virtual machines on the Azure virtual network gets forwarded to the VPN gateway, which then forwards the traffic across the site-to-site VPN connection to the VPN device on the on-premises network. The routing infrastructure of the on-premises network then forwards the traffic to its destination.
  
To set up the VPN connection between your Azure virtual network and your on-premises network, do the following steps: 
  
1. **On-premises:** Define and create an on-premises network route for the address space of the Azure virtual network that points to your on-premises VPN device.
    
2. **Microsoft Azure:** Create an Azure virtual network with a site-to-site VPN connection. This article does not describe the use of [ExpressRoute](https://azure.microsoft.com/services/expressroute/).
    
3. **On premises:** Configure your on-premises hardware or software VPN device to terminate the VPN connection, which uses Internet Protocol security (IPsec).
    
After you establish the site-to-site VPN connection, you add Azure virtual machines to the subnets of the virtual network.
  
## Plan your Azure virtual network
<a name="PlanningVirtual"> </a>

### Prerequisites
<a name="Prerequisites"> </a>

- An Azure subscription. For information about Azure subscriptions, go to the [Microsoft Azure subscription page](https://azure.microsoft.com/pricing/purchase-options/).
    
- An available private IPv4 address space to assign to the virtual network and its subnets, with sufficient room for growth to accommodate the number of virtual machines needed now and in the future.
    
- An available VPN device in your on-premises network to terminate the site-to-site VPN connection that supports the requirements for IPsec. For more information, see [About VPN devices for site-to-site virtual network connections](https://go.microsoft.com/fwlink/p/?LinkId=393093).
    
- Changes to your routing infrastructure so that traffic routed to the address space of the Azure virtual network gets forwarded to the VPN device that hosts the site-to-site VPN connection.
    
- A web proxy that gives computers that are connected to the on-premises network and the Azure virtual network access to the Internet.
    
### Solution architecture design assumptions
<a name="DesignAssumptions"> </a>

The following list represents the design choices that have been made for this solution architecture. 
  
- This solution uses a single Azure virtual network with a site-to-site VPN connection. The Azure virtual network hosts a single subnet that can contain multiple virtual machines. 
    
- You can use the Routing and Remote Access Service (RRAS) in Windows Server 2016 or Windows Server 2012 to establish an IPsec site-to-site VPN connection between the on-premises network and the Azure virtual network. You can also use other options, such as Cisco or Juniper Networks VPN devices.
    
- The on-premises network might still have network services like Windows Server Active Directory (AD), Domain Name System (DNS), and proxy servers. Depending on your requirements, it might be beneficial to place some of these network resources in the Azure virtual network.
    
For an existing Azure virtual network with one or more subnets, determine whether there is remaining address space for an additional subnet to host your needed virtual machines, based on your requirements. If you don't have remaining address space for an additional subnet, create an additional virtual network that has its own site-to-site VPN connection.
  
### Plan the routing infrastructure changes for the Azure virtual network
<a name="routing"> </a>

You must configure your on-premises routing infrastructure to forward traffic destined for the address space of the Azure virtual network to the on-premises VPN device that is hosting the site-to-site VPN connection.
  
The exact method of updating your routing infrastructure depends on how you manage routing information, which can be:
  
- Routing table updates based on manual configuration.
    
- Routing table updates based on routing protocols, such as Routing Information Protocol (RIP) or Open Shortest Path First (OSPF).
    
Consult with your routing specialist to make sure that traffic destined for the Azure virtual network is forwarded to the on-premises VPN device.
  
### Plan for firewall rules for traffic to and from the on-premises VPN device
<a name="firewall"> </a>

If your VPN device is on a perimeter network that has a firewall between the perimeter network and the Internet, you might have to configure the firewall for the following rules to allow the site-to-site VPN connection.
  
- Traffic to the VPN device (incoming from the Internet):
    
  - Destination IP address of the VPN device and IP protocol 50
    
  - Destination IP address of the VPN device and UDP destination port 500
    
  - Destination IP address of the VPN device and UDP destination port 4500
    
- Traffic from the VPN device (outgoing to the Internet):
    
  - Source IP address of the VPN device and IP protocol 50
    
  - Source IP address of the VPN device and UDP source port 500
    
  - Source IP address of the VPN device and UDP source port 4500
    
### Plan for the private IP address space of the Azure virtual network
<a name="IPAddresses"> </a>

The private IP address space of the Azure virtual network must be able to accommodate addresses used by Azure to host the virtual network and with at least one subnet that has enough addresses for your Azure virtual machines.
  
To determine the number of addresses needed for the subnet, count the number of virtual machines that you need now, estimate for future growth, and then use the following table to determine the size of the subnet.
  
|**Number of virtual machines needed**|**Number of host bits needed**|**Size of the subnet**|
|:-----|:-----|:-----|
|1-3  <br/> |3  <br/> |/29  <br/> |
|4-11  <br/> |4  <br/> |/28  <br/> |
|12-27  <br/> |5  <br/> |/27  <br/> |
|28-59  <br/> |6  <br/> |/26  <br/> |
|60-123  <br/> |7  <br/> |/25  <br/> |
   
### Planning worksheet for configuring your Azure virtual network
<a name="worksheet"> </a>

Before you create an Azure virtual network to host virtual machines, you must determine the settings needed in the following tables.
  
For the settings of the virtual network, fill in Table V.
  
 **Table V: Cross-premises virtual network configuration**
  
|**Item**|**Configuration element**|**Description**|**Value**|
|:-----|:-----|:-----|:-----|
|1.  <br/> |Virtual network name  <br/> |A name to assign to the Azure virtual network (example DirSyncNet).  <br/> |__________________  <br/> |
|2.  <br/> |Virtual network location  <br/> |The Azure datacenter that will contain the virtual network (such as West US).  <br/> |__________________  <br/> |
|3.  <br/> |VPN device IP address  <br/> |The public IPv4 address of your VPN device's interface on the Internet. Work with your IT department to determine this address.  <br/> |__________________  <br/> |
|4.  <br/> |Virtual network address space  <br/> |The address space (defined in a single private address prefix) for the virtual network. Work with your IT department to determine this address space. The address space should be in Classless Interdomain Routing (CIDR) format, also known as network prefix format. An example is 10.24.64.0/20.  <br/> |__________________  <br/> |
|5.  <br/> |IPsec shared key  <br/> |A 32-character random, alphanumeric string that will be used to authenticate both sides of the site-to-site VPN connection. Work with your IT or security department to determine this key value and then store it in a secure location. Alternately, see [Create a random string for an IPsec preshared key](https://social.technet.microsoft.com/wiki/contents/articles/32330.create-a-random-string-for-an-ipsec-preshared-key.aspx).  <br/> |__________________  <br/> |
   
Fill in Table S for the subnets of this solution.
  
- For the first subnet, determine a 28-bit address space (with a /28 prefix length) for the Azure gateway subnet. See [Calculating the gateway subnet address space for Azure virtual networks](https://blogs.technet.microsoft.com/solutions_advisory_board/2016/12/01/calculating-the-gateway-subnet-address-space-for-azure-virtual-networks/) for information about how to determine this address space.
    
- For the second subnet, specify a friendly name, a single IP address space based on the virtual network address space, and a descriptive purpose.
    
Work with your IT department to determine these address spaces from the virtual network address space. Both address spaces should be in CIDR format.
  
 **Table S: Subnets in the virtual network**
  
|**Item**|**Subnet name**|**Subnet address space**|**Purpose**|
|:-----|:-----|:-----|:-----|
|1.  <br/> |GatewaySubnet  <br/> |_____________________________  <br/> |The subnet used by the Azure gateway.  <br/> |
|2.  <br/> |_____________________________  <br/> |_____________________________  <br/> |_____________________________  <br/> |
   
For the on-premises DNS servers that you want the virtual machines in the virtual network to use, fill in Table D. Give each DNS server a friendly name and a single IP address. This friendly name does not need to match the host name or computer name of the DNS server. Note that two blank entries are listed, but you can add more. Work with your IT department to determine this list.
  
 **Table D: On-premises DNS servers**
  
|**Item**|**DNS server friendly name**|**DNS server IP address**|
|:-----|:-----|:-----|
|1.  <br/> |_____________________________  <br/> |_____________________________  <br/> |
|2.  <br/> |_____________________________  <br/> |_____________________________  <br/> |
   
To route packets from the Azure virtual network to your organization network across the site-to-site VPN connection, you must configure the virtual network with a local network. This local network contains a list of the address spaces (in CIDR format) for all of the locations on your organization's on-premises network that the virtual machines in the virtual network must reach. This can be all of the locations on the on-premises network or a subset. The list of address spaces that define your local network must be unique and must not overlap with the address spaces used for this virtual network or your other cross-premises virtual networks.
  
For the set of local network address spaces, fill in Table L. Note that three blank entries are listed but you will typically need more. Work with your IT department to determine this list.
  
 **Table L: Address prefixes for the local network**
  
|**Item**|**Local network address space**|
|:-----|:-----|
|1.  <br/> |_____________________________  <br/> |
|2.  <br/> |_____________________________  <br/> |
|3.  <br/> |_____________________________  <br/> |
   
## Deployment roadmap
<a name="DeploymentRoadmap"> </a>

Creating the cross-premises virtual network and adding virtual machines in Azure consists of three phases:
  
- Phase 1: Prepare your on-premises network.
    
- Phase 2: Create the cross-premises virtual network in Azure.
    
- Phase 3 (Optional): Add virtual machines.
    
### Phase 1: Prepare your on-premises network
<a name="Phase1"> </a>

You must configure your on-premises network with a route that points to and ultimately delivers traffic destined for the address space of the virtual network to the router on the edge of the on-premises network. Consult with your network administrator to determine how to add the route to the routing infrastructure of your on-premises network.
  
Here is your resulting configuration.
  
![The on-premises network must have a route for the virtual network's address space that points toward the VPN device.](images/90bab36b-cb60-4ea5-81d5-4737b696d41c.png)
  
### Phase 2: Create the cross-premises virtual network in Azure
<a name="Phase2"> </a>

First, open an Azure PowerShell prompt. If you have not installed Azure PowerShell, see [Get started with Azure PowerShell cmdlets](https://docs.microsoft.com/powershell/azureps-cmdlets-docs/).
  
> [!NOTE]
> These commands are for Azure PowerShell 1.0 and above. For a text file that contains all the PowerShell commands in this article, click [here](https://gallery.technet.microsoft.com/scriptcenter/PowerShell-commands-for-5c5a7c19). 
  
Next, login to your Azure account with this command.
  
```
Login-AzureRMAccount
```

Get your subscription name using the following command.
  
```
Get-AzureRMSubscription | Sort SubscriptionName | Select SubscriptionName
```

Set your Azure subscription with these commands. Replace everything within the quotes, including the < and > characters, with the correct subscription name.
  
```
$subscrName="<subscription name>"
Select-AzureRMSubscription -SubscriptionName $subscrName -Current
```

Next, create a new resource group for your virtual network. To determine a unique resource group name, use this command to list your existing resource groups.
  
```
Get-AzureRMResourceGroup | Sort ResourceGroupName | Select ResourceGroupName
```

Create your new resource group with these commands.
  
```
$rgName="<resource group name>"
$locName="<Table V - Item 2 - Value column>"
New-AzureRMResourceGroup -Name $rgName -Location $locName

```

Resource Manager-based virtual machines require a Resource Manager-based storage account. You must pick a globally unique name for your storage account that contains only lowercase letters and numbers. You can use this command to list the existing storage accounts.
  
```
Get-AzureRMStorageAccount | Sort Name | Select Name
```

Use this command to test whether a proposed storage account name is unique.
  
```
Get-AzureRmStorageAccountNameAvailability "<proposed name>"
```

To create a new storage account, run these commands.
  
```
$rgName="<your new resource group name>"
$locName="<the location of your new resource group>"
$saName="<unique storage account name>"
New-AzureRMStorageAccount -Name $saName -ResourceGroupName $rgName -Type Standard_LRS -Location $locName
```

Next, you create the Azure virtual network.
  
```
# Fill in the variables from previous values and from Tables V, S, and D
$rgName="<name of your new resource group>"
$locName="<Azure location of your new resource group>"
$vnetName="<Table V - Item 1 - Value column>"
$vnetAddrPrefix="<Table V - Item 4 - Value column>"
$gwSubnetPrefix="<Table S - Item 1 - Subnet address space column>"
$SubnetName="<Table S - Item 2 - Subnet name column>"
$SubnetPrefix="<Table S - Item 2 - Subnet address space column>"
$dnsServers=@( "<Table D - Item 1 - DNS server IP address column>", "<Table D - Item 2 - DNS server IP address column>" )

# Get the shortened version of the location
$rg=Get-AzureRmResourceGroup -Name $rgName
$locShortName=$rg.Location

# Create the Azure virtual network and a network security group that allows incoming remote desktop connections to the subnet that is hosting virtual machines
$gatewaySubnet=New-AzureRmVirtualNetworkSubnetConfig -Name "GatewaySubnet" -AddressPrefix $gwSubnetPrefix
$vmSubnet=New-AzureRMVirtualNetworkSubnetConfig -Name $SubnetName -AddressPrefix $SubnetPrefix
New-AzureRMVirtualNetwork -Name $vnetName -ResourceGroupName $rgName -Location $locName -AddressPrefix $vnetAddrPrefix -Subnet $gatewaySubnet,$vmSubnet -DNSServer $dnsServers
$rule1=New-AzureRMNetworkSecurityRuleConfig -Name "RDPTraffic" -Description "Allow RDP to all VMs on the subnet" -Access Allow -Protocol Tcp -Direction Inbound -Priority 100 -SourceAddressPrefix Internet -SourcePortRange * -DestinationAddressPrefix * -DestinationPortRange 3389
New-AzureRMNetworkSecurityGroup -Name $SubnetName -ResourceGroupName $rgName -Location $locShortName -SecurityRules $rule1
$vnet=Get-AzureRMVirtualNetwork -ResourceGroupName $rgName -Name $vnetName
$nsg=Get-AzureRMNetworkSecurityGroup -Name $SubnetName -ResourceGroupName $rgName
Set-AzureRMVirtualNetworkSubnetConfig -VirtualNetwork $vnet -Name $SubnetName -AddressPrefix $SubnetPrefix -NetworkSecurityGroup $nsg
```

Here is your resulting configuration.
  
![The virtual network is not yet connected to the on-premises network.](images/54a37782-a6cc-4d48-b38d-73e128b44a82.png)
  
Next, use these commands to create the gateways for the site-to-site VPN connection.
  
```
# Fill in the variables from previous values and from Tables V and L
$vnetName="<Table V - Item 1 - Value column>"
$localGatewayIP="<Table V - Item 3 - Value column>"
$localNetworkPrefix=@( <comma-separated, double-quote enclosed list of the local network address prefixes from Table L, example: "10.1.0.0/24", "10.2.0.0/24"> )
$vnetConnectionKey="<Table V - Item 5 - Value column>"
$vnet=Get-AzureRMVirtualNetwork -Name $vnetName -ResourceGroupName $rgName
# Attach a virtual network gateway to a public IP address and the gateway subnet
$publicGatewayVipName="PublicIPAddress"
$vnetGatewayIpConfigName="PublicIPConfig"
New-AzureRMPublicIpAddress -Name $vnetGatewayIpConfigName -ResourceGroupName $rgName -Location $locName -AllocationMethod Dynamic
$publicGatewayVip=Get-AzureRMPublicIpAddress -Name $vnetGatewayIpConfigName -ResourceGroupName $rgName
$vnetGatewayIpConfig=New-AzureRMVirtualNetworkGatewayIpConfig -Name $vnetGatewayIpConfigName -PublicIpAddressId $publicGatewayVip.Id -SubnetId $vnet.Subnets[0].Id
# Create the Azure gateway
$vnetGatewayName="AzureGateway"
$vnetGateway=New-AzureRMVirtualNetworkGateway -Name $vnetGatewayName -ResourceGroupName $rgName -Location $locName -GatewayType Vpn -VpnType RouteBased -IpConfigurations $vnetGatewayIpConfig
# Create the gateway for the local network
$localGatewayName="LocalNetGateway"
$localGateway=New-AzureRMLocalNetworkGateway -Name $localGatewayName -ResourceGroupName $rgName -Location $locName -GatewayIpAddress $localGatewayIP -AddressPrefix $localNetworkPrefix
# Create the Azure virtual network VPN connection
$vnetConnectionName="S2SConnection"
$vnetConnection=New-AzureRMVirtualNetworkGatewayConnection -Name $vnetConnectionName -ResourceGroupName $rgName -Location $locName -ConnectionType IPsec -SharedKey $vnetConnectionKey -VirtualNetworkGateway1 $vnetGateway -LocalNetworkGateway2 $localGateway
```

Here is your resulting configuration.
  
![The virtual network now has a gateway.](images/82dd66b2-a4b7-48f6-a89b-cfdd94630980.png)
  
Next, configure your on-premises VPN device to connect to the Azure VPN gateway. For more information, see [About VPN Devices for site-to-site Azure Virtual Network connections](https://docs.microsoft.com/azure/vpn-gateway/vpn-gateway-about-vpn-devices).
  
To configure your VPN device, you will need the following:
  
- The public IPv4 address of the Azure VPN gateway for your virtual network. Use the **Get-AzureRMPublicIpAddress -Name $vnetGatewayIpConfigName -ResourceGroupName $rgName** command to display this address.
    
- The IPsec pre-shared key for the site-to-site VPN connection (Table V- Item 5 - Value column).
    
Here is your resulting configuration.
  
![The virtual network is now connected to the on-premises network.](images/6379c423-4f22-4453-941b-7ff32484a0a5.png)
  
### Phase 3 (Optional): Add virtual machines
<a name="Phase2"> </a>

Create the virtual machines you need in Azure. For more information, see [Create your first Windows virtual machine in the Azure portal](https://go.microsoft.com/fwlink/p/?LinkId=393098).
  
Use the following settings:
  
- On the **Basics** pane, select the same subscription and resource group as your virtual network. Record the user name and password in a secure location. You will need these later to sign in to the virtual machine.
    
- On the **Size** pane, choose the appropriate size.
    
- On the **Settings** pane, in the **Storage** section, select the **Standard** storage type and the storage account set up with your virtual network. In the **Network** section, select the name of your virtual network and the subnet for hosting virtual machines (not the GatewaySubnet). Leave all other settings at their default values.
    
Verify that your virtual machine is using DNS correctly by checking your internal DNS to ensure that Address (A) records were added for you new virtual machine. To access the Internet, your Azure virtual machines must be configured to use your on-premises network's proxy server. Contact your network administrator for additional configuration steps to perform on the server.
  
Here is your resulting configuration.
  
![The virtual network now hosts virtual machines that are accessible from the on-premises network.](images/86ab63a6-bfae-4f75-8470-bd40dff123ac.png)
  
## See Also

<a name="DeploymentRoadmap"> </a>

[Cloud adoption and hybrid solutions](cloud-adoption-and-hybrid-solutions.md)
  
[Deploy Office 365 Directory Synchronization (DirSync) in Microsoft Azure](deploy-office-365-directory-synchronization-dirsync-in-microsoft-azure.md)

[How to create the virtual machine](https://go.microsoft.com/fwlink/p/?LinkId=393098)
  
[About VPN Devices for site-to-site Azure Virtual Network connections](https://azure.microsoft.com/documentation/articles/vpn-gateway-about-vpn-devices/)
  
[How to install and configure Azure PowerShell](https://azure.microsoft.com/documentation/articles/powershell-install-configure/#how-to-install-azure-powershell)


