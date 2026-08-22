---

title: Azure ↔ AWS Site-to-Site VPN — Setup Guide

tags:

- azure

- aws

- vpn

- ipsec

- strongswan

---


````
# Azure ↔ AWS Site-to-Site VPN

## 1. Overview

A **Site-to-Site VPN** creates an encrypted connection between an Azure VNet and an AWS VPC over the public internet.

In this lab:

- **Azure** → VPN Gateway
- **AWS** → EC2 running strongSwan
- **Protocol** → IKEv2 + IPsec
- **Authentication** → Pre-Shared Key (PSK)
- **Tunnel** → Bidirectional
- **Traffic** → Split tunnel

---

## 2. Lab Architecture

```mermaid
flowchart LR
    AWS["AWS VPC<br>10.20.0.0/16"]
    EC2["EC2 + strongSwan<br>10.20.1.9"]
    VPN["Encrypted IPsec Tunnel<br>IKEv2"]
    AZ["Azure VPN Gateway"]
    AZVNET["Azure VNet<br>10.0.0.0/16"]

    AWS --> EC2
    EC2 <--> VPN
    VPN <--> AZ
    AZ --> AZVNET
````

> If Obsidian still has trouble rendering `<br>`, use the even simpler version below:

````
```mermaid
flowchart LR
    A[AWS VPC] --> B[AWS EC2 + strongSwan]
    B <--> C[IPsec VPN Tunnel]
    C <--> D[Azure VPN Gateway]
    D --> E[Azure VNet]
```
````

---

## 3. Important Components

|Component|Purpose|
|---|---|
|**Azure VNet**|Azure private network|
|**Azure VPN Gateway**|VPN endpoint on Azure|
|**AWS VPC**|AWS private network|
|**AWS EC2**|Acts as the VPN endpoint|
|**strongSwan**|IPsec VPN software on EC2|
|**IKEv2**|Negotiates the VPN security association|
|**IPsec**|Encrypts the actual network traffic|
|**PSK**|Authenticates both VPN endpoints|

---

## 4. VPN Configuration

### Azure

```
Azure VNet:       10.0.0.0/16
VPN Gateway:      Azure VPN Gateway
Public IP:        Azure VPN Gateway Public IP
IKE Protocol:     IKEv2
Authentication:   PSK
```

### AWS

```
AWS VPC:          10.20.0.0/16
VPN Endpoint:     EC2
VPN Software:     strongSwan
Public IP:        EC2 Public IP
IKE Protocol:     IKEv2
Authentication:   PSK
```

---

## 5. strongSwan Configuration

The important parts of `/etc/ipsec.conf`:

```
keyexchange=ikev2
authby=psk

left=%defaultroute
leftid=<AWS_PUBLIC_IP>
leftsubnet=10.20.0.0/16

right=<AZURE_VPN_PUBLIC_IP>
rightsubnet=10.0.0.0/16

ike=aes256-sha256-modp2048!
esp=aes256-sha256!

auto=start
```

The `!` means:

> Use **only** this proposal. Do not negotiate other algorithms.

---

## 6. IKE vs IPsec

```
IKEv2
  │
  ├── Authenticates peers
  ├── Negotiates encryption
  └── Creates security association
          │
          ▼
       IPsec
          │
          └── Encrypts application traffic
```

Think of it as:

**IKEv2 = negotiates the secure tunnel**

**IPsec = carries encrypted traffic through the tunnel**

---

## 7. Split Tunnel vs Full Tunnel

### Split Tunnel

Only traffic destined for the remote private network goes through the VPN.

```
AWS → 10.0.0.0/16
        ↓
     VPN Tunnel
        ↓
      Azure

AWS → Internet
        ↓
   Normal Internet
```

Our lab uses **split tunneling** because the traffic selectors are:

```
10.20.0.0/16 === 10.0.0.0/16
```

So only traffic between these networks enters the VPN.

### Full Tunnel

All traffic goes through the VPN.

```
AWS → VPN → Azure → Internet
```

A full tunnel would typically use:

```
0.0.0.0/0
```

as the remote traffic selector, along with the appropriate routing configuration.

---

## 8. How to Verify the Tunnel

### Check strongSwan

```
sudo ipsec statusall
```

Look for:

```
Security Associations (1 up)
```

and:

```
ESTABLISHED
```

Example:

```
azure-vpn[1]: ESTABLISHED
azure-vpn{1}: INSTALLED, TUNNEL
```

### Check traffic

```
ping <Azure-VM-private-IP>
```

If the ping succeeds, traffic is successfully crossing the VPN.

---

## 9. Useful Troubleshooting

### Check VPN status

```
sudo ipsec statusall
```

### Restart strongSwan

```
sudo ipsec restart
```

### Manually initiate VPN

```
sudo ipsec up azure-vpn
```

### Check logs

```
sudo journalctl -u strongswan -f
```

Common errors:

|Error|Meaning|
|---|---|
|`NO_PROPOSAL_CHOSEN`|IKE/IPsec algorithms don't match|
|`AUTHENTICATION_FAILED`|PSK or identity mismatch|
|`duplicate CHILD_SA`|Tunnel is already established|
|`ESTABLISHED`|IKE tunnel is successfully established|
|`INSTALLED, TUNNEL`|IPsec CHILD_SA is installed|

---

## 10. Final Mental Model

```
AWS VPC
10.20.0.0/16
      │
      ▼
EC2 + strongSwan
      │
      │ IKEv2
      │ IPsec
      │ PSK
      ▼
══════════════════════
   Encrypted Tunnel
══════════════════════
      │
      ▼
Azure VPN Gateway
      │
      ▼
Azure VNet
10.0.0.0/16
```

### Remember

> **IKEv2 establishes the secure relationship.**
> 
> **IPsec encrypts the traffic.**
> 
> **PSK authenticates the endpoints.**
> 
> **Routing/traffic selectors determine what traffic enters the tunnel.**
> 
> **Our lab is a split-tunnel, site-to-site VPN.**

```

The main change I'd recommend is **not using complex `subgraph` syntax** for this diagram. The simple `flowchart LR` version is much less likely to cause rendering issues in Obsidian.
```