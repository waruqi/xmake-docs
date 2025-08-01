# XPack 组件接口 {#xpack-component-interfaces}

## set_title

- 设置包组件的简单描述

```lua
xpack_component("LongPath")
    set_title("Enable Long Path")
```

## set_description

- 设置包组件的详细描述

```lua
xpack_component("LongPath")
    set_description("Increases the maximum path length limit, up to 32,767 characters (before 256).")
```

## set_default

- 设置包组件的默认启用状态

通常包组件都会被默认启用，但是我们也可以使用这个接口，默认禁用这个组件，仅仅当用户安装包时候，选择勾选此组件，才会被启用安装。

```lua
xpack_component("LongPath")
    set_default(false)
    set_title("Enable Long Path")
```

## on_load

- 自定义加载脚本

我们可以在 on_load 自定义脚本域中，进一步灵活的配置包组件。

```lua
xpack_component("test")
    on_load(function (component)
        local package = component:package()
        -- TODO
    end)
```

## before_installcmd

- 添加组件安装之前的脚本

它不会重写整个安装脚本，但是会在现有的安装脚本执行之前，新增一些自定义的安装脚本：

```lua
xpack_component("test")
    before_installcmd(function (component, batchcmds)
        batchcmds:mkdir(package:installdir("resources"))
        batchcmds:cp("src/assets/*.txt", package:installdir("resources"), {rootdir = "src"})
        batchcmds:mkdir(package:installdir("stub"))
    end)
```

需要注意的是，通过 `batchcmds` 添加的 cp, mkdir 等命令都不会被立即执行，而是仅仅生成一个命令列表，后面实际生成包的时候，会将这些命令，翻译成打包命令。

它跟 xpack 的 before_installcmd 使用是完全一样的，唯一的区别是，仅仅当这个组件被启用时候，才会执行这里的安装脚本。

## on_installcmd

- 重写组件的安装脚本

这会重写整个组件的安装脚本，类似 xpack 的 on_installcmd。

```lua
xpack_component("test")
    on_installcmd(function (component, batchcmds)
        -- TODO
    end)
```

## after_installcmd

- 添加组件安装之后的脚本

在组件安装之后，会执行这里的脚本，类似 xpack 的 after_installcmd。

```lua
xpack_component("test")
    after_installcmd(function (component, batchcmds)
        -- TODO
    end)
```

## before_uninstallcmd

- 添加组件卸载之前的脚本

在组件安装之后，会执行这里的脚本，类似 xpack 的 before_uninstallcmd。

```lua
xpack_component("test")
    before_uninstallcmd(function (component, batchcmds)
        -- TODO
    end)
```

## on_uninstallcmd

- 重写组件卸载的脚本

这会重写整个组件的卸载脚本，类似 xpack 的 on_uninstallcmd。

```lua
xpack_component("test")
    on_uninstallcmd(function (component, batchcmds)
        -- TODO
    end)
```

## after_uninstallcmd

- 添加组件卸载之后的脚本

在组件卸载之后，会执行这里的脚本，类似 xpack 的 before_uninstallcmd。

```lua
xpack_component("test")
    before_uninstallcmd(function (component, batchcmds)
        -- TODO
    end)
```

## add_sourcefiles

- 添加组件源文件

这类似于 xpack 的 add_sourcefiles，但这里仅仅当组件被启用后，才会将这些源文件打入安装包。

## add_installfiles

- 添加组件二进制安装文件

这类似于 xpack 的 add_installfiles，但这里仅仅当组件被启用后，才会将这些二进制文件打入安装包。
