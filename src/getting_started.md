# Getting Started

Hallo Allemal.

In order to build the `sonic-buildimage` repository, your machine requires a one time-setup process.

## Machine setup

Ensure these variables are always set:
```bash
# Tell SONiC not to build stuff dependent on old Debian versions
echo 'export NOJESSIE=1 NOSTRETCH=1 NOBUSTER=1 NOBULLSEYE=1' > $HOME/.sonic_env

# Tell SONiC to take third-party images (e.g. debians) from our registry
echo 'export DEFAULT_CONTAINER_REGISTRY=dn19.dev.drivenets.net:5000' >> $HOME/.sonic_env

# Tell SONiC to take SONiC slave images from our registry
echo 'export REGISTRY_SERVER=dn19.dev.drivenets.net REGISTRY_PORT=5000 ENABLE_DOCKER_BASE_PULL=y' >> $HOME/.sonic_env

echo 'source $HOME/.sonic_env' >> ~/.bashrc
# Or .zshrc
```

```bash
# Ubuntu 22.04 Build Dependencies:
sudo apt install -y \
  build-essential \
  python3-pip \
  autoconf automake libtool \
  libssl-dev libxml2-dev zlib1g-dev \
  docker.io

# Ubuntu 22.04 Runtime dependencies
sudo apt install -y redis-server \
  libvirt-clients qemu-kvm libvirt-daemon-system virt-manager bridge-utils

# Add yourself to the docker group to enable docker daemon access
sudo usermod -aG docker $(whoami)

# Enable HTTP access to Amir's registry
echo '{"insecure-registries":["dn19.dev.drivenets.net:5000"]}' | sudo tee /etc/docker/daemon.json
sudo systemctl restart docker

# Used by SONiC's build-system to generate Dockerfiles for slaves.
sudo python3 -m pip install jinjanator

# Logout and login to apply new group and env-variables.
```

> [!NOTE]
> Also, please make sure to set-up [DPKG Caching](./caching_framework.md) at this point.

> [!IMPORTANT]
> Also, please make sure to set-up [DPKG Caching](./caching_framework.md) at this point.

> [!WARNING]
> ALSO, PLEASE MAKE SURE TO SET-UP [DPKG CACHING](./caching_framework.md) AT THIS POINT.

## Repository setup

```bash
git clone https://github.com/sonic-net/sonic-buildimage.git sonic

# One-time(-ish) setup. Initializes submodules etc.
# Might need to be called after checking out.
make init

# One-time(-ish) setup.
# Should be called when changing platform.
make configure PLATFORM=vs

# Build a VM image
make target/docker-sonic-vs.gz
```

## Common Issues

### Permission Denied Error (`/etc/apt/sources.list`)

Problem:
```
mv: cannot move '/etc/apt/sources.list.d/debian.sources' to '/etc/apt/sources.list.d/debian.sources.back': Permission denied
```

Solution:
It is safe to ignore.

### "Unable to locate package" Error

Problem:
```
Checking sonic-slave-base image: sonic-slave-bookworm:a39c8dc6459
Checking sonic-slave-user image: sonic-slave-bookworm-dn:18cc13581bb
E: Unable to locate package libmbedcrypto3
E: Unable to locate package libmbedcrypto3
E: Unable to locate package libmbedx509-0
E: Unable to locate package libmbedx509-0
E: Unable to locate package libmbedtls12
E: Unable to locate package libmbedtls12
SONiC Build System
```

Solution:
Safe to ignore.

Happens as a result of [this PR](https://github.com/sonic-net/sonic-buildimage/pull/22492/files).
Nothing to worry about; just some shoddy workmanship.
