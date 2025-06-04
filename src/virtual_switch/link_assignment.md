# Link assignment

According to [this article](https://medium.com/sonic-nos/creating-a-sonic-nos-virtual-lab-5a9ec431e0d0), sonic will always take `eth0` to be the management port,
and subsequent ports are mapped to SONiC logical ports. ("front-panel")

So e.g. `eth1` will be mapped to SoNIC `Ethernet0`, `eth2` to SoNIC `Ethernet4` and so on.

If you're system is deployed, assuming the links are up in both ends, you should see something like this:

```
root@leaf-0:/# show interfaces status
/bin/sh: 1: sudo: not found
  Interface            Lanes    Speed    MTU    FEC           Alias    Vlan    Oper    Admin    Type    Asym PFC
-----------  ---------------  -------  -----  -----  --------------  ------  ------  -------  ------  ----------
  Ethernet0      25,26,27,28      10G   9100    N/A    fortyGigE0/0  routed      up       up     N/A         N/A
```

## In Linux

When SONiC-VS comes up, it creates one `tap` interface for each logical interface that the switch knows about.
These interfaces are called `Ethernet0`, `Ethernet4`, `Ethernet8`, and so on, both in Linux *and* SONiC. [^1]

```
root@leaf-0:/# ip -c l
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: Ethernet0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9100 qdisc mq state UNKNOWN mode DEFAULT group default qlen 1000
    link/ether 02:42:ac:14:14:04 brd ff:ff:ff:ff:ff:ff
```

When the `vslib` implementation discovers an `ethX` interface, it will internally map it to a SONiC interface.

It is possible to see this mapping in the log:
```log
root@spine-0:/var/log# grep vs_get_veth_name /var/log/syslog
Jun  4 10:13:59.874392 spine-0 NOTICE #syncd: :- vs_get_veth_name: using eth1 instead of vEthernet0
Jun  4 10:13:59.876242 spine-0 NOTICE #syncd: :- vs_get_veth_name: using eth1 instead of vEthernet0
Jun  4 10:14:00.118259 spine-0 NOTICE #syncd: :- vs_get_veth_name: using eth2 instead of vEthernet4
Jun  4 10:14:00.120009 spine-0 NOTICE #syncd: :- vs_get_veth_name: using eth2 instead of vEthernet4
```

`vslib` then opens 2 threads, one for copying ethernet-frames from the `tap` to the "real" interface, and one
for the other way around. [^2]


[^1]: As opposed to DNOS, where DNOS interface names are longer than the maximum Linux allows, and are minimized.
[^2]: This happens in `src/sonic-sairedis/vslib/HostInterfaceInfo.cpp`.