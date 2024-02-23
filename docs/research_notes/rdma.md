# Random direct memory access

Random direct memory access (RDMA) allows computers to access the memory of another computer over a network without the need to involve the host machines' CPUs. 

- This could lead to high-throughput, low-latency networking netween systems.
- RDMA is commonly used for high-performance computing with applications in big data, video streaming, and cloud computing

> RDMA supports zero-copy networking by enabling the network adapter to transfer data from the wire directly to application memory or from application memory directly to the wire, eliminating the need to copy data between application memory and the data buffers in the operating system.

--- 

There are several ways that RDMA is implemented:
1. [RDMA over Converged Ethernet (RoCE)](roce.md)
2. [InfiniBand](infiniband.md)
3. [iWARP](iwarp.md)

Typically this involves implementing a network adapter that implements RDMA over an Ethernet connection. This adapter can be the form of a hardware networking interface card (NIC), or, as a software-based adatper, e.g., Soft-RoCE.   


# References
- [https://community.fs.com/article/roce-rdma-over-converged-ethernet.html](https://community.fs.com/article/roce-rdma-over-converged-ethernet.html)



