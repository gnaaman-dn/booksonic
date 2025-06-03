# Deploying VS on virsh

Once you've finished building `target/sonic-vs.img.gz`, you can start the deployment process.
`target/sonic-vs.img.gz` is a compressed QEMU QCOW2 file. (virtual disk image)

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

# Interface assignment
According to [this article](https://medium.com/sonic-nos/creating-a-sonic-nos-virtual-lab-5a9ec431e0d0), sonic will always take `eth0` to be the management port,
and subsequent ports are mapped to SONiC logical ports.

So e.g. `eth1` will be mapped to SoNIC `Ethernet0`, `eth2` to SoNIC `Ethernet4` and so on.

In order to add more ports, you can edit the machine definition file with `virsh edit sonic-0` and add more interfaces, e.g.:
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