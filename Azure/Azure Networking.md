![[Pasted image 20260815202446.png]]

Publish everything via Firewall instead of exposing VM over a public ip address.

**Firewall are designed for controlling both outbound and inbound traffic to and from resources within a VNET. (For VNETs)**

**NSG are typically associated with subnets or individual network interfaces to control traffic within a VNET & between VNETs. (For Subnets & Instances within VNets)**

![[Pasted image 20260815221701.png]]

![[Pasted image 20260815221741.png]]

**NSG in azure are stateful in nature. That means if you allow a port for inbound traffic to receive a request, you don't have to open the port in outbound rules to send response back.** 


![[Pasted image 20260815223213.png]]

![[Pasted image 20260815224335.png]]

![[Pasted image 20260815225459.png]]
If we have 2 VMs in different 2 Vnet and want DNS resolution between them then we have use Private DNS Zones.
You enable automatic registration and resolution of DNS records across multiple VNets with help of Private DNS Zones.
An auto-registration feature can be optionally enabled so that any new VM added to the linked networks automatically registers its DNS record.

Once connected via virtual network links, VMs are able to resolve each other’s names across different networks.

![[Pasted image 20260816104941.png]]

**User data in azure is the scripts we pass at the time of provisioning of VM (Booting).**
**Custom data in azure is the data the is being used and persist in VM till the lifecycle of VM.**


**App Gateway : L7 Loadbalancer (host based routing, path based routing)**
**Azure Load Balancer : L4 Loadbalancer (TCP packets, UDP packets, Low Latency response, IP address:port based)**


![[Pasted image 20260816110202.png]]

