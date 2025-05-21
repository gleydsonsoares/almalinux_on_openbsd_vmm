## 🐧 AlmaLinux on OpenBSD VMM Hypervisor 🐡

Running AlmaLinux as a guest operating system on top of OpenBSD’s built-in hypervisor, [vmm(4)](https://man.openbsd.org/vmm) / [vmd(8)](https://man.openbsd.org/vmd).

This approach utilizes officially distributed generic cloud disk images in a straightforward manner. Instead of relying on ISO files, it boots directly on the fly, delivering a fully operational Linux OS after just a few steps.

![AlmaLinux on OpenBSD VMM](https://github.com/gleydsonsoares/almalinux_on_openbsd_vmm/blob/main/almalinux_openbsd_vmm.png)


## Enable and start the OpenBSD's [vmd(8)](https://man.openbsd.org/vmd) daemon

```bash
doas rcctl enable vmd
doas rcctl start vmd
```
Run [fw_update(8)](https://man.openbsd.org/fw_update) to grab and install the vmm-firmware package which brings firmware binary images. It is not installed by default

``` bash
$ doas fw_update
```

Otherwise, you can find an error when trying to start the VM in next steps:
***vmctl: vmm bios firmware file not found.***

## Create the cloud init data

Most people overlook how simple it is to create a working cloud-init ISO using just two files — meta-data and user-data — along with the [mkhybrid(8)](https://man.openbsd.org/OpenBSD-6.9/mkhybrid) utility, which is already included in the OpenBSD base system. No additional tools or packages are required.

Create a empty meta-data file:

```bash
$ touch meta-data
```

Create the user-data file, it stores login's data that will be loaded during the first boot to get access to VM, 
so I'd recommend using ssh pub/key since it is safer, the file is in an YAML format, the first file's line needs to be hard-coded to
**#cloud-config**, otherwise, it won't be loaded by cloud-init during boot stage

 ```bash
#cloud-config

ssh_authorized_keys:
 - ssh-ed25519 AAAAAAAAAAAAAA user@example
```
It should result in two files:

```text
blowfish$ ls -la user-data meta-data                                                                                                                      
-rw-r--r--  1 gsoares  gsoares    0 May  9 08:32 meta-data
-rw-r--r--  1 gsoares  gsoares  155 May  9 08:34 user-data
```

Pack them into a CDROM .iso. OpenBSD already ships with a tool in base [mkhybrid(8)](https://man.openbsd.org/OpenBSD-6.9/mkhybrid), so there's no need to install extra software:

```bash
$ mkhybrid -o cloud.iso -V CIDATA -J -R user-data meta-data
```


## Fetch the latest generic cloud disk from Official AlmaLinux repo

```bash
$ curl -O -s https://repo.almalinux.org/almalinux/9/cloud/x86_64/images/AlmaLinux-9-GenericCloud-latest.x86_64.qcow2
```

Verify the image, https://wiki.almalinux.org/cloud/Generic-cloud.html#verify-almalinux-9-images

## Preparing the disk image

AlmaLinux qcow2 disk uses compression, there's no support on OpenBSD:

```
May 10 18:56:02 blowfish vmd[10407]: vm/AlmaLinux/vioblk0: xlate: compressed clusters unsupported
```

It can be managed by converting it into uncompressed qcow2 format by using qemu-img available in OpenBSD ports. 

```bash
$ doas pkg_add qemu
$ qemu-img convert -O qcow2 AlmaLinux-9-GenericCloud-latest.x86_64.qcow2 \
    AlmaLinux-9-GenericCloud-latest.x86_64-uncompressed.qcow2
```

## OpenBSD VM Networking with Built-in Local Interfaces

The simplest and most straightforward method for setting up networking in OpenBSD virtual machines uses the built-in local interfaces via the `-L` flag in [`vmctl(8)`](https://man.openbsd.org/vmctl). This approach makes daily use and configuration hassle-free.

When a local interface is assigned to a VM:

- A `vio(4)` interface is created inside the VM.
- A matching `tap(4)` interface is created on the host.
- The `vmd(8)` daemon automatically:
  - Generates an IPv4 subnet for the interface.
  - Configures a gateway address on the host.
  - Runs a simple DHCP/BOOTP server to provide network settings to the VM.

To complete the setup, add the following rules to your `/etc/pf.conf` file. 
These enable NAT for the VM subnet and redirect DNS requests to your chosen DNS server.


```pf
pass out on egress from 100.64.0.0/10 to any nat-to (egress)

pass in proto { udp tcp } from 100.64.0.0/10 to any port domain \
    rdr-to $dns_server port domain
```

Reload firewall rules:

```bash
doas pfctl -Fr -f /etc/pf.conf
```

Enable ip forwarding in the kernel:

```bash
$ doas sysctl net.inet.ip.forwarding=1
```

```text
$ doas echo "net.inet.ip.forwarding=1" >> /etc/sysctl.conf
```


## Booting the VM

```bash
$ doas vmctl start -c -L -r cloud.iso -m 2G \
    -d AlmaLinux-9-GenericCloud-latest.x86_64-uncompressed.qcow2 "AlmaLinux-9"
```
## Connecting to VM

At this point, SSH access to the VM is enabled. The generic cloud image provisions a default almalinux user, 
with the SSH public key automatically added from the user-data during cloud-init for secure key-based authentication.

```plain
$ ssh almalinux@100.64.x.x -i .ssh/your_pub_key
```


## Build / Install vmm_clock kernel module

This Linux kernel module, written by dv@openbsd.org, 
implements a kvmclock-derived clocksource for Linux guests running under OpenBSD’s hypervisor.

Builds and works fine for this setup. No clock drift occurs when resuming from an extended zzz(8) suspend on laptops.

```bash
$ git clone https://github.com/voutilad/vmm_clock
$ cd vmm_clock && make && sudo make install
$ sudo modprobe vmm_clock
$ sudo echo "vmm_clock" >>  /etc/systemd/network/20-enp0s2.network
```
