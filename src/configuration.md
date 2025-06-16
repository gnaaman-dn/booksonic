# Configuration

## Configuring a Management-VRF

By default, the management interface (`eth0`) is in the same VRF as all other inband interfaces.
This fucks-up routing because the management interface usually has a default-route.

It is possible to configure a management-VRF to isolate it.

From CLI:
```shell
sudo config vrf add mgmt
# -or-
sudo config vrf add management
```
Both names are keywords, and do not affect the name of the actual VRF (it's going to be `mgmt` in Linux either way).

In the config-file it looks like this:
```json
{
    "MGMT_VRF_CONFIG": {
        "vrf_global": {
            "mgmtVrfEnabled": "true"
        }
    }
}
```
Note that applying this using a containerlab `startup-config` doesn't work, probably due to a race.

## Switching

 - By default all interfaces are in "routed" switchmode, and frames aren't forwarded without explicit routes.
   - We can switch interfaces to trunk mode and add them as untagged vlan members to enable forwarding. This enslaves them to the `Bridge` interface.
 - "default" router-type is `LeafRouter`, but it can also be a few other options (Like `ToRRouter` or `SpineRouter`)
   - All options are listed in `src/sonic-yang-models/yang-models/sonic-device_metadata.yang` in `leaf type`.
   - Seems to affect mostly BGP/FRR configuration

## Routing

FRR Configurations can be managed in a couple of different ways.
It can either be "unified" or "split".

### Unified
When the configuration is "unified" (which is the default), SONiC will use either `bgpcfgd` (the default) or `frrcfgd` (or `frr-mgmt-framework`) to configure FRR with data
taken from the main CONFIG_DB.

The system can still be configured with `vtysh`, but the changes will not be persistent.

These two config daemons are mutually exclusive and use slightly different configuration syntax.
SONiC can be configured to use `frrcfgd` by adding this config option.
```json
{
    "DEVICE_METADATA": {
        "localhost": {
            "frr_mgmt_framework_config": "true"
        }
    }
}
```
This value is only checked on the startup of the `bgp` container, so a container-restart is in order.

> [!WARNING]
> It is not enough to use `systemctl restart bgp` or `docker restart bgp`.
> The container should be recreated, otherwise you'll spot Jinja2 error when it starts.
>
> You need to remove the container to force the systemd service to recreate it.
>
> This is true even if you deploy a topology with `containerlab`.

Full example:
```bash
sonic-db-cli CONFIG_DB hset 'DEVICE_METADATA|localhost' frr_mgmt_framework_config true
sudo systemctl stop bgp
docker  rm bgp 
sudo systemctl start bgp
```

`bgpcfgd` is much less capable than `frrcfgd`, but the latter unfortunately has *NO* API documentation whatsoever,
and very little examples, forcing you to read the source code and reverse engineer it. (See `src/sonic-frr-mgmt-framework/frrcfgd/frrcfgd.py`)

### Split
When the configuration is split, FRR is in control of its own configuration, but this means that the overall machine configuration is split to 2 files (or more, depending on exact config mode).


Resources:
 - [Don’t use split-mode, use frr-mgmt-framework](https://medium.com/sonic-nos/sonic-dont-use-split-mode-use-frr-mgmt-framework-a67ad76ec1a6)
 - [How FRR is configured in SONiC](https://www.sonicnos.net/content/guides/config-frr)
 - [EVPN Route Reflector with SONiC using frr-mgmt-framework](https://medium.com/sonic-nos/evpn-route-reflector-with-sonic-using-frr-mgmt-framework-db6d12b85ce7)
