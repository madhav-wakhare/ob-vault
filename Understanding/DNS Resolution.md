# What Happens When You Type www.cnn.com in Your Browser?

#networking #dns #http #interview-prep

> [!info] Overview
> When you type a URL into your browser, three major things happen in sequence:
> 1. **DNS** – find the IP address for the domain
> 2. **Network Communication** – establish a connection to that IP
> 3. **HTTP** – actually request and receive the webpage

**Assumption:** You're on a Linux client, connected via a Comcast home cable connection.

---

## 1️⃣ DNS — Finding the IP Address

- The client first checks its **name service cache daemon** — "have I looked this up before?"
- If not cached, the request goes to the **name server** listed in `/etc/resolv.conf`.
- That name server is usually your **home router**, which forwards the request to Comcast's DNS resolver.
- Comcast's resolver forwards the request to the **.com root name servers**.
- The root servers look up the **NS (nameserver) records** for `cnn.com` — these were set when the domain was registered with the registrar.
- The request is sent to **CNN's own nameservers**, which check their **zone files** and reply with the actual IP.

> [!note] Zone Files
> - **Forward lookup zone** → matches hostname → IP address
> - **Reverse lookup zone** → matches IP address → hostname

> [!tip] Recursive vs Non-Recursive Queries
> - **Recursive query** — client asks a question and expects the *final answer*; the DNS server does all the work of finding it.
> - **Non-recursive query** — DNS server doesn't give the final answer; the client must go ask the *next* server itself.

> [!note] Caching
> Each DNS server along the way may have its own cache. So at any point, the answer might come straight from cache instead of a fresh lookup.

---

## 2️⃣ Network Communication — Connecting to the Server

Once the IP address is known, a **TCP 3-way handshake** happens:

1. **SYN** – client sends a connection request
2. **SYN-ACK** – server acknowledges and sends its own request
3. **ACK** – client confirms → connection established ✅

### How the packet actually travels:
- The client checks its **routing table** for an entry to CNN's network.
- No entry? → send to the **default gateway**.
- The gateway repeats this process, hopping along until it reaches the **Internet gateway running BGP**.
- **BGP (Border Gateway Protocol)** holds the routing table of all public IPs assigned by ISPs — including CNN's.
- Comcast sets the destination and sends the packet across the internet to CNN's server.
- CNN's server responds with SYN-ACK, then the client sends the final ACK.

> [!question] How does the client know what's "local" vs "outside"?
> Through the **netmask**.
> Example: IP `10.1.1.100` with netmask `255.255.255.0` means `10.1.1.0–10.1.1.256` is the local subnet — anything outside that range is external.

> [!note] ARP (Address Resolution Protocol)
> Used to find the **MAC address** of the default gateway at Layer 2.
> - Client asks: *"Who has 10.1.1.1?"*
> - Router replies with its MAC address
> - Client then wraps the packet with that MAC address to send it

---

## 3️⃣ HTTP — Getting the Actual Webpage

- The request arrives at CNN's server — possibly behind a **load balancer**.

> [!note] Load Balancer Modes
> - **In-line** — load balancer manages both incoming *and* outgoing traffic between client and server
> - **DSR (Direct Server Return)** — incoming traffic passes through the load balancer, but outgoing traffic goes directly from server to client

- **Apache** (the web server) receives the request on **port 80**, then hands it off using either:
  - **Pre-fork mode** (default) → uses **forked processes**
  - **Worker mode** (`worker.c`) → uses **threads** (lighter on resources, more complex)

---

## 🎯 Interview Tip

This question is a great "starting point" answer — from here, you can go deeper into any branch:

| Topic | Deeper Dive Ideas |
|---|---|
| **DNS** | SOA record, other DNS record types, setting up a BIND server |
| **Networking** | TCP vs UDP, routing protocols (RIP, OSPF, BGP) |
| **HTTP** | Setting up Apache, encrypted vs unencrypted traffic, SSL/TLS |

---

## 🔗 Related Notes
- [[DNS Records Overview]]
- [[TCP 3-Way Handshake]]
- [[Load Balancers - In-line vs DSR]]