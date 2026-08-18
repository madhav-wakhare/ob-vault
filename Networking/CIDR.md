
Classless Inter Domain Routing

Network have its own IP as it needs to be denoted on the basis of IP.  (ex : 192.168.1.0)
It helps network to identify if a device is within its network or not.

**CIDR  (X) :**
X is called CIDR notation here.

192.168.1.0/X -> X is the integer between 0 & 32 bits which is representing how many bits are reserved to my network.

So if we take 192.168.1.0/24 so my first 24 bits or first 3 octet of an ipare reserved for my network.
That means whichever devices/hosts are present in the network, their ip's first 24 bits are always same and constant.

Last 8 bits or last octet is reserved for host. With help of DHCP (Dynamic Host Configuration Protocol) a host gets IP from the network in which it is present.

So the above network can only contain 2^8 machines as only last octet is reserved for host (256 Ips). 256 hosts per network out of which only 254 are usable as 1st IP is for network (192.168.1.0) and last IP is for Broadcast.

![[Pasted image 20260818220853.png]]

With help of subnet mask we can tell if a device belongs to a network or not.

0.0.0.0/0 -> This represents entire internet IPs. Every IP satisfies this, no network part is reserved in this case.

If we have /16 as CIDR :
 192.168.1.0/16 -> Then we can have 2^16 (65000) hosts per network.