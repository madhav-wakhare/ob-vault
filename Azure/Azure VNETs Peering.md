
## Virtual Network Peering

Virtual Network Peering simplifies the process of connecting resources across different VNets without the need for additional gateways, hubs, or public internet exposure. There are two primary types of peering:

- **Global VNet Peering:** Connects VNets across different Azure regions.
- **Regional VNet Peering:** Connects VNets within the same Azure region.

Both configurations offer secure, high-speed communication and reduce complexity when managing network architectures.


**Utilization of Microsoft Backbone Network:** When peered, data flows over Microsoft’s secure private backbone network instead of the public internet.

**Seamless Connectivity:** Supports connectivity across VNets in different regions, subscriptions, and even Azure Entra ID tenants, facilitating scalable and resilient network architectures.
![[Pasted image 20260819130043.png]]

How can you prevent traffic from a peered VNet from reaching a specific subnet within your VNet?

Use NSG to block the traffic.