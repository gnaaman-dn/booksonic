# Deploying VS on virsh

SONiC-VS (Virtual Switch) can be built into a KVM-image and deployed as a VM.

The name of the Make target is `target/sonic-vs.img.gz`, which is a compressed QEMU QCOW2 file. (virtual disk image)

The file is compressed for distribution, but it has to be decompressed before usage:
```bash
# This creates a target/sonic-vs.img file, removing the compressed version.
gunzip target/sonic-vs.img.gz
```

For example:
```bash
qemu-system-x86_64 -name sonic-simulator_1 -m 8192M -smp cpus=4 -drive file=target/sonic-vs.img,index=0,media=disk,id=drive0 -serial telnet:127.0.0.1:5001,server,nowait  -net bridge,br=virbr0 -net nic,model=virtio -display none
```

Note that the VM *will* change the data on the virtual disk, which means that multiple machines will have to have different files.
If you're planning to run multiple machines, make a copy.

## Running with virsh
`virsh`/`libvirt` is virtual machine manager.

```bash
# Creating copies for different machines
sudo cp target/sonic-vs.img.gz /var/lib/libvirt/images/sonic-vm-0.img
sudo cp target/sonic-vs.img.gz /var/lib/libvirt/images/sonic-vm-1.img

virt-install -n sonic-0 --ram=8192 --vcpus=4 --disk=/var/lib/libvirt/images/sonic-vm-0.img --graphics none --network bridge:virbr0,address.type=pci,address.bus=0x00,address.slot=0x03 --osinfo linux2022 --import --autoconsole none
virt-install -n sonic-1 --ram=8192 --vcpus=4 --disk=/var/lib/libvirt/images/sonic-vm-1.img --graphics none --network bridge:virbr0,address.type=pci,address.bus=0x00,address.slot=0x03 --osinfo linux2022 --import --autoconsole none
```

Now you should have 2 machines:
```bash
$ virsh list
 Id   Name      State
-------------------------
 6    sonic-0   running
 7    sonic-1   running
```

These machines all have a single management-port which is connected to the `virbr0` bridge interface on the host.
It's probably a better idea to use `--network network:...` so you wouldn't have to create the bridge interface by hand; letting `libvirt` create it.

Note that we're setting the VM-facing PCI address to known values, this will be explained later.

If all went well and the machines are up, you should be able to see their acquired IP addresses:

```bash
$ virsh net-dhcp-leases  --network default
 Expiry Time           MAC address         Protocol   IP address           Hostname   Client ID or DUID
---------------------------------------------------------------------------------------------------------
 2025-06-03 15:02:49   52:54:00:34:b2:18   ipv4       192.168.122.175/24   sonic      -
 2025-06-03 15:02:39   52:54:00:7d:76:73   ipv4       192.168.122.97/24    -          -
```

You can connect to them either via SSH, or using `virsh console`.
By default, the credentials are `admin`/`YourPaSsWoRd`.

# Link assignment
Also refer to [Link Assignment](link_assignment.md)

In order to add more front-panel ports, you can edit the machine definition file with `virsh edit sonic-0` and add more interfaces, e.g.:
```xml
    <interface type='bridge'>
      <source bridge='br_2_to_1'/>
      <model type='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x0'/>
    </interface>

    <interface type='bridge'>
      <source bridge='br_2_ep'/>
      <model type='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x07' function='0x0'/>
    </interface>
```
These can also be, of course, arguments to `virt-install`.

It's important to make sure that the management port has the lowest PCI address, so it would be recognized by the VM-kernel as `eth0`.
By default, if not given any other address, `libvirt` might assign it to bus-1, which only has one slot, since it is the `pcie-root-port`.
Forcing the port to be in bus-0 allows us to assign to a lower PCI address then any other port.

# Topology Example

