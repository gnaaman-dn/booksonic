# CLI Examples

Note that like in DNOS-CLI, the SONiC CLI supports keyword shortening.
The top level-command (e.g. `show` or `config`) are shell-commands, but otherwise all subcommands can be shortened as long as the short-form is unique.

So, `config int sta Ethernet0` is equivalent to `config interface startup Ethernet0`

Since all commands are actually just shell-commands, you might need to execute them with sudo, depending on the deployment.

## Manipulating admin-state

```
# Admin-up
config interface startup Ethernet0

# Admin-down
config interface shutdown Ethernet0
```