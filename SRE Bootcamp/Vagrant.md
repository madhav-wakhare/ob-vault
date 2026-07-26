Local development is crucial to mimic the production ready environment without internet, here at one2n it is called **Airplane Mode Development**.
Vagrant helps in this by spinning up quick, portable & reproducible virtual machines. It is declarative, ensuring consistency in provisioning resources. 

#### Hypervisors :
A **hypervisor**, also known as a **virtual machine monitor** (**VMM**), is a type of computer [software](https://en.wikipedia.org/wiki/Software "Software"), [firmware](https://en.wikipedia.org/wiki/Firmware "Firmware") or [hardware](https://en.wikipedia.org/wiki/Computer_hardware "Computer hardware") that creates and runs [virtual machines](https://en.wikipedia.org/wiki/Virtual_machine "Virtual machine"). A computer on which a hypervisor runs one or more virtual machines is called a _host machine_ or _virtualization server_, and each virtual machine is called a _guest machine_. The hypervisor presents the guest operating systems with a [virtual operating platform](https://en.wikipedia.org/wiki/Platform_virtualization "Platform virtualization") and manages the execution of the guest operating systems. Unlike an [emulator](https://en.wikipedia.org/wiki/Emulator "Emulator"), the guest executes most instructions on the native hardware.[[1]](https://en.wikipedia.org/wiki/Hypervisor#cite_note-goldberg1973-1) Multiple instances of a variety of operating systems may share the virtualized hardware resources: for example, [Linux](https://en.wikipedia.org/wiki/Linux "Linux"), [Windows](https://en.wikipedia.org/wiki/Microsoft_Windows "Microsoft Windows"), and [macOS](https://en.wikipedia.org/wiki/MacOS "MacOS") instances can all run on a single physical [x86](https://en.wikipedia.org/wiki/X86 "X86") machine. This contrasts with [operating system-level virtualization](https://en.wikipedia.org/wiki/Operating_system-level_virtualization "Operating system-level virtualization"), where all instances (usually called _[containers](https://en.wikipedia.org/wiki/Containerization_\(computing\) "Containerization (computing)")_) must share a single kernel, though the guest operating systems can differ in [user space](https://en.wikipedia.org/wiki/User_space "User space"), such as different [Linux distributions](https://en.wikipedia.org/wiki/Linux_distribution "Linux distribution") with the same kernel.

**Hardware virtualization** (or platform virtualization) uses specialized processor features (like Intel VT-x or AMD-V) to create a virtual machine (VM) that behaves like a real physical computer. A hypervisor interacts directly with the physical hardware to assign CPU, memory, and storage, making it incredibly fast and efficient.

**Software virtualization** creates virtual environments entirely within software or at the operating system level, without relying on special processor extensions. It is commonly used for emulators (e.g., running old console games on a PC) or OS-level virtualization like containers, which share the host system's kernel for high agility.

#### Type 1 hypervisors :
They run directly on bare-metal hardware, it requires underlying OS to manage system resources, making it ideal for individual PCs, software development.

#### Type 2 hypervisors (hosted hypervisor):
A Type 2 hypervisor (or hosted hypervisor) is a virtualization software layer that runs as a standard application on top of an existing host operating system.

![[Pasted image 20260708114023.png]]


**The Problem with MAC :**
Vagrant provides **support for Type-2 Hypervisor** in market (VirtualBox, Hyper-V, VMWare & Docker) .

VirtualBox (Type 2 hypervisor doesn't support Mac M-series apple products).
Even if we run the the Virtualbox, VMs get crashed and gets stuck in boot phases on Mac.
### **Then Comes the UTM : A Native hypervisor for MacOS.**

UTM is a 3rd Party - MacOS native [QEMU](https://github.com/qemu/qemu)/[hypervisor](https://en.wikipedia.org/wiki/Hypervisor) that leverages Apple's Hypervisor virtualization framework. It allows you to run **ARM64** operating systems on Apple Silicon at near-native speeds and also supports lower performance emulation for running **x86/x64** on Apple Silicon and **ARM64** on Intel Macs.

UTM doesn't officially supports the Vagrant but there is community driven plugin (vagrant_utm) for vagrant which enables Vagrant to control, provision, and destroy VMs using UTM's APIs.
This plugin is a crucial step in making UTM work seamlessly with Vagrant.

## Install Vagrant - 
brew tap hashicorp/tap
brew install hashicorp/tap/hashicorp-vagrant

## Install UTM -  
brew install --cask utm

## Install Vagrant UTM Plugin - 
vagrant plugin install vagrant_utm

Vagrant automate the configuration of VM, All things such as :
1. Download the os image
2. Create VM
3. Create Networks
4. Configure Networking
5. Configure Port fowarding 
6. Boot up VM

`vagrant init ubuntu/24.04` : This deploy a vagrant box consisting of ubuntu 24.04

A box is a vagrant term & refers to a packaged format of a vagrant environment. 
It contains :
- os image
- scripts required to configure the environment.

Running `vagrant init` command initializes the vagrant box in the current directory and creates a Vagrantfile.

The Vagrantfile has the instructions on customizing your box.

 To start the vagrant box use `vagrant up` command. Vagrant downloads the image required to create the VM, it then creates the VM, gives it a random name and configures any settings such as port forwarding & waits for it to be ready. 


#### Gotchas to watch out for when using this Vagrant file:

1. **UTM Pop-Up and Permission Prompt**
    
    - Don’t miss the UTM pop-up to confirm the VM setup.
        
    - Your terminal will prompt for a `y/N` confirmation to download the VM image. Say `y` or the setup will halt.
        
2. **Downloading the VM Image**
    
    - Ensure a stable internet connection and sufficient disk space. A failed download means starting over.
        
3. **Manual Steps in UTM**
    
    - After the VM image downloads, manually mount the project folder in UTM’s "Shared Directory" section.
        
    - **Set the sharing mode to "virtFS"** for smooth file access, or your files won’t appear.
        
    - If the VM doesn’t boot right away, check and configure boot options or firmware manually, especially for non-standard setups.
        
4. **Hidden Files Aren’t Your Friend**
    
    - UTM skips hidden files during mounting. Rename `.env` to `env` so the VM can see it.
        
5. **Port Forwarding Traps**
    
    - Ensure port **8080** on your host is free. If it’s in use, NGINX forwarding will fail.
        
    - Verify everything works by testing `http://localhost:8080` once the VM is up.


##### **Vagrantfile**

The primary function of the Vagrantfile is to describe the type of machine required for a project, and how to configure and provision these machines.

Vagrant is meant to run with one Vagrantfile per project, and the Vagrantfile is supposed to be committed to version control. This allows other developers involved in the project to check out the code, run `vagrant up`, and be on their way. Vagrantfiles are portable across every platform Vagrant supports.

The syntax of Vagrantfiles is **Ruby**.

## Lookup Path

When you run any `vagrant` command, Vagrant climbs up the directory tree looking for the first Vagrantfile it can find, starting first in the current directory. So if you run `vagrant` in `/home/mitchellh/projects/foo`, it will search the following paths in order for a Vagrantfile, until it finds one:

```
/home/mitchellh/projects/foo/Vagrantfile
/home/mitchellh/projects/Vagrantfile
/home/mitchellh/Vagrantfile
/home/Vagrantfile
/Vagrantfile
```

You can change the starting directory where Vagrant looks for a Vagrantfile by setting the `VAGRANT_CWD` environmental variable to some other path.

```
Vagrant.configure("2") do |config|
  # ...
end
```

The `"2"` in the first line above represents the version of the configuration object `config` that will be used for configuration for that block (the section between the `do` and the `end`). This object can be very different from version to version.

## Mounting Folders in Vagrant : 

**Vagrant does not mount folders by itself. It tells the provider (UTM, VirtualBox, VMware, etc.) how to mount them.**

So, 

```
u.directory_share_mode = "virtFS"
```

is not adding an extra mount on top of Vagrant’s mount.
It is telling UTM **which mechanism to use for the mount that Vagrant wants.**

