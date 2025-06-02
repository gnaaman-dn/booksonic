# Caching Framework

The SONiC Build-system has a caching-system for individual targets
Please refer to the documentation [Makefile.cache](https://github.com/sonic-net/sonic-buildimage/blob/master/Makefile.cache).

The documentation refers to a [rules/template.dep](https://github.com/sonic-net/sonic-buildimage/blob/master/rules/template.dep) as a template.

When you add a new target but don't create a `.dep` file, you will see these warnings:
```
[ DPKG ] Cache is not enabled for YOUR_TARGET_NAME package
```