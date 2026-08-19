
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
Alternate method for Azure-to-Azure Connectivity(Not for on-prem to Vnet connectivity). This is more straight forward approach that connects VNETs without the need for a VPN Gateway
