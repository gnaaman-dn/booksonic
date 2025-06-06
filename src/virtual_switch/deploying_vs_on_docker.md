# Deploying SONiC-VS on Docker

SONiC-VS (Virtual Switch) can be built into a Docker-image and deployed as a container.


The name of the Make target is `target/docker-sonic-vs.gz`, which is an saved Docker image.
It can be loaded by running:

```bash
docker image load < ./target/docker-sonic-vs.gz
```

The container can the be run like any other container, with
```bash
docker run -it docker-sonic-vs:latest
```

The container is created with an `eth0`, which is treated as the management-port, and no other front-panel ports.
It is technically possible to create more ports when running with plain Docker, but it is recommeneded to use [containerlab](./deploying_vs_on_containerlab.md).

> [!IMPORTANT]
>  `docker-sonic` is missing *many* basic features, and a lot of commands are basically useless/broken.
>
> e.g. the command `show lldp neighbors` is internally using `sudo docker exec lldp lldpctl`.
> The `docker-sonic` container is missing `sudo` and `docker`, and it does not have an `lldp` container, nor does it have `lldpctl` or `lldpd`.