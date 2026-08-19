
## Site to Site VPN Connection

Site-to-Site connection is essentially a secure VPN that spans between an on-premises VPN device and an Azure VPN Gateway which is located within your Azure GatewaySubnet.

This setup enables our on-premises network to send & receive data to & from a virtual network as of they were part of single large local network.

### Azure GatewaySubnet:
This subnet is a dedicated segment of our VNet, specifically reserved for hosting Azure Virtual Network Gateway & it forms the azure side of the site-to-site VPN connection.

Within the subnet sits the **Virtual Network Gateway (VPN Gateway)**. 

This critical piece of infrastructure is responsible for handling the encryption & decryption of data, maintaining the secure tunnel, and managing the cross-premises connectivity.

Within Azure, we have to create one more resource, which is called **local network gateway**.
**Local network gateway is used to reference our on-premises device IP address.** 

**On-Premises VPN Device :**
This device is a physical or virtual appliance that facilitates a VPN connection from our on-premises network to Azure.
It is configured to match the settings of the Azure VPN Gateway so that a secure tunnel can be established.

If my on prem device has an IP address of 13.12.11.11 , So in local network gateway, We'll tell that my on-premises device IP is 13.12.11.11.

Now when we set up site-to-site connection, I'll be referencing this local-network gateway  inside the connection which actually points to our on-premises device. 