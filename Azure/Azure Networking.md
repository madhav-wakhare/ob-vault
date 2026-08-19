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

Azure networking = two separate problems. Beginner confusion come from mixing them.

1. Reachability — can packet get from A to B? IP, routing, firewall.
2. Name resolution — what IP does mydb.database.windows.net map to? DNS.

Peering solve #1. Private DNS solve #2. Neither replace other.

Building blocks (purpose each)

- VNet — private IP address space, isolated network boundary. Nothing outside reach in by default.
- Subnet — slice of VNet. Group resources, attach rules, delegate to services.
- NIC / private IP — what VM actually own inside subnet.
- NSG — allow/deny rules on subnet or NIC. 5-tuple filter. Purpose: stop unwanted traffic.
- Route table (UDR) — override default routing. Purpose: force traffic through firewall/NVA.
- Public IP / NAT Gateway / Load Balancer — controlled in/out to internet.
- VPN Gateway / ExpressRoute — connect VNet to on-prem.
- Private Endpoint — NIC inside your subnet that represent a PaaS service (Storage, SQL, Key Vault). Purpose: reach PaaS over private IP, no public internet.

VNet peering

Connect two VNets so their private IPs route to each other. Traffic stay on Azure backbone.

- Non-transitive. A-B and B-C peered does not give A-C. Need hub-spoke with gateway/firewall, or direct peering.
- Address spaces must not overlap.
- Both sides must create own peering link.
- Gives routing only. No DNS effect at all.

Purpose: "my VM in VNet-A must talk to VM in VNet-B."

Private DNS zone

Azure-hosted DNS zone visible only to VNets you link. Example zone: privatelink.blob.core.windows.net.

- Link zone to VNet via virtual network link. That link is what make VNet resolve names in zone.
- Optional autoregistration — VMs in linked VNet auto-create own A records. Max one autoregistration zone per VNet.
- Resolution happen through Azure-provided DNS at 168.63.129.16.

Purpose: "when app ask for mystorage.blob.core.windows.net, give private IP of private endpoint, not public IP."

Your question — does Private DNS need peering?

No. Independent features.

- Private DNS zone resolve for any VNet that has virtual network link to zone. Peering irrelevant to resolution.
- One zone can link to 10 unrelated, unpeered VNets. All resolve fine.

Two traps, opposite directions:

Trap 1 — peering does not share DNS. Peer VNet-A to VNet-B, link zone only to A. B still resolve nothing. Must create separate virtual network link for B. Links never inherit through peering.

Trap 2 — DNS without route is useless. Link zone to VNet-C, no peering to VNet where private endpoint live. C resolve name to 10.1.0.4 correctly, then connection time out. Name resolved, packet nowhere to go.

So: resolution needs link. Working connection needs link + route (peering, VPN, or endpoint in same VNet).

Typical hub-spoke pattern

- Private endpoints in hub VNet.
- Private DNS zones linked to hub and every spoke (or use DNS forwarder in hub).
- Spokes peered to hub for routing.
- On-prem needs Azure DNS Private Resolver or DNS forwarder VM — on-prem cannot query Azure private zones directly, must forward to 168.63.129.16 from inside a linked VNet.

Both pieces required. Peering carry packet, DNS link tell packet where to go.

result: Explained Azure networking layers, VNet peering vs Private DNS zones — separate features, Private DNS works without peering (needs virtual network link), but peering never propagates DNS links and DNS without routing resolves to unreachable IPs.