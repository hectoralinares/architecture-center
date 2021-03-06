---
title: Run Linux VMs for an N-tier application on Azure
description: How to run Linux VMs for an N-tier architecture in Microsoft Azure.

author: MikeWasson

ms.date: 11/22/2017

pnp.series.title: Linux VM workloads
pnp.series.next: multi-region-application
pnp.series.prev: multi-vm
---

# Run Linux VMs for an N-tier application

This reference architecture shows a set of proven practices for running Linux virtual machines (VMs) for an N-tier application. [**Deploy this solution**.](#deploy-the-solution)  

![[0]][0]

*Download a [Visio file][visio-download] of this architecture.*

## Architecture

There are many ways to implement an N-tier architecture. The diagram shows a typical 3-tier web application. This architecture builds on [Run load-balanced VMs for scalability and availability][multi-vm]. The web and business tiers use load-balanced VMs.

* **Availability sets.** Create an [availability set][azure-availability-sets] for each tier, and provision at least two VMs in each tier.  This makes the VMs eligible for a higher [service level agreement (SLA)][vm-sla] for VMs. You can deploy a single VM in an availability set, but the single VM will not qualify for an SLA guarantee unless the single VM is using Azure Premium Storage for all OS and data disks.  
* **Subnets.** Create a separate subnet for each tier. Specify the address range and subnet mask using [CIDR] notation. 
* **Load balancers.** Use an [Internet-facing load balancer][load-balancer-external] to distribute incoming Internet traffic to the web tier, and an [internal load balancer][load-balancer-internal] to distribute network traffic from the web tier to the business tier.
* **Azure DNS**. [Azure DNS][azure-dns] is a hosting service for DNS domains, providing name resolution using Microsoft Azure infrastructure. By hosting your domains in Azure, you can manage your DNS records using the same credentials, APIs, tools, and billing as your other Azure services.
* **Jumpbox.** Also called a [bastion host]. A secure VM on the network that administrators use to connect to the other VMs. The jumpbox has an NSG that allows remote traffic only from public IP addresses on a safe list. The NSG should permit secure shell (SSH) traffic.
* **Monitoring.** Monitoring software such as [Nagios], [Zabbix], or [Icinga] can give you insight into response time, VM uptime, and the overall health of your system. Install the monitoring software on a VM that's placed in a separate management subnet.
* **NSGs.** Use [network security groups][nsg] (NSGs) to restrict network traffic within the VNet. For example, in the 3-tier architecture shown here, the database tier does not accept traffic from the web front end, only from the business tier and the management subnet.
* **Apache Cassandra database**. Provides high availability at the data tier, by enabling replication and failover.

## Recommendations

Your requirements might differ from the architecture described here. Use these recommendations as a starting point. 

### VNet / Subnets

When you create the VNet, determine how many IP addresses your resources in each subnet require. Specify a subnet mask and a VNet address range large enough for the required IP addresses using [CIDR] notation. Use an address space that falls within the standard [private IP address blocks][private-ip-space], which are 10.0.0.0/8, 172.16.0.0/12, and 192.168.0.0/16.

Choose an address range that does not overlap with your on-premises network, in case you need to set up a gateway between the VNet and your on-premises network later. Once you create the VNet, you can't change the address range.

Design subnets with functionality and security requirements in mind. All VMs within the same tier or role should go into the same subnet, which can be a security boundary. For more information about designing VNets and subnets, see [Plan and design Azure Virtual Networks][plan-network].

For each subnet, specify the address space for the subnet in CIDR notation. For example, '10.0.0.0/24' creates a range of 256 IP addresses. VMs can use 251 of these; five are reserved. Make sure the address ranges don't overlap across subnets. See the [Virtual Network FAQ][vnet faq].

### Network security groups

Use NSG rules to restrict traffic between tiers. For example, in the 3-tier architecture shown above, the web tier does not communicate directly with the database tier. To enforce this, the database tier should block incoming traffic from the web tier subnet.  

1. Create an NSG and associate it to the database tier subnet.
2. Add a rule that denies all inbound traffic from the VNet. (Use the `VIRTUAL_NETWORK` tag in the rule.) 
3. Add a rule with a higher priority that allows inbound traffic from the business tier subnet. This rule overrides the previous rule, and allows the business tier to talk to the database tier.
4. Add a rule that allows inbound traffic from within the database tier subnet itself. This rule allows communication between VMs in the database tier, which is needed for database replication and failover.
5. Add a rule that allows SSH traffic from the jumpbox subnet. This rule lets administrators connect to the database tier from the jumpbox.
   
   > [!NOTE]
   > An NSG has [default rules][nsg-rules] that allow any inbound traffic from within the VNet. These rules can't be deleted, but you can override them by creating higher-priority rules.
   > 
   > 

### Load balancers

The external load balancer distributes Internet traffic to the web tier. Create a public IP address for this load balancer. See [Creating an Internet-facing load balancer][lb-external-create].

The internal load balancer distributes network traffic from the web tier to the business tier. To give this load balancer a private IP address, create a frontend IP configuration and associate it with the subnet for the business tier. See [Get started creating an Internal load balancer][lb-internal-create].

### Cassandra

We recommend [DataStax Enterprise][datastax] for production use, but these recommendations apply to any Cassandra edition. For more information on running DataStax in Azure, see [DataStax Enterprise Deployment Guide for Azure][cassandra-in-azure]. 

Put the VMs for a Cassandra cluster in an availability set to ensure that the Cassandra replicas are distributed across multiple fault domains and upgrade domains. For more information about fault domains and upgrade domains, see [Manage the availability of virtual machines][azure-availability-sets]. 

Configure three fault domains (the maximum) per availability set and 18 upgrade domains per availability set. This provides the maximum number of upgrade domains that can still be distributed evenly across the fault domains.   

Configure nodes in rack-aware mode. Map fault domains to racks in the `cassandra-rackdc.properties` file.

You don't need a load balancer in front of the cluster. The client connects directly to a node in the cluster.

### Jumpbox

The jumpbox will have minimal performance requirements, so select a small VM size for the jumpbox such as Standard A1. 

Create a [public IP address] for the jumpbox. Place the jumpbox in the same VNet as the other VMs, but in a separate management subnet.

Do not allow SSH access from the public Internet to the VMs that run the application workload. Instead, all SSH access to these VMs must come through the jumpbox. An administrator logs into the jumpbox, and then logs into the other VM from the jumpbox. The jumpbox allows SSH traffic from the Internet, but only from known, safe IP addresses.

To secure the jumpbox, create an NSG and apply it to the jumpbox subnet. Add an NSG rule that allows SSH connections only from a safe set of public IP addresses. The NSG can be attached either to the subnet or to the jumpbox NIC. In this case, we recommend attaching it to the NIC, so SSH traffic is permitted only to the jumpbox, even if you add other VMs to the same subnet.

Configure the NSGs for the other subnets to allow SSH traffic from the management subnet.

## Availability considerations

Put each tier or VM role into a separate availability set. 

At the database tier, having multiple VMs does not automatically translate into a highly available database. For a relational database, you will typically need to use replication and failover to achieve high availability.  

If you need higher availability than the [Azure SLA for VMs][vm-sla] provides, replicate the application across two regions and use Azure Traffic Manager for failover. For more information, see [Run Linux VMs in multiple regions for high availability][multi-dc].  

## Security considerations

Consider adding a network virtual appliance (NVA) to create a DMZ between the public Internet and the Azure virtual network. NVA is a generic term for a virtual appliance that can perform network-related tasks such as firewall, packet inspection, auditing, and custom routing. For more information, see [Implementing a DMZ between Azure and the Internet][dmz].

## Scalability considerations

The load balancers distribute network traffic to the web and business tiers. Scale horizontally by adding new VM instances. Note that you can scale the web and business tiers independently, based on load. To reduce possible complications caused by the need to maintain client affinity, the VMs in the web tier should be stateless. The VMs hosting the business logic should also be stateless.

## Manageability considerations

Simplify management of the entire system by using centralized administration tools such as [Azure Automation][azure-administration], [Microsoft Operations Management Suite][operations-management-suite], [Chef][chef], or [Puppet][puppet]. These tools can consolidate diagnostic and health information captured from multiple VMs to provide an overall view of the system.

## Deploy the solution

A deployment for this reference architecture is available on [GitHub][github-folder]. 

### Prerequisites

Before you can deploy the reference architecture to your own subscription, you must perform the following steps.

1. Clone, fork, or download the zip file for the [AzureCAT reference architectures][ref-arch-repo] GitHub repository.

2. Make sure you have the Azure CLI 2.0 installed on your computer. To install the CLI, follow the instructions in [Install Azure CLI 2.0][azure-cli-2].

3. Install the [Azure building blocks][azbb] npm package.

  ```bash
  npm install -g @mspnp/azure-building-blocks
  ```

4. From a command prompt, bash prompt, or PowerShell prompt, login to your Azure account by using one of the commands below, and follow the prompts.

  ```bash
  az login
  ```

### Deploy the solution using azbb

To deploy the Linux VMs for an N-tier application reference architecture, follow these steps:

1. Navigate to the `virtual-machines\n-tier-linux` folder for the repository you cloned in step 1 of the pre-requisites above.

2. The parameter file specifies a default adminstrator user name and password for each VM in the deployment. You must change these before you deploy the reference architecture. Open the `n-tier-linux.json` file and replace each **adminUsername** and **adminPassword** field with your new settings.   Save the file.

3. Deploy the reference architecture using the **azbb** command line tool as shown below.

  ```bash
  azbb -s <your subscription_id> -g <your resource_group_name> -l <azure region> -p n-tier-linux.json --deploy
  ```

For more information on deploying this sample reference architecture using Azure Building Blocks, visit the [GitHub repository][git].

<!-- links -->
[multi-dc]: multi-region-application.md
[dmz]: ../dmz/secure-vnet-dmz.md
[multi-vm]: ./multi-vm.md
[naming conventions]: /azure/guidance/guidance-naming-conventions
[azbb]: https://github.com/mspnp/template-building-blocks/wiki/Install-Azure-Building-Blocks
[azure-administration]: /azure/automation/automation-intro
[azure-availability-sets]: /azure/virtual-machines/virtual-machines-linux-manage-availability
[azure-cli-2]: https://docs.microsoft.com/cli/azure/install-azure-cli?view=azure-cli-latest
[azure-dns]: /azure/dns/dns-overview
[bastion host]: https://en.wikipedia.org/wiki/Bastion_host
[cassandra-in-azure]: https://docs.datastax.com/en/datastax_enterprise/4.5/datastax_enterprise/install/installAzure.html
[cidr]: https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing
[chef]: https://www.chef.io/solutions/azure/
[datastax]: http://www.datastax.com/products/datastax-enterprise
[git]: https://github.com/mspnp/template-building-blocks
[github-folder]: https://github.com/mspnp/reference-architectures/tree/master/virtual-machines/n-tier-linux
[lb-external-create]: /azure/load-balancer/load-balancer-get-started-internet-portal
[lb-internal-create]: /azure/load-balancer/load-balancer-get-started-ilb-arm-portal
[load-balancer-external]: /azure/load-balancer/load-balancer-internet-overview
[load-balancer-internal]: /azure/load-balancer/load-balancer-internal-overview
[nsg]: /azure/virtual-network/virtual-networks-nsg
[nsg-rules]: /azure/azure-resource-manager/best-practices-resource-manager-security#network-security-groups
[operations-management-suite]: https://www.microsoft.com/server-cloud/operations-management-suite/overview.aspx
[plan-network]: /azure/virtual-network/virtual-network-vnet-plan-design-arm
[private-ip-space]: https://en.wikipedia.org/wiki/Private_network#Private_IPv4_address_spaces
[public IP address]: /azure/virtual-network/virtual-network-ip-addresses-overview-arm
[puppet]: https://puppetlabs.com/blog/managing-azure-virtual-machines-puppet
[ref-arch-repo]: https://github.com/mspnp/reference-architectures
[vm-sla]: https://azure.microsoft.com/support/legal/sla/virtual-machines
[vnet faq]: /azure/virtual-network/virtual-networks-faq
[visio-download]: https://archcenter.azureedge.net/cdn/vm-reference-architectures.vsdx
[Nagios]: https://www.nagios.org/
[Zabbix]: http://www.zabbix.com/
[Icinga]: http://www.icinga.org/
[0]: ./images/n-tier-diagram.png "N-tier architecture using Microsoft Azure"

