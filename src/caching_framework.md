# Caching Framework

The SONiC Build-system has a caching mechanism for individual targets.
Please refer to the documentation [Makefile.cache](https://github.com/sonic-net/sonic-buildimage/blob/master/Makefile.cache).

## Enabling caching for local build

By default, both reading and writing to cache are disabled.
In order to enable it, you'll have to set a couple of environment variables.

It is recommended to add them to your `.bashrc`, `.zshrc`, `.sonic_env`, or whatever other file gets loaded
into your shell when you start building.

The first one is mandatory, and just tells the SONiC Build-System to enable caching:
```bash
export SONIC_DPKG_CACHE_METHOD=rwcache
```

Once this variable is set, SONiC Build-System will try using `/var/cache/sonic/artifacts` by default.
This directory should sit on a file-system with about ~10GiB free space, (Cached VS artifacts take around 7GiB)
but the more the merrier.

You can optionally also set `SONIC_DPKG_CACHE_SOURCE` to a different path.

Ensure that either the default or your chosen directory exists, and that
they have write-permissions:
```bash
sudo chmod -R 0777 ${SONIC_DPKG_CACHE_SOURCE:-/var/cache/sonic/artifacts}
```

## Adding caching rule for your target
The documentation refers to a [rules/template.dep](https://github.com/sonic-net/sonic-buildimage/blob/master/rules/template.dep) as a template.

When you add a new target but don't create a `.dep` file, you will see these warnings:
```
[ DPKG ] Cache is not enabled for YOUR_TARGET_NAME package
```