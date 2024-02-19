# soft-roce
Using the [following](https://enterprise-support.nvidia.com/s/article/howto-configure-soft-roce) NVIDIA article on deploying Soft-RoCE on Ethernet-connected infrastructure. This repository and README.md contains the steps that I have followed and used to get the software implementation of RDMA transport working on CSC (i.e., Finnish state IT and computing infrastruture) infrastructure. 

The NVIDIA article has some errors/instructions which do not work and have been changed in this documentation.

To get Soft-RoCE running you need to configure and install,
1. A Linux kernel with "Software RDMA over Ethernet" support
2. User space libraries which support Soft-RoCE itself

## Installation
As described in the NVIDIA document, both kernel and user space libraries are needed on both servers. 

In terms of servers that I am using/have used, the following table contains the tested CSC virtual machine flavours and whether they worked or not. 

| Flavour | Cores |  Memory (GB) | Notes |
|---|---|---|---|
| standard.tiny | 1 | 1 | [Unsuccesful] Kernel cloning from repository takes too long |
| standard.small | 2 | 3 | [Unsuccesful] Kernel cloning from repositort takes too long |
| standard.medium | 3 | 4 | [Unsuccesful] Kernel and user space libaries are able to be built, but note that kernel compilation takes a **long** time. However, once restarting, there is not enough memory to start |
| standard.large | 4 | 8 | [Succesful] Kernel and user space libaries are able to be built, but note that kernel compilation takes a **long** time |

For each of the virtual machines, I chose to use Ubuntu 22.04 as the base image/operating system.

### Kernel installation
1. The kernel should be cloned. The NVIDIA instructions say to clone version 4.9.0 of the kernel but from testing, the latest version of the Linux kernel appears to have the appropriate support. As of writing this documentation (2024-02-19 10:50), the cloned version of the kernel was Linux 6.8-rc5.

Move into `/usr/src/`.

```sh
cd /usr/src
```

Clone the Linux kernel - `sudo` permissions may or may not be needed.

```sh
sudo git clone git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git linux
```

2. Compiling the cloned Linux kernel requires some configuration.

```sh
cd linux
sudo cp /boot/config-$(uname -r)* .config
```

Before configuring the kernel, there are some packages which need to be installed before running the `make` command. 

```sh
sudo apt-get install make gcc libncurses-dev flex bison
```

Once these are installed, now the kernel can be compiled.

```sh
sudo make menuconfig
```

This window will open:

![kernel configuration window](images/kernel_configuration1.png)

3. To enable Soft-RoCE, the following needs to be done.
    1. Press the `/` key and a search field will be opened.
    2. Type `rxe` and hit the `Enter` key, i.e., selecting the highlighted `<  Ok  >`.
    3. Press the `1` key to go to the "Software RDMA over Ethernet (RoCE) driver".
    4. You should see that it is selected as it is specified as `<M>`. If not, press the `Space` key until you see `<M>` and **not** `< >`.
    5. Using the right arrow key, move to the option `< Save >` and press the `Enter` key.
    6. The filename should be left as `.config` and the `Enter` key can be pressed to select `<  Ok  >`.
    7. The `Enter` key can be pressed again to select `< Exit >`.
    8. Using the right arrow key, select `< Exit >` and press the `Enter` key until the wizard is exited. This might require you to re-selected `< Exit >`, so do not spam the `Enter` key all at once. 

![kernel configuration window with Soft-RoCE driver selected](images/kernel_configuration2.png)

4. With the kernel configured, steps can be taken towards compiling it. 

There are some additional libraries which need to be installed.

```sh
sudo apt-get install libelf-dev libssl-dev
```

To prevent the following make error during compilation, several steps need to be taken.

```sh
make[3]: *** No rule to make target 'debian/canonical-certs.pem', needed by 'certs/x509_certificate_list'.  Stop.
```

These steps are from [here](https://stackoverflow.com/questions/67670169/compiling-kernel-gives-error-no-rule-to-make-target-debian-certs-debian-uefi-ce). 

```sh 
sudo mkdir -p /usr/local/src/debian
sudo apt-get install linux-source
sudo cp -v /usr/src/linux-source-*/debian/canonical-*.pem /usr/local/src/debian/
sudo apt-get purge linux-source*
```

Then, the `.config` file needs to be updated so the certificate paths for signature checking are correct.
```sh
sudo vim .config
```

The first variable to change is `CONFIG_SYSTEM_TRUSTED_KEYS`, from,
```conf
CONFIG_SYSTEM_TRUSTED_KEYS="debian/canonical-certs.pem"
```

to, 
```conf
CONFIG_SYSTEM_TRUSTED_KEYS="/usr/local/src/debian/canonical-certs.pem"
```

Then, the second variable to change is `CONFIG_SYSTEM_REVOCATION_KEYS`, from, 
```conf
CONFIG_SYSTEM_REVOCATION_KEYS="debian/canonical-revoked-certs.pem"
```

to
```conf
CONFIG_SYSTEM_REVOCATION_KEYS="/usr/local/src/debian/canonical-revoked-certs.pem"
```

5. Now the kernel can be compiled. 

Using `nproc` check how many processing units are available for the compilation, e.g., `4`. Then the kernel can be compiled. NB: replace 4 in the following commands with whatever output `nproc` gives. 

```sh
sudo make -j 4
```

Depending on the number of available procesing units, this compilation may take a while, e.g., 3 to 4 hours.

Install the modules.

```sh
sudo make -j 4 modules_install
```

Install the kernel.

```sh
sudo make -j 4 install
```

6. Reboot the server and login again.

```sh
sudo shutdown -r now
```

7. Check that the new kernel and the rdma_rxe module were installed.

```sh
uname -r
```

![uname check](images/uname_check.png)

```sh
modinfo rdma_rxe
```

![rdma rxe check](images/rdma_rxe_check.png)

If it looks similar then the kernel and module were correctly configured.

## User Space Libraries Installation

According to the NVIDIA article, the user space libraries which support Soft-RoCE have not been distributed and requires installation. The article provides instructions for manual building and installation from the [rdma-core repository](https://github.com/linux-rdma/rdma-core). However, there appears to be a pre-compiled version available through `apt-get`.  

1. Install some pre-requisites.

```sh
sudo apt-get install build-essential cmake gcc libudev-dev libnl-3-dev libnl-route-3-dev ninja-build pkg-config valgrind python3-dev cython3 python3-docutils pandoc
```

2. Install `rdma-core`.

```sh
sudo apt-get install rdma-core
```

## Usage and testing
Software RDMA should now be made available on an existing (Ethernet) interface using one of the available drivers. 

1. Check which interfaces are available and especially the name of the interface.

```sh
ip link
```

![ip info](images/ip_info.png)

In my example, the Ethernet interface is called `ens3`.

2. Now an RXE device should be created over the network interfae, e.g., `ens3`.

```sh
sudo rdma link add rxe_ens3 type rxe netdev ens3
```

Where `rxe_ens3` is the *name* of the device, `rxe` is the type of the driver, and `ens3` is the network interfacce in question. 

3. The link command provides no output so the status command can be called to display the current configuration. 

```sh
rdma link
```

![rdma link](images/rdma_link.png)

Alternatively, `ibv_devices` can be used to verify the device was created succesfully but this requires some tools to be installed.

```sh
sudo apt-get install ibverbs-utils
```

![ibv devices](images/ibv_devices.png)

---

Now, this installation should be replicated on one (or more) servers/virtual machines which are connected by Ethernet. This can be done through following the instructions again, or by creating a snapshot of the server and deploying that.  