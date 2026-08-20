
### Internet Protocol : 

A network protocol is an established set of rules that governs how data is formatted, transmitted, received, and processed between devices. Just as human languages require shared grammar and vocabulary to work, protocols act as a common digital language. They allow different hardware and software to communicate seamlessly across a network.
Core Functions of Protocols

- **Syntax:** Defines the structure and format of the data.
- **Semantics:** Specifies the meaning of each section of the message.
- **Synchronization:** Controls the timing of when data is sent and received.
- **Error Control:** Detects lost or corrupted data and handles retransmission. 

### Latency vs Throughput

**Latency** is the time it takes for a single piece of data or request to travel from its source to its destination (measured in milliseconds). **Throughput** is the total volume of work or data successfully handled by the system over a specific period of time (measured in items or bits per second). 

Core Differences
- **Focus:** Latency measures speed and responsiveness; throughput measures capacity and volume.
- **Units:** Latency is counted in milliseconds (ms) or seconds; throughput is counted in requests per second (RPS) or bits per second (bps).
- **The Highway Analogy:** Latency is how long it takes one car to drive from Exit 1 to Exit 10. Throughput is how many total cars pass through the toll booth per hour.

### Data States :

Data at rest, in transit, and in use describe the three primary states of digital information based on whether it is stored, moving, or being actively processed. Securing each state requires different security methods.

1. Data at Rest
- **Meaning**: Inactive data stored persistently on a physical or cloud medium.
- **Where it lives**: Hard drives, solid-state drives, databases, data warehouses, and cloud storage buckets.
- **Security focus**: Protected using disk encryption, file-level passwords, and strict access controls. 

2. Data in Transit
- **Meaning**: Active data moving from one location to another across a private network or the internet.
- **Where it happens**: Emails sent across servers, web traffic via HTTPS, or file transfers between a device and the [Cloudflare](https://www.cloudflare.com/learning/security/glossary/data-at-rest/).
- **Security focus**: Protected using transport-level security like TLS/SSL encryption to prevent interception.

1. Data in Use
- **Meaning**: Active data being accessed, read, or modified by a software application or a computer's processor.
- **Where it lives**: Random-access memory (RAM), CPU caches, and processor registers while a program runs.
- **Security focus**: Protected using hardware-based security, memory isolation, and specialized execution environments


### Mac Address (Media Access Control) : 

Once we reach to network of ec2 instance via public IP over the internet, that means the ec2 instance must be connected to a router (local network), 
So for forwarding the data from that router to that particular ec2 instance, MAC address is the requirement.

MAC is kind of physical address of that device, IP is kind of virtual address of that device.

IP tells what is the location of device on internet, and once we reach to the local network of that device, we do require MAC address in order to forward the data.

![[Pasted image 20260820221711.png]]

Source & Destination MAC addresses refer to **Network Interface Card Address (NIC)** of devices.

### Network Interface Card :

*It is  a physical device which have kind of an antenna, which can transmits or receive data in form of signals.*
![[Pasted image 20260820222323.png]]

But device A doesn't have information about destination MAC (device B MAC )because device B is not in not in Local network of A. (Simply not part of Local Area Network (LAN) of Device A).

So basically the device A only is aware of device in LAN, it doesn't know any other IP on Internet.

It can only discover the other devices on internet via router. Whenever device A don't know MAC address of device B, it adds a fallback address of default gateway (Router).

**ARP (Address Resolution Protocol):**

**IP address as an input -> ARP -> MAC Address of the device that is holding that IP.**

Because LAN can contain multiple devices, so to identify correct device ARP is used.

**Once the MAC Frame is ready, in the form of bits Network Interface Address (NIC) transmit this frame over the wire - radio signals (WiFi)/ Electric signals (Ethernet)/ Light signals(Fiber opt)**


