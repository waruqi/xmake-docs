
# linuxos

The linux system operation module is a built-in module, no need to use [import](/api/scripts/builtin-modules/import) to import,
and its interface can be called directly in the script scope.

## linuxos.name

- Get linux system name

We can also quickly get the view through the following command

```sh
xmake l linuxos.name
```

Some names currently supported are:

- ubuntu
- debian
- archlinux
- manjaro
- linuxmint
- centos
- fedora
- opensuse

## linuxos.version

- Get linux system version

The version returned is the semver semantic version object

```lua
if linux.version():ge("10.0") then
    -- ...
end
```

## linuxos.kernelver

- Get linux system kernel version

What is returned is also a semantic version object, you can also execute `xmake l linuxos.kernelver` to quickly view.
