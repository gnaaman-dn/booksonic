# Insights

## Debug

 - Host file-system does not have `strace`. (Installed in Docker-images when building in Debug mode)

## Health-Check

 - Looks like there's no orchastrator, all HA is local to a container.
 - `show system-health ...` seems to perform an on-demand check on containers and processes
 - `show system-health ...` doesn't seem to be implemented on VS. Missing files:
    - `/usr/share/sonic/device/x86_64-kvm_x86_64-r0/system_health_monitoring_config.json`
    - `/etc/sonic/vs_chassis_metadata.json`
    - Mayber others

## ONIE Image build.

 - Starts with `build_debian.sh`
 - Uses `sonic_debian_extension.sh` which is generated from `files/build_templates/sonic_debian_extension.j2`
    - Seems to perform installations on host image.

## Misc.

 - Config engine might be using Python 2?
 - SONiC VS default config info `device/virtual/x86_64-kvm_x86_64-r0/README.md`
    - It loads something default and annoying, should figure out best-way to turn it off.

<!-- ![Excalidraw example](kroki-excalidraw:../assets/foo.excalidraw) -->