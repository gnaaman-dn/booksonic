# IXIA

IXIA-C implements the ["Open Traffic Generator" API](https://github.com/open-traffic-generator/models).
There are several clients (Go, Python, CLI, etc.).
My experience is that they are very badly documented, and code examples are often years out of date.

The CLI is fine-ish for creating OTG configration files, but I found it lacking for running the flows and pulling the metrics.
It's also possible to just use `curl`, but some of the `curl` examples are also outdated.

[This](https://redocly.github.io/redoc/?url=https://raw.githubusercontent.com/open-traffic-generator/models/master/artifacts/openapi.yaml&nocors) is the official updated API specification.

IXIA listens on port `8443`, and it's possible to refer to it simply by the container name.

```bash
# For usage in further examples
export OTG_HOST="https://clab-sonic-ixia-0:8443"
```

## CURL

### Uploading a config
```bash
curl -sk "${OTG_HOST}/config" -H "Content-Type: application/json" -d @otg.json
```

Configuration can be created manually or by running `otgen` (See example below).

### Viewing config
```bash
curl -sk "${OTG_HOST}/config"
```

### Running a flow
All flows:
```bash
curl -sk "${OTG_HOST}/control/state" \
    -H  "Content-Type: application/json" \
    -d '{"choice": "traffic", "traffic": {"flow_transmit": {"state": "start"}}}'
```

A specific flow `f1`, given in the configuration:
```bash
curl -sk "${OTG_HOST}/control/state" \
    -H  "Content-Type: application/json" \
    -d '{"choice": "traffic", "traffic": {"flow_transmit": {"state": "start", "flow_names": ["f1"]}}}'
```

### Metrics retrieval

https://redocly.github.io/redoc/?url=https://raw.githubusercontent.com/open-traffic-generator/models/master/artifacts/openapi.yaml&nocors#tag/Monitor/operation/get_metrics
```bash
curl -sk "${OTG_HOST}/monitor/metrics" -H "Content-Type: application/json" -d '{"choice": "flow"}'
```

Example output:
```json
{
  "choice": "flow_metrics",
  "flow_metrics": [
    {
      "name": "f1",
      "transmit": "stopped",
      "frames_tx": "4",
      "frames_rx": "3",
      "bytes_tx": "280",
      "bytes_rx": "210",
      "frames_tx_rate": 2.0000217,
      "frames_rx_rate": 1.3340125,
      "tx_rate_bps": 1120.0122,
      "rx_rate_bps": 747.047
    }
  ]
}
```


## OTGEN
https://otg.dev/clients/otgen/#installation
```bash
bash -c "$(curl -sL https://get.otgcdn.net/otgen)"
```

```bash
otgen create flow --name f1 --tx leaf-0 --txl eth1 --rx leaf-0 --rxl eth1 --ipv4 --proto icmp --dst 10.0.0.1 --count 0 --rate 2 > otg.yml
otgen run --api https://172.20.20.8:8443 --insecure --file otg.yml --log debug
```

The IXIA server doesn't seem to like YML files, only JSON (but I would love to be proven wrong),
so it is necessary to convert the `otgen` output before `curl`-ing it.
```bash
wget https://github.com/mikefarah/yq/releases/download/v4.45.4/yq_linux_amd64 -Oyq
chmod +x ./yq
cat otg.yml | ./yq -p yml -o json > otg.json
```

## SNAPPI
https://github.com/open-traffic-generator/otg-examples/blob/main/clab/ixia-c-b2b/otg.py


``` bash
admin@spine-0:/proc/sys/net$ sudo config vlan add 10
admin@spine-0:/proc/sys/net$ sudo config vlan member add -u 10 Ethernet0
admin@spine-0:/proc/sys/net$ sudo config vlan member add -u 10 Ethernet4
```


sudo config vlan member add 10 Ethernet0
sudo config vlan member add 10 Ethernet4