This example creates 2 SONiCs connected via their respective Ethernet0.
```bash
# Creating copies for different machines
gunzip target/sonic-vs.img.gz
sudo cp target/sonic-vs.img /var/lib/libvirt/images/sonic-vm-0.img
sudo cp target/sonic-vs.img /var/lib/libvirt/images/sonic-vm-1.img

sudo ip link add sonic-0-1 type bridge
sudo ip link set sonic-0-1 up

virt-install -n sonic-0 --ram=8192 --vcpus=4 --disk=/var/lib/libvirt/images/sonic-vm-0.img \
    --graphics none --osinfo linux2022 --import --autoconsole none \
    --network bridge:virbr0,address.type=pci,address.bus=0x00,address.slot=0x03 \
    --network bridge:sonic-0-1,address.type=pci,address.bus=0x00,address.slot=0x05

virt-install -n sonic-1 --ram=8192 --vcpus=4 --disk=/var/lib/libvirt/images/sonic-vm-1.img \
    --graphics none --osinfo linux2022 --import --autoconsole none \
    --network bridge:virbr0,address.type=pci,address.bus=0x00,address.slot=0x03 \
    --network bridge:sonic-0-1,address.type=pci,address.bus=0x00,address.slot=0x05
```

# Memory Backing
If you're on a capable machine with hugepages enabled, you can (and should) add `--memorybacking=hugepages=yes` to use them.

# Working with `isolcpus`
`isolcpus` is a kernel command-line argument (i.e. given to it by the bootloader) telling it to avoid scheduling
a set of cores.
These cores a excluded from almost all host scheduling, which makes them optimal for uninterrupted VM execution.

## Checking if you have `isolcpus` enabled

The command-line arguments given to the kernel are available in `/proc/cmdline`:
```bash
$ cat /proc/cmdline
BOOT_IMAGE=/vmlinuz-5.15.0-141-generic root=/dev/mapper/ubuntu--vg-ubuntu--lv ro console=tty0 console=ttyS0,115200n8 hugepagesz=1G default_hugepagesz=1G hugepages=300 iommu=pt intel_iommu=on isolcpus=35-43,79-87 nohz_full=35-43,79-87 rcu_nocbs=35-43,79-87 nosoftlockup intel_idle.max_cstate=0 intel_pstate=disable audit=0
```

Here you can see `isolcpus=35-43,79-87`, which means that in total 18 CPUs are excluded from scheduling.
(The ranges are inclusive)

## Enabling/Disabling `isolcpus`

If you want to disable core isolation, edit `/etc/default/grub` and find where the `GRUB_CMDLINE_LINUX` is set.
Remove all references to `isolcpus=...`, `nohz_full=...`, and `rcu_nocbs=...`.

If you want to enable isolation, choose some range of cores and set all three-variables to the same range.
The exact semantics of these variables can be found in the [kernel docs](https://docs.kernel.org/admin-guide/kernel-parameters.html).

If your CPU has 2 NUMAs and hyperthreading, it is recommended to isolate both logical cores of the same physical core.
In the example above, we isolate the top 9 physical-cores of NUMA-1 by specifying 2 ranges of logical-cores.
This is derived from this snippet from `lscpu`:
```bash
NUMA:
  NUMA node(s):           2
  NUMA node0 CPU(s):      0-21,44-65
  NUMA node1 CPU(s):      22-43,66-87            <<<<<<
```

## Using isolated CPUs

If you have some set of CPUs that are isolated and you want to run the VM on them, you'll have to explicitly
tell `libvirt` that you want to use them.

Let's say you've isoalted the ranges `35-43,79-87`:
```bash
$ cat /proc/cmdline
BOOT_IMAGE=/vmlinuz-5.15.0-141-generic root=/dev/mapper/ubuntu--vg-ubuntu--lv ro console=tty0 console=ttyS0,115200n8 hugepagesz=1G default_hugepagesz=1G hugepages=300 iommu=pt intel_iommu=on isolcpus=35-43,79-87 nohz_full=35-43,79-87 rcu_nocbs=35-43,79-87 nosoftlockup intel_idle.max_cstate=0 intel_pstate=disable audit=0
```

You can add the following argument to make libvirt pin the vCPUs to the isolated (p)CPUs.
```bash
--cputune vcpupin0.vcpu=0,vcpupin0.cpuset=35,vcpupin1.vcpu=1,vcpupin1.cpuset=36,vcpupin2.vcpu=2,vcpupin2.cpuset=37,vcpupin3.vcpu=3,vcpupin3.cpuset=38
```

Unfortunately, using `--vcpus=4,cpuset=35-38` doesn't seem to be enough, as fully utilizing the cores require the host-kernel scheduler,
which is impossible due to the isolation.
