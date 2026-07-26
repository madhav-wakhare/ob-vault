Proxmox :
**Proxmox is a bare-metal Type-1 hypervisor**, whereas ==VirtualBox (vmbox) is a **hosted Type-2 hypervisor**==.

The fundamental difference lies in how they interact with your physical hardware. Proxmox _is_ the operating system and runs directly on the machine's hardware, while VirtualBox is an application that you install inside an existing operating system like Windows or macOS.

┌───────────────────────────────┐ ┌───────────────────────────────┐ │ Virtual Machines / LXC │ │ Virtual Machines │ ├───────────────────────────────┤ ├──[[Questions]]─────────────────────────────┤ │ Proxmox VE (Type 1) │ │ VirtualBox (Type 2) │ ├───────────────────────────────┤ ├───────────────────────────────┤ │ Physical Hardware │ │ Host OS (Windows/macOS) │ │ │ ├───────────────────────────────┤ │ │ │ Physical Hardware │ └───────────────────────────────┘ └───────────────────────────────┘

![[Pasted image 20260724111447.png]]