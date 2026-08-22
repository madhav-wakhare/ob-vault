
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




