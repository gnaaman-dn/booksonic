# Adding a new container to SONiC

This How-To guide will explain in-detail how to add a new Docker container using
the SONiC Build-System.

We're going to add a new "DriveNets Daemon" container, called DND.

When actually adding a container, consider copy-pasting files from the work-tree instead of from here.
All examples file are generalized and annotated versions of the SNMP container.

## Table of Contents
<!-- toc -->

## Prerequisites:
 - Basic knowledge of Docker containers
 - Basic syntax of the Jinja2 template-engine syntax.
 - Read the overview of the [SONiC Build-System](https://github.com/sonic-net/sonic-buildimage/blob/master/README.buildsystem.md) (which will be referred to as SONiC-BS)

## Executive Summary (tl;dr)

The SONiC-BS includes all `*.mk` files in the `rules` directory.
When a rule file wants to define a Docker-image target, it will add its to one of several global lists (e.g. `SONIC_DOCKER_IMAGES`).
This will trigger the buildsystem to generate a proper target for it, using other properties defined in the rule file.

The rule file defines both dependencies for building the image, and runtime properties for the container itself.

When building the target, SONiC-BS will use Jinja2 to render a `Dockerfile.j2` into a proper Dockerfile before building it.
When building the final image, SONiC-BS will also generate a shell-script for creating the container, and a `systemd` unit for running it.

## Step-by-step
### 1. Creating the rule-file

Rule files reside in the top-level `rules` directory of the `sonic-buildimage` repository.
All `*.mk` are auto-included by `slave.mk`, so all you need to do is create your own file.

Convention states that files containing Docker-Image targets will be called `rules/docker-<IMAGE_NAME>.mk`.
If you're adding both a build-target and a docker-target, they're expected to be in two separate files, e.g.:
 
  - `rules/dnd.mk` for building the code into an executable
  - `rules/docker-dnd.mk` for building the docker-image containing the exectuable

Here's an annotated rule-file:

```makefile
# docker image for DND agent

DOCKER_DND_STEM = docker-dnd
# The target names that we're going to use
DOCKER_DND = $(DOCKER_DND_STEM).gz
DOCKER_DND_DBG = $(DOCKER_DND_STEM)-$(DBG_IMAGE_MARK).gz

# Path to the source path containing Dockerfile.j2.
# `$DOCKERS_PATH` is the `dockers/` directory.
#
# Note that all properties of the target start with the target-name.
$(DOCKER_DND)_PATH = $(DOCKERS_PATH)/docker-dnd

# Base layer used for this image.
# AFAICT, This is used solely in order to create a build-system dependency,
# as the Dockerfile will later need to re-specify this value.
$(DOCKER_DND)_LOAD_DOCKERS += $(DOCKER_CONFIG_ENGINE_BOOKWORM)

# `.deb` Dependencies that should be built before building the Dockerfile.
# This is later used to populate the `docker_dnd_debs` list when rendering the Dockerfile.
$(DOCKER_DND)_DEPENDS += $(LIBNL3_DEV)
$(DOCKER_DND)_DBG_DEPENDS = $($(DOCKER_CONFIG_ENGINE_BOOKWORM)_DBG_DEPENDS)

# `.whl` Dependencies that should be built before building the Dockerfile.
# This is later used to populate the `docker_dnd_whls` list when rendering the Dockerfile.
$(DOCKER_DND)_PYTHON_WHEELS += $(SONIC_PY_COMMON_PY3) $(SONIC_PLATFORM_COMMON_PY3)

# Additional debug tools to install from `apt`
$(DOCKER_DND)_DBG_IMAGE_PACKAGES = $($(DOCKER_CONFIG_ENGINE_BOOKWORM)_DBG_IMAGE_PACKAGES)

# Add depencency on a static-file target.
# AFAICT, This is used solely in order to create a build-system dependency,
# as the Dockerfile will later need to re-specify this value.
$(DOCKER_DND)_FILES += $(SUPERVISOR_PROC_EXIT_LISTENER_SCRIPT)

# Add a file named `monit_dnd` from the `$(DOCKER_DND)_PATH/base_image_files` firectory,
# to the specified directory in the *host*
$(DOCKER_DND)_BASE_IMAGE_FILES += monit_dnd:/etc/monit/conf.d

# Package metadata.
$(DOCKER_DND)_VERSION = 1.0.0
$(DOCKER_DND)_PACKAGE_NAME = dnd

# Adding our image to these global lists causes SONiC-BS to generate targets
SONIC_DOCKER_IMAGES += $(DOCKER_DND)
SONIC_INSTALL_DOCKER_IMAGES += $(DOCKER_DND)

SONIC_DOCKER_DBG_IMAGES += $(DOCKER_DND_DBG)
SONIC_INSTALL_DOCKER_DBG_IMAGES += $(DOCKER_DND_DBG)

# Runtime properties
$(DOCKER_DND)_CONTAINER_NAME = dnd
$(DOCKER_DND)_RUN_OPT += -t
$(DOCKER_DND)_RUN_OPT += -v /etc/sonic:/etc/sonic:ro
$(DOCKER_DND)_RUN_OPT += -v /etc/localtime:/etc/localtime:ro

SONIC_BOOKWORM_DOCKERS += $(DOCKER_DND)
SONIC_BOOKWORM_DBG_DOCKERS += $(DOCKER_DND_DBG)
```

Alongside the `.mk` file with instruction for how to build your image, it's recommended that you create a `.dep` file with [caching instructions](./caching_framework.md):

```Makefile
# Local variables
DPATH       := $($(DOCKER_DND)_PATH)
DEP_FILES   := $(SONIC_COMMON_FILES_LIST) rules/docker-dnd.mk rules/docker-dnd.dep
DEP_FILES   += $(SONIC_COMMON_BASE_FILES_LIST)
DEP_FILES   += $(shell git ls-files $(DPATH))

# Image properties
$(DOCKER_DND)_CACHE_MODE  := GIT_CONTENT_SHA
$(DOCKER_DND)_DEP_FLAGS   := $(SONIC_COMMON_FLAGS_LIST)
$(DOCKER_DND)_DEP_FILES   := $(DEP_FILES)

# This line aligns the debug docker image to follow the regular docker image.
# This syncs the three properties in this file, but also those in the `*.mk` file
$(eval $(call add_dbg_docker,$(DOCKER_DND),$(DOCKER_DND_DBG)))
```

### 2. Creating the image directory

This is what an Image folder might look-like:

```
dockers/docker-dnd
├── base_image_files                 Files for the "base-image" (Host File-System)
│   └── monit_dnd
├── critical_processes              List of Critical-Processes within this container
├── Dockerfile.j2
├── docker-dnd-init.sh              Entrypoint (convention)
├── start.sh                        Supervisor-run initialization one-shot
└── supervisord.conf.j2
```

The SONiC-BS renders the `Dockerfile.j2` file and then builds it into a Docker-image, saving the result in `target/docker-dnd.gz`.
Other `*.j2` files aren't used/rendered by the build-system, but instead they are rendered at runtime using the `sonic-cfggen` utility.

Example of a Dockerfile:

```Dockerfile
{% from "dockers/dockerfile-macros.j2" import install_debian_packages, install_python3_wheels, copy_files %}
FROM docker-config-engine-bookworm-{{DOCKER_USERNAME}}:{{DOCKER_USERTAG}}

ARG docker_container_name
ARG image_version

# Enable -O for all Python calls
ENV PYTHONOPTIMIZE 1

# Make apt-get non-interactive
ENV DEBIAN_FRONTEND=noninteractive

# Pass the image_version to container
ENV IMAGE_VERSION=$image_version

# Copy & Install locally-built Debian packages and implicitly install their dependencies
{% if docker_dnd_debs.strip() -%}
{{ copy_files("debs/", docker_dnd_debs.split(' '), "/debs/") }}
{{ install_debian_packages(docker_dnd_debs.split(' ')) }}
{%- endif %}

# Install dependencies used by some plugins
RUN pip3 install --no-cache-dir pyyaml

# Copy and Install locally-built Python wheel dependencies
{% if docker_dnd_whls.strip() -%}
{{ copy_files("python-wheels/", docker_dnd_whls.split(' '), "/python-wheels/") }}
{{ install_python3_wheels(docker_dnd_whls.split(' ')) }}
{% endif %}

# Clean up (Exact steps should differ based on your Dockerfile)
RUN apt-get clean -y        && \
    apt-get autoclean -y    && \
    apt-get autoremove -y --purge && \
    find / | grep -E "__pycache__" | xargs rm -rf && \
    rm -rf /debs /python-wheels ~/.cache

COPY ["docker-dnd-init.sh", "/usr/bin/"]
COPY ["start.sh", "/usr/bin/"]
COPY ["*.j2", "/usr/share/sonic/templates/"]
COPY ["files/supervisor-proc-exit-listener", "/usr/bin"]
COPY ["critical_processes", "/etc/supervisor"]

# Although exposing ports is not needed for host net mode, keep it for possible bridge mode
EXPOSE 161/udp 162/udp

RUN chmod +x /usr/bin/docker-dnd-init.sh
ENTRYPOINT ["/usr/bin/docker-dnd-init.sh"]
```

Your container should now be build-able by running `make target/docker-dnd.gz`.

### 3. Connecting the container to the rest of the system

Now that you can build your container, the next step is to hook it up to the rest of the running system.

#### systemd unit file

The first step is adding a systemd-unit file in `files/build_templates/dnd.service.j2`:

```ini
[Unit]
Description=DND container
Requires=config-setup.service
Requisite=swss.service
After=config-setup.service swss.service syncd.service interfaces-config.service
BindsTo=sonic.target
After=sonic.target
StartLimitIntervalSec=1200
StartLimitBurst=3

[Service]
ExecStartPre=/usr/bin/{{docker_container_name}}.sh start
ExecStart=/usr/bin/{{docker_container_name}}.sh wait
ExecStop=/usr/bin/{{docker_container_name}}.sh stop
RestartSec=30
```

This file is rendered during build and copied to the correct place automatically.
Note that it refers to `/usr/bin/dnd.sh`, which is also auto-generated by SONiC-BS.

You can read more about systemd unit-file syntax [here](https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html).

#### Adding a feature
In order to get SONiC to start your container, you need to make it known to the system,
or more specifically, to the `featured` daemon.

The daemon takes the data from the database, which is initialized by `files/build_templates/init_cfg.json.j2`.
Edit this file and look for a `set features =` Jinja2 directive.
You'll see this variable initialized and then appended to with values looking like:

```python
                   ("snmp", "enabled", true, "enabled"),
                   ("swss", "enabled", false, "enabled"),
```

These values correspond to `feature, state, delayed, autorestart`.[^note-1]

Add a line for your service:
```python
                   ("dnd", "enabled", false, "enabled"),
```

## Open-questions

 - What is the difference between `SONIC_DOCKER_IMAGES` and `SONIC_INSTALL_DOCKER_IMAGES`? (aside from what the names hint-at)
 - Why are we adding our target both to a global list and to a debian-version-specific list? (e.g. `SONIC_BOOKWORM_DOCKERS`)

[^note-1]: According to the same j2-file, delayed features are started by a systemd `.timer` service. I have not looked for where this timer is configured or for reasoning.