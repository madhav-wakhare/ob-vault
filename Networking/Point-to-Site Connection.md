
Company provides Domian Joined Laptop. 
*(A domain-joined laptop is a computer connected to a company's centralized network (using systems like Windows Active Directory or Azure AD). This link lets your company's IT department control security rules, push software updates, and verify your work login from afar.)*

Remote Laptop (Away from Office Premises) can connect to Office Network through a VPN Tunnel.
The data flowing through this tunnel will be encrypted in nature.

For P2S Connection, SSTP (Secure Socket Tunneling Protocol) & IKEv2 (Internet Key Version) protocols are used under the hood.

Secure Socket Tunneling Protocol (SSTP) helps in creating tunnel while IKEv2 helps in generating encryption keys and helps in encrypting the data.

![[Pasted image 20260822115522.png]]
We cannot connect directly to VNET of Azure through laptop, we need a VPN Gateway in between. 

VPN Gateway is attached with VNET through Gateway Subnet created in that VNET.
This VPN Gateway is assigned with a Public IP.

![[Pasted image 20260822120257.png]]

---
### SKU : 
Nothing but the configuration based on price and performance of the resource.
![[Pasted image 20260822122331.png]]
Max 128 connections here means we can connect remote 128 laptops through SSTP to Azure.
If our need is more than this we can use IKEv2/OpenVPN Connections which give 250 max connections to Azure.
In Basic SKU, we can have 200 VM in a VNET which can be connected through Tunnel.

**We can only upgrade the SKU version, we can't rollback to previous version.**
 
 ---
### active-active mode  and active-passive mode:
If VPN gateway goes down then the connections to VM in VNET attached to that VPN will be terminated which will bring business impact as Remote workers will not be able to work.

So we need High Availability for VPN Gateway as well.

So there are 2 modes for maintaining high-availability :
1. active-active
2. active-passive

In backend Microsoft Azure always deploys 2 VPN Gateways regardless of the mode.

In Active-Active mode, both VPN Gateways will be active all time. Both will have public IPs. Traffic distribution will be equal on both the gateways. 

So in Active-Active we need 2 public ips for 2 VPN gateways in background.

In Active-Passive mode, If a VPN Gateways goes down, Other will then only handle traffic. It will not be active all time.


***Connections are used for only site-to-site connectivity.***

