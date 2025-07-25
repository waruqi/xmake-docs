---
title: Xmake v2.7.3 发布，包组件和 C++ 模块增量构建支持
tags: [xmake, lua, C/C++, package, components]
date: 2022-11-08
author: Ruki
---

## 新特性介绍

### 包组件支持

#### 背景简介

这个新特性主要用于实现从一个 C/C++ 包中集成特定的子库，一般用于一些比较大的包中的库组件集成。

因为这种包里面提供了很多的子库，但不是每个子库用户都需要，全部链接反而有可能会出问题。

尽管，之前的版本也能够支持子库选择的特性，例如：

```lua
add_requires("sfml~foo", {configs = {graphics = true, window = true}})
add_requires("sfml~bar", {configs = {network = true}})

target("foo")
    set_kind("binary")
    add_packages("sfml~foo")

target("bar")
    set_kind("binary")
    add_packages("sfml~bar")
```

这是通过每个包的自定义配置来实现的，但这种方式会存在一些问题：

1. `sfml~foo` 和 `sfml~bar` 会作为两个独立的包，重复安装，占用双倍的磁盘空间
2. 也会重复编译一些共用代码，影响安装效率
3. 如果一个目标同时依赖了 `sfml~foo` 和 `sfml~bar`，会存在链接冲突

如果是对于 boost 这种超大包的集成，重复编译和磁盘占用的影响会非常大，如果在子库组合非常多的情况下，甚至会导致超过 N 倍的磁盘占用。

为了解决这个问题，Xmake 新增了包组件模式，它提供了以下一些好处：

1. 仅仅一次编译安装，任意多个组件快速集成，极大提升安装效率，减少磁盘占用
2. 组件抽象化，跨编译器和平台，用户不需要关心如何配置每个子库之间链接顺序依赖
3. 使用更加方便

