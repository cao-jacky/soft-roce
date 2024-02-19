# soft-roce
Using the [following](https://enterprise-support.nvidia.com/s/article/howto-configure-soft-roce) NVIDIA article on deploying Soft-RoCE on Ethernet-connected infrastructure. This repository and README.md contains the steps that I have followed and used to get the software implementation of RDMA transport working on CSC (i.e., Finnish state IT and computing infrastruture) infrastructure. 

To get Soft-RoCE running you need to configure and install,
1. A Linux kernel with "Software RDMA over Ethernet" support
2. User space libraries which support Soft-RoCE itself

## Installation
As described in the NVIDIA document, both kernel and user space libraries are needed on both servers. 

In terms of servers that I am using/have used, the following table contains the tested CSC virtual machine flavours and whether they worked or not. 

| Flavour | Cores |  Memory (GB) | Notes |
|---|---|---|---|---|
| standard.tiny | 1 | 0.9 | [Unsuccesful] Kernel compilation takes too long |
| standard.small | 2 | 2.9 | [To-test] |
| standard.medium | 3 | 3.9 | [To-test] |
| standard.large | 4 | 7.8 | [Succesful] Kernel and user space libaries are able to be built |

For each of the virtual machines, I chose to use Ubuntu 22.04 as the base image/operating system.

### Kernel installation
1. The kernel should be cloned 