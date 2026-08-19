
## Azure-to-Azure connectivity

Two common ways to connect separate VNets (for example, VNet-A and VNet-B) are VNet Peering and VPN Gateway (VNet-to-VNet).

- By default VNets are isolated. To enable communication you must explicitly connect them.
- Consider region, throughput needs, whether traffic must traverse the public internet, and whether you require encryption in transit for your selection.

Comparison of the two approaches:

| Option                     | How it works                                                                                 | Best for                                                                          | Pros                                                                                         | Cons                                                                                                          |
| -------------------------- | -------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| VNet Peering               | Direct connection over the Azure backbone between two VNets (same region or global)          | Low-latency, high-throughput intra-Azure connectivity                             | Low latency, high bandwidth, simple, no gateway cost                                         | Requires non-overlapping address spaces; traffic is not encrypted over the internet (stays on Azure backbone) |
| VPN Gateway (VNet-to-VNet) | GatewaySubnet with Azure VPN Gateway establishes IPsec/IKE encrypted tunnel between gateways | Encrypted connectivity across regions or when you need site-to-site-style tunnels | Encrypts traffic end-to-end across public internet; supports cross-region gateway-to-gateway | Higher latency and cost vs peering; requires GatewaySubnet and gateway SKUs                                   |
![[Pasted image 20260819113155.png]]
### Gateway subnet :
Gateway subnet is a special subnet that we create for for deploying something called as VPN Gateway.

With help of VPN Gateway, we will be able to establish a VNET-to-VNET connection. 
So VNet A machine will be able to send packets over the Vnet-to-Vnet connection to the other Vnet B.

**The same VPN Gateway can be used for on-premises connectivity as well.**

### VNet Peering :
Alternate method for Azure-to-Azure Connectivity(Not for on-prem to Vnet connectivity). This is more straight forward approach that connects VNETs without the need for a VPN Gateway.

Vnet peering allows us to for low-latency and high-bandwidth connections between resources in different Vnets.

---

## Azure to on-premises Connectivity

![[Pasted image 20260819123137.png]]

### Site-to-Site VPN :

**Site-to-Site connection** means I have a site in Azure, I have a site on-premises and We want to connect it via **VPN Gateway**.

*In VPN Gateway we use public internet to send traffic to the on-premises site, but this will be over an encrypted connection in a VPN Tunnel.*

### Express Routes :

But if we want a private connection in our on-premises environment, that when we deploy **ExpressRoute**.

Express Route will be more expensive, but will be a private connection coming from Azure Datacenter to your on-premises datacenter. (Dedicated line for you from on-premises to Azure Datacenter through ISP and partners.)


**Point-to-Site Connection** :

There is one more option called **Point-to-Site** Connection. The Point to site connection is also established with help of VPN Gateway.
Individual client devices connect to Azure via VPN Gateway (SSTP/OpenVPN/IKEv2) without needing that person to be present on on-premises site. (Remote workers, developers, ad-hoc access)

---
Key operational notes:

- GatewaySubnet: When deploying any Azure VPN Gateway (VNet-to-VNet, Site-to-Site, or Point-to-Site), create a subnet named exactly `GatewaySubnet`. Size it according to the gateway SKU you choose. See Azure VPN Gateway guidance: [https://learn.microsoft.com/azure/vpn-gateway/vpn-gateway-about-gateway-subnet](https://learn.microsoft.com/azure/vpn-gateway/vpn-gateway-about-gateway-subnet).
- VNet Peering: Uses Azure’s private backbone, requires non-overlapping address spaces, and provides low-latency/high-throughput connectivity without explicit gateways.
- ExpressRoute: Provides a private circuit via connectivity providers—better throughput and latency than internet VPNs but requires provider setup and higher cost.

Guidance:

- Use Site-to-Site VPN when you need encrypted connectivity quickly or cost-effectively.
- Use ExpressRoute when you require predictable performance, large bandwidth, and a private, SLA-backed link.
- Use P2S for remote-user access to resources in Azure without exposing your on-premises network.

---
Important caveats:

- VNet Peering does not encrypt traffic across public internet because peering traffic stays on Azure’s backbone; if you require encryption end-to-end, use VPN Gateway (IPsec/IKE) or implement application-level encryption.
- Address space overlap prevents peering and can complicate routing for all connection types—plan IP addressing to avoid conflicts.

---

## Quick decision checklist

- Need lowest latency and highest bandwidth within Azure? -> VNet Peering
- Need encrypted site-to-site connectivity across the internet? -> VPN Gateway (Site-to-Site or VNet-to-VNet)
- Need a private, high-throughput connection with SLA? -> ExpressRoute
- Need secure remote access for individual users? -> Point-to-Site VPN