更多背景详情见：[#2636](https://github.com/xmake-io/xmake/issues/2636)

#### 使用包组件

对于用户，使用包组件是非常方便的，因为用户是不需要维护包的，只要使用的包，它配置了相关的组件集，我们就可以快速集成和使用它，例如：

```lua
add_requires("sfml")

target("foo")
    set_kind("binary")
    add_packages("sfml", {components = "graphics"})

target("bar")
    set_kind("binary")
    add_packages("sfml", {components = "network"})
```







#### 查看包组件

那么，如何知道指定的包提供了哪些组件呢？我们可以通过执行下面的命令查看：

```bash
$ xrepo info sfml
The package info of project:
    require(sfml):
      -> description: Simple and Fast Multimedia Library
      -> version: 2.5.1
      ...
      -> components:
         -> system:
         -> graphics: system, window
         -> window: system
         -> audio: system
         -> network: system
```

#### 包组件配置

如果你是包的维护者，想要将一个包增加组件支持，那么需要通过下面两个接口来完成包组件的配置：

- add_components: 添加包组件列表
- on_component: 配置每个包组件

##### 包组件的链接配置

大多数情况下，包组件只需要配置它自己的一些子链接信息，例如：

```lua
package("sfml")
    add_components("graphics")
    add_components("audio", "network", "window")
    add_components("system")

    on_component("graphics", function (package, component)
        local e = package:config("shared") and "" or "-s"
        component:add("links", "sfml-graphics" .. e)
        if package:is_plat("windows", "mingw") and not package:config("shared") then
            component:add("links", "freetype")
            component:add("syslinks", "opengl32", "gdi32", "user32", "advapi32")
        end
    end)

    on_component("window", function (package, component)
        local e = package:config("shared") and "" or "-s"
        component:add("links", "sfml-window" .. e)
        if package:is_plat("windows", "mingw") and not package:config("shared") then
            component:add("syslinks", "opengl32", "gdi32", "user32", "advapi32")
        end
    end)

    ...
```

上面是一个不完整的包配置，我仅仅摘取一部分跟包组件相关的配置。

一个关于包组件的配置和使用的完整例子见：[components example](https://github.com/xmake-io/xmake/blob/master/tests/projects/package/components/xmake.lua)

##### 配置组件的编译信息

我们不仅可以配置每个组件的链接信息，还有 includedirs, defines 等等编译信息，我们也可以对每个组件单独配置。

```lua
package("sfml")
    on_component("graphics", function (package, component)
        package:add("defines", "TEST")
    end)
```

##### 配置组件依赖

```lua
package("sfml")
    add_components("graphics")
    add_components("audio", "network", "window")
    add_components("system")

    on_component("graphics", function (package, component)
          component:add("deps", "window", "system")
    end)
```

上面的配置，告诉包，我们的 graphics 组件还会额外依赖 `window` 和 `system` 两个组件。

因此，在用户端，我们对 graphics 的组件使用，可以从

```lua
add_packages("sfml", {components = {"graphics", "window", "system"})
```

简化为：

```lua
add_packages("sfml", {components = "graphics")
```

因为，只要我们开启了 graphics 组件，它也会自动启用依赖的 window 和 system 组件，并且自动保证链接顺序正确。

另外，我们也可以通过 `add_components("graphics", {deps = {"window", "system"}})` 来配置组件依赖关系。

##### 从系统库中查找组件

我们知道，在包配置中，配置 `add_extsources` 可以改进包在系统中的查找，比如从 apt/pacman 等系统包管理器中找库。

当然，我们也可以让每个组件也能通过 `extsources` 配置，去优先从系统库中找到它们。

例如，sfml 包，它在 homebrew 中其实也是组件化的，我们完全可以让包从系统库中，找到对应的每个组件，而不需要每次源码安装它们。

```bash
$ ls -l /usr/local/opt/sfml/lib/pkgconfig
-r--r--r--  1 ruki  admin  317 10 19 17:52 sfml-all.pc
-r--r--r--  1 ruki  admin  534 10 19 17:52 sfml-audio.pc
-r--r--r--  1 ruki  admin  609 10 19 17:52 sfml-graphics.pc
-r--r--r--  1 ruki  admin  327 10 19 17:52 sfml-network.pc
-r--r--r--  1 ruki  admin  302 10 19 17:52 sfml-system.pc
-r--r--r--  1 ruki  admin  562 10 19 17:52 sfml-window.pc
```

我们只需要，对每个组件配置它的 extsources：

```lua
if is_plat("macosx") then
    add_extsources("brew::sfml/sfml-all")
end

on_component("graphics", function (package, component)
    -- ...
    component:add("extsources", "brew::sfml/sfml-graphics")
end)
```

##### 默认的全局组件配置

除了通过指定组件名的方式，配置特定组件，如果我们没有指定组件名，默认就是全局配置所有组件。

```lua
package("sfml")
    on_component(function (package, component)
        -- configure all components
    end)
```

当然，我们也可以通过下面的方式，指定配置 graphics 组件，剩下的组件通过默认的全局配置接口进行配置：

```lua
package("sfml")
    add_components("graphics")
    add_components("audio", "network", "window")
    add_components("system")

    on_component("graphics", function (package, component)
        -- configure graphics
    end)

    on_component(function (package, component)
        -- component audio, network, window, system
    end)
```

### C++ 模块构建改进

#### 增量构建支持

原本以为 Xmake 对 C++ 模块已经支持的比较完善了，后来才发现，它的增量编译还无法正常工作。

因此，这个版本 Xmake 对 C++ 模块的增量编译也做了很好的支持，尽管支持过程还是花了很多精力的。

我分析了下，各家的编译器对生成带模块的 include 依赖信息格式（`*.d`），差异还是非常大的。

gcc 的格式最复杂，不过我还是将它支持上了。

```
build/.objs/dependence/linux/x86_64/release/src/foo.mpp.o: src/foo.mpp\
build/.objs/dependence/linux/x86_64/release/src/foo.mpp.o  gcm.cache/foo.gcm: bar.c++m cat.c++m\
foo.c++m: gcm.cache/foo.gcm\
.PHONY: foo.c++m\
gcm.cache/foo.gcm:|  build/.objs/dependence/linux/x86_64/release/src/foo.mpp.o\
CXX_IMPORTS += bar.c++m cat.c++m\
```

clang 的格式兼容性最好，没有做任何特殊改动就支持了。

```
build//hello.pcm:   /usr/lib/llvm-15/lib/clang/15.0.2/include/module.modulemap   src/hello.mpp\
```

msvc 的格式扩展性比较好，解析和支持起来比较方便：

```
{
    "Version": "1.2",
    "Data": {
        "Source": "c:\users\ruki\desktop\user_headerunit\src\main.cpp",
        "ProvidedModule": "",
        "Includes": [],
        "ImportedModules": [
            {
                "Name": "hello",
                "BMI": "c:\users\ruki\desktop\user_headerunit\src\hello.ifc"
            }
        ],
        "ImportedHeaderUnits": [
            {
                "Header": "c:\users\ruki\desktop\user_headerunit\src\header.hpp",
                "BMI": "c:\users\ruki\desktop\user_headerunit\src\header.hpp.ifc"
            }
        ]
    }
}
```

#### 循环依赖检测支持

由于模块之间是存在依赖关系的，因此如果有几个模块之间存在循环依赖引用，那么是无法编译通过的。

但是之前的版本中，Xmake 无法检测到这种情况，遇到循环依赖，编译就会卡死，没有任何提示信息，这对用户非常不友好。

而新版本中，我们对这种情况做了改进，增加了模块的循环依赖检测，编译时候会出现以下错误提示，方便用户定位问题：

```bash
$ xmake
[  0%]: generating.cxx.module.deps Foo.mpp
[  0%]: generating.cxx.module.deps Foo2.mpp
[  0%]: generating.cxx.module.deps Foo3.mpp
[  0%]: generating.cxx.module.deps main.cpp
error: circular modules dependency(Foo2, Foo, Foo3, Foo2) detected!
  -> module(Foo2) in Foo2.mpp
  -> module(Foo) in Foo.mpp
  -> module(Foo3) in Foo3.mpp
  -> module(Foo2) in Foo2.mpp
```

### 更加 LSP 友好的语法格式

我们默认约定的域配置语法，尽管非常简洁，但是对自动格式化缩进和 IDE 不是很友好，如果你格式化配置，缩进就完全错位了。

```lua
target("foo")
    set_kind("binary")
    add_files("src/*.cpp")
```

另外，如果两个 target 之间配置了一些全局的配置，那么它不能自动结束当前 target 作用域，用户需要显式调用 `target_end()`。

```lua
target("foo")
    set_kind("binary")
    add_files("src/*.cpp")
target_end()

add_defines("ROOT")

target("bar")
    set_kind("binary")
    add_files("src/*.cpp")
```

虽然，上面我们提到，可以使用 `do end` 模式来解决自动缩进问题，但是需要 `target_end()` 的问题还是存在。

```lua
target("foo") do
    set_kind("binary")
    add_files("src/*.cpp")
end
target_end()

add_defines("ROOT")

target("bar") do
    set_kind("binary")
    add_files("src/*.cpp")
end
```

因此，在新版本中，我们提供了一种更好的可选域配置语法，来解决自动缩进，target 域隔离问题，例如：

```lua
target("foo", function ()
    set_kind("binary")
    add_files("src/*.cpp")
end)

add_defines("ROOT")

target("bar", function ()
    set_kind("binary")
    add_files("src/*.cpp")
end)
```

foo 和 bar 两个域是完全隔离的，我们即使在它们中间配置其他设置，也不会影响它们，另外，它还对 LSP 非常友好，即使一键格式化，也不会导致缩进混乱。

注：这仅仅只是一只可选的扩展语法，现有的配置语法还是完全支持的，用户可以根据自己的需求喜好，来选择合适的配置语法。

### 为特定编译器添加 flags

使用 `add_cflags`, `add_cxxflags` 等接口配置的值，通常都是跟编译器相关的，尽管 Xmake 也提供了自动检测和映射机制，
即使设置了当前编译器不支持的 flags，Xmake 也能够自动忽略它，但是还是会有警告提示。

新版本中，我们改进了所有 flags 添加接口，可以仅仅对特定编译器指定 flags，来避免额外的警告，例如：

```lua
add_cxxflags("clang::-stdlib=libc++")
add_cxxflags("gcc::-stdlib=libc++")
```

或者：

```lua
add_cxxflags("-stdlib=libc++", {tools = "clang"})
add_cxxflags("-stdlib=libc++", {tools = "gcc"})
```

注：不仅仅是编译flags，对 add_ldflags 等链接 flags，也是同样生效的。

### renderdoc 调试器支持

感谢 [@SirLynix](https://github.com/SirLynix) 贡献了这个很棒的特性，它可以让 Xmake 直接加载 renderdoc 去调试一些图形渲染程序。

使用非常简单，我们先确保安装了 renderdoc，然后配置调试器为 renderdoc，加载调试运行：

```bash
$ xmake f --debugger=renderdoc
$ xmake run -d
```

具体使用效果如下：

<img src="/assets/img/posts/xmake/renderdoc.gif">

### 新增 C++ 异常接口配置

Xmake 新增了一个 `set_exceptions` 抽象化配置接口，我们可以通过这个配置，配置启用和禁用 C++/Objc 的异常。

通常，如果我们通过 add_cxxflags 接口去配置它们，需要根据不同的平台，编译器分别处理它们，非常繁琐。

例如：

```lua
on_config(function (target)
    if (target:has_tool("cxx", "cl")) then
        target:add("cxflags", "/EHsc", {force = true})
        target:add("defines", "_HAS_EXCEPTIONS=1", {force = true})
    elseif(target:has_tool("cxx", "clang") or target:has_tool("cxx", "clang-cl")) then
        target:add("cxflags", "-fexceptions", {force = true})
        target:add("cxflags", "-fcxx-exceptions", {force = true})
    end
end)
```

而通过这个接口，我们就可以抽象化成编译器无关的方式去配置它们。

开启 C++ 异常:

```lua
set_exceptions("cxx")
```

禁用 C++ 异常:

```lua
set_exceptions("no-cxx")
```

我们也可以同时配置开启 objc 异常。

```lua
set_exceptions("cxx", "objc")
```

或者禁用它们。

```lua
set_exceptions("no-cxx", "no-objc")
```

Xmake 会在内部自动根据不同的编译器，去适配对应的 flags。

### 支持 ispc 编译规则

Xmake 新增了 ipsc 编译器内置规则支持，非常感谢 [@star-hengxing](https://github.com/star-hengxing) 的贡献，具体使用方式如下：

```lua
target("test")
    set_kind("binary")
    add_rules("utils.ispc", {header_extension = "_ispc.h"})
    set_values("ispc.flags", "--target=host")
    add_files("src/*.ispc")
    add_files("src/*.cpp")
```

### 支持 msvc 的 armasm 编译器

之前的版本，Xmake 增加了 Windows ARM 的初步支持，但是对 asm 编译还没有很好的支持，因此这个版本，我们继续完善 Windows ARM 的支持。

对 msvc 的 `armasm.exe` 和 `armasm64.exe` 都支持上了。

另外，我们也改进了包对 Windows ARM 平台的交叉编译支持。

### 新增 gnu-rm 构建规则

Xmake 也新增了一个使用 gnu-rm 工具链去构建嵌入式项目的规则和例子工程，非常感谢 [@JacobPeng](https://github.com/JacobPeng) 的贡献。

```lua
add_rules("mode.debug", "mode.release")

add_requires("gnu-rm")
set_toolchains("@gnu-rm")
set_plat("cross")
set_arch("armv7")

target("foo")
    add_rules("gnu-rm.static")
    add_files("src/foo/*.c")

target("hello")
    add_deps("foo")
    add_rules("gnu-rm.binary")
    add_files("src/*.c", "src/*.S")
    add_files("src/*.ld")
    add_includedirs("src/lib/cmsis")
```

完整工程见：[Embed GNU-RM Example](https://github.com/xmake-io/xmake/blob/master/tests/projects/embed/gnu-rm/hello/xmake.lua)

### 新增 OpenBSD 系统支持

之前的版本，Xmake 仅仅支持 FreeBSD 系统，而 OpenBSD 跟 FreeBSD 还是有不少差异的，导致 Xmake 无法在它上面正常编译安装。

而新版本已经完全支持在 OpenBSD 上运行 Xmake 了。

## 更新内容

### 新特性

* 一种新的可选域配置语法，对 LSP 友好，并且支持域隔离。
* [#2944](https://github.com/xmake-io/xmake/issues/2944): 为嵌入式工程添加 `gnu-rm.binary` 和 `gnu-rm.static` 规则和测试工程
* [#2636](https://github.com/xmake-io/xmake/issues/2636): 支持包组件
* 支持 msvc 的 armasm/armasm64
* [#3023](https://github.com/xmake-io/xmake/pull/3023): 改进 xmake run -d，添加 renderdoc 调试器支持
* [#3022](https://github.com/xmake-io/xmake/issues/3022): 为特定编译器添加 flags
* [#3025](https://github.com/xmake-io/xmake/pull/3025): 新增 C++ 异常接口配置
* [#3017](https://github.com/xmake-io/xmake/pull/3017): 支持 ispc 编译器规则

### 改进

* [#2925](https://github.com/xmake-io/xmake/issues/2925): 改进 doxygen 插件
* [#2948](https://github.com/xmake-io/xmake/issues/2948): 支持 OpenBSD
* 添加 `xmake g --insecure-ssl=y` 配置选项去禁用 ssl 证书检测
* [#2971](https://github.com/xmake-io/xmake/pull/2971): 使 vs/vsxmake 工程生成的结果每次保持一致
* [#3000](https://github.com/xmake-io/xmake/issues/3000): 改进 C++ 模块构建支持，实现增量编译支持
* [#3016](https://github.com/xmake-io/xmake/pull/3016): 改进 clang/msvc 去更好地支持 std 模块

### Bugs 修复

* [#2949](https://github.com/xmake-io/xmake/issues/2949): 修复 vs 分组
* [#2952](https://github.com/xmake-io/xmake/issues/2952): 修复 armlink 处理长命令失败问题
* [#2954](https://github.com/xmake-io/xmake/issues/2954): 修复 c++ module partitions 路径无效问题
* [#3033](https://github.com/xmake-io/xmake/issues/3033): 探测循环模块依赖
