<style>
table,
th,
td {
    border: 1px solid black;
    border-collapse: collapse;
}
</style>

### Structure of this Article
- [Introduction](#introduction)
  - [Scope](#scope)
  - [Not in scope](#not-in-scope)

- [Concept: How Azure Meters Network Costs](#concept-how-azure-meters-network-costs)
  - [Service: Bandwidth](#service-bandwidth)
  - [Service: Virtual Network](#service-virtual-network)
  - [Service: Express Route](#service-express-route)
  - [Services: Firewall, Application Gateway, Loadbalancers](#service-vwan-firewall-application-gateway-loadbalancer)

- [Billing Scenarios](#billing-scenarios)
  - [VM  to Internet via Firewall](#vm-to-internet-via-firewall)
  - [VM to Private Endpoint PaaS](#vm-to-private-endpoint-paas)
  - [On-Prem VM to Load Balanced Azure VM (Interregion)](#on-premises-vm-to-load-balanced-azure-vm-interregion)

- [Conclusion](#conclusion)

- [Annex - Further documentation on cost management in azure](#annex)
  - General introduction to cost management in the azure portal
  - Cost management API(#api)
  - Cost Management via Azure CLI(#az-cli)
  - Cost Management via Powershell(#powershell)

## Introduction 

Cost Management in Azure can be a straight forward topic. Assuming there is  a resource with a fixed cost billed for the resource instance and some dynamic cost component which occurs based on the usage.

When it comes to network costs the situation gets confusing. Data transfer over the network is included in nearly all Azure operations. But not all  traffic is billed.

When you skim the documentation for this topic it is mostly pointed out that incoming traffic is free and Azure outgoing traffic costs 0.036 €/GB (at the time of creation) with 100GB free monthly.

Unfortunately it gets way more complex once you look into the details. This blog article shall make some of the details easier to understand by calculating cost examples using sample scenarios. 

### Scope

This article covers regional and multi regional scenarios taking place in one Zone (Zone 1).
Also only data transfer costs are analyzed. Up front costs for resources are neglected due to clarity considerations.

### Not in scope

When it comes to intercontinental traffic the cost structure is even more complex. This is not a part of this article though

## Concept: How does Azure meter network incurred costs

There are two types of network related costs. <a href="https://azure.microsoft.com/en-us/pricing/details/virtual-network/">vNet Peering</a> / <a href="https://azure.microsoft.com/en-us/pricing/details/bandwidth/?cdn=disable">Bandwidth charges</a> and Data processed charges.


Network transfer costs arise whereever your traffic traverses parts of the azure infrastructure. So one request can generate network-based costs in various places. 

In the subsequent chapters scenarios are described that shall help to identify when
which type of network related costs are triggered.

Microsoft published a network pricing   overview which is based on a virtualWan enabled network  <a href="https://learn.microsoft.com/en-us/azure/virtual-wan/pricing-concepts">here</a>.



### Service Bandwidth

Bandwidth describes the traffic leaving the regional Azure premises or the traffic routed
to the Internet.

The service name "Bandwidth" consists of following meter subcategories

 * Outbound Data Transfer (Internet egress)
   - "Rtn Preference: MGN"  -> uses Microsoft global network - billed per GB
   - "Rtn Preference: ISP"  -> uses your ISPs breakout - billed per GB, slightly cheaper

 * Bandwidth Inter-Region , Data transfers between different Azure regions ( traffic that stays in the azure premises)

 The charges are then billed under one of the following meters 

  - Standard Data Transfer Out 
  - Intra Continent Data Transfer Out
  - Inter Continent Data Transfer Out - NAM or EU To Any   
  - Standard Data Transfer In (usually free)
  

### Service Virtual Network

⚠️ Do not confuse the service "Virtual Network" with the resource Virtual Network. The costs for the service virtual network are assigned to each resource that generate them. (VMs, VM Scale Sets, AKS)

Virtual Networks are free so you will never find them in your cost analysis. All network related costs are assigned to the resources and services that generate them.

Service: **Virtual Network**
Meter Subcategory: **Virtual Network Peering**

In this subcategory you will find the traffic costs generated by VMs accessing Services though the vNet

* Intra-Region Ingress
* Intra-Region Egress


Meter Subcategory **Virtual Network-Private Link**

All costs generated by private endpoints will be gathered in this subcategory

* Standard Private Endpoint
* Standard Data Processed Egress
* Standard Data Processed Ingress

Note: A private endpoint allows a Platform-as-a-Service (PaaS) resource to be accessed from within a virtual network, instead of the public internet. 
<a href="https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-overview">Further Reading on private endpoint</a>

### Service Express Route

In this article we assume a metered Express Route of sku standard, which costs are described by the following metres:

* Standard Metered Data  - Fixed costs for the Express Route provisioning / month
* Metered Data - Data Transfer Out - Billed / GB
* Data Transfer In - Usually Free

### Service vWan, Firewall, Application Gateway, Loadbalancer

These are all network related ressources which costs are 
connected to the amount of traffic they process. No matter if its ingress or egress. 

## Billing Scenarios

### VM to Internet via Firewall

A VM accesses a service that is provided by the internet.

# ![scenario1](scenario1-network-costs.drawio.png)

(1)
A VM generates Traffic (In this example 1 TB). The traffic is routed via the virtual network peering
to the vNet in which the firewall resides.

(Important) In this scenario the traffic is routed via the existing vNet Peering (the red line). It is not routed via the vWWan Hub.

Then it enters the vNet (2) and the data is processed by the firewall (3).

(4) The traffic egresses to the internet via the firewall by traversing the Microsoft Global Network. (Routing Preference: MGN) 

There are 2 routing options for internet destined traffic:

MGN - Microsoft Global Network - Traffic will use the microsoft backbone until it exits over the microsoft network edge pop closest to the destination. (more costly)

ISP - The traffic will leave microsoft premises by the next pop and travels to the destination via the public internet.
(a bit less costly)

https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/routing-preference-overview

Cost breakdown for this scenario #1

|Waypoint | Amount billed| Cost descrtiption
------------|-----------|-------|
1.| 18,12 € | VNET Peering Egress
2.| 18,012 € | VNET Peering Ingress
3.| 14,40 € | Firewall Data Processed
4.| 69,28 € | Bandwidth Internet Egress "Rtn Preference: MGN"
-- | **120,02** | Total Cost for 1TB (1024 GB)

💡 Find the scenario in the azure cost calculator here:
https://azure.com/e/1b9ed1a4c96e42289291a7294144b27a

💡 Summary: Traffic leaving the region or going to the internet is billed per GB. Intra-region traffic within Azure is also billed. Bandwidth costs depend on routing preference and target location.

### VM to Private Endpoint PaaS

A VM accesses a private endpoint enabled storage account (can be any private endpoint enabled service like data base, key vault, cache, etc.).

# ![scenario2](scenario2-network-costs.drawio.png)

(1)
The traffic traverses the vNet-Link peering to the vHub.
(2)
The vHub processes (routes) the traffic.
(3) 
The traffic leaves the vHub to the Ressource via the vNet  -Link Peering.
(4) 
The requests then reach the network interface of the private Link enabled PaaS.

Cost Breakdown for Scenario #2

|Waypoint | Amount billed| Cost description
------------|-----------|-------|
1.| 18,12 € | vNet Link Peering Egress 
2.| 18,12 € | vHub Processing
3.| 18,12 € | vNet Link Peering Ingress
4.| 9,06 € | private Endpoint processing costs (inbound) 
-- | **63,42** | Total Cost for 1TB (1024 GB)

💡 Find the scenario in the azure cost calculator here:
https://azure.com/e/320ede77328a46f0a0c5c3ac19c1a350

💡 Summary: Traffic going to private endpoint enabled services generates additional processing costs. It is billed per GB.

### On-Prem VM to Load Balanced Azure VM (Interregion)

A VM on premises accesses a service that  is provided by VMs that are behind a  loadbalancer in azure. The on premises DC is connected with an express route.

# ![scenario3](scenario3-network-costs.drawio.png)


(1) Traffic entering Azure through the express Route is not billed.

Note: It is likely though that the ISP provising the link for the ExpressRoute will charge traffic fees.

(2)
The traffic is then processed (routed) by the source regional vHub which incurs costs.

(3) The traffic is routed to the vWan Hub in the destination region through the microsoft 
backbone which is billed accordingly.

(4)
The traffic is then processed (routed) by the source regional vHub which incurs costs.

(5)
The ingressing traffic is billed again in the destination vNet in which the lb and the vm reside.

(6) finally the standard loadbalancer will process the traffic according to the configured
loadbalancing rules which generates costs.


|Waypoint | Amount billed| Cost description
------------|-----------|-------|
1.| 0,00 € | ingress trafic enetering Azure free, might be subject to Internet provider billing
2.| 18,12 € | vHub Processing (EU Region-1)
3.| 18,03 € | Bandwidth Inter Region, Intra Continent Data Transfer Out
4.| 18,12 € | vHub Processing (EU Region-2)
5.| 18,12 € | vNet Peering Ingress
6.|  4,53 € | LB, SKU Standard Data Processed
-- | **76,92** | Total Cost for 1 TB 

💡 Find the scenario in the azure cost calculator here:
https://azure.com/e/dd02313770d444eda1722aa3b6662d6f

💡 Summary: Traffic between two regions inside a virtual wan casuses bandwidth costs. Also processing costs for the virtual wan hubs in the regions are billed.


## Conclusion

🔄 Analyze if  resources that provide a service (e.g. frontend application gateway, vm, database) can be placed in one vNet. There will be significantly less network induced costs than when they are all distributed in different vNets or even regions.

🔄 The management of a meshed hub and spoke network architecture can be complex and costly. A virtualWan
offers a centrally managed full mesh hub and spoke architecture out of the box. These two costs must be compared.

-> 🔄 It makes sense to evaluate different network designs with a focus on costs. 

Minimize Data Transfer Costs:

✅ Keep resources within the same region to avoid  supra-regional or even intercontinental data transfer fees.

✅ Monitor network traffic and the processing load of network related ressources. Use this data to adjust sizing of your core network components.

⚠️  Do not overprovision infrastructure components as virtual wan hubs, express routes or application gateways.

💸 When Load testing your applications do  consider to not route the traffic through your azure firewall each time. As it might be feasible to test the interplay of all components once it is
most likely not required to route the load testing traffic through the firewall each time as it can generate significant costs.

## Annex 
https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/overview-cost-management

Azure CLI billing / consumption & reservations
https://learn.microsoft.com/en-us/cli/azure/service-page/cost%20management?view=azure-cli-latest

Powershell costmanagement, billing and reservations module

https://learn.microsoft.com/en-us/powershell/module/az.billing/
https://learn.microsoft.com/en-us/powershell/module/az.costmanagement/
https://learn.microsoft.com/en-us/powershell/module/az.reservations/

Azure REST APIs for Cost Management, Billing and Consumption

https://learn.microsoft.com/en-us/rest/api/cost-management/
https://learn.microsoft.com/en-us/rest/api/billing/
https://learn.microsoft.com/en-us/rest/api/consumption/