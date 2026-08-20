
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

3. Data in Use

- **Meaning**: Active data being accessed, read, or modified by a software application or a computer's processor.

- **Where it lives**: Random-access memory (RAM), CPU caches, and processor registers while a program runs.

- **Security focus**: Protected using hardware-based security, memory isolation, and specialized execution environments