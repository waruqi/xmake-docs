---
outline: deep
---

# 编译缓存加速 {#build-cache-acceleration}

## 本地编译缓存 {#local-build-cache}

默认情况下，Xmake 会开启本地缓存。2.6.5 之前的版本默认使用外置的 ccache，而 2.6.6 之后版本，Xmake 提供了内置的跨平台本地缓存方案。

相比 ccache 等第三方独立进程，xmake 内部状态维护，更加便于优化，也避免了频繁的独立进程加载耗时，也避免了与守护进程额外的通信。

另外，内置的缓存能够更好地支持跨平台，Windows 上 msvc 也能够很好地支持，而 ccache 仅仅支持 gcc/clang。

当然，我们也可以通过下面的命令禁用缓存：

```sh
$ xmake f --ccache=n
```

注意：无论是否使用内置本地缓存，配置名都是 `--ccache=`，意思是 c/c++ 构建缓存，而不仅仅是指 ccache 工具的名字。

我们如果想继续使用外置的其他缓存工具，也可以通过下面的方式来配置。

```sh
$ xmake f --ccache=n --cxx="ccache gcc" --cc="ccache gcc"
$ xmake
```

## 远程编译缓存 {#remote-build-cache}

除了本地缓存，我们也提供了远程缓存服务，类似 mozilla 的 sscache，如果只是个人开发，平常不会用到它。

但是，如果是公司内部多人协同开发一个大型项目，仅靠分布式编译和本地缓存是不够的。我们还需要将编译的对象文件缓存到独立的服务器上进行共享。

这样，其他人即使首次编译，也不需要每次都去分布式编译它，直接从远程拉取缓存来加速编译。

另外，Xmake 提供的远程缓存服务，也是全平台支持的，不仅支持 gcc/clang，还支持 msvc。

### 开启服务 {#start-service}

我们可以指定 `--ccache` 参数来开启远程编译缓存服务。当然如果不指定这个参数，xmake 会默认开启所有服务端配置的服务。

```sh
$ xmake service --ccache
<remote_cache_server>: listening 0.0.0.0:9092 ..
```

我们也可以在开启服务的同时，回显详细日志信息。

```sh
$ xmake service --ccache -vD
<remote_cache_server>: listening 0.0.0.0:9092 ..
```

### 以 Daemon 模式开启服务

```sh
$ xmake service --ccache --start
$ xmake service --ccache --restart
$ xmake service --ccache --stop
```

### 配置服务端 {#configure-server}

我们可以运行 `xmake service` 命令，它会自动生成一个默认的 `server.conf` 配置文件，存储到 `~/.xmake/service/server.conf`。

```sh
$ xmake service
generating the config file to /Users/ruki/.xmake/service/server.conf ..
an token(590234653af52e91b9e438ed860f1a2b) is generated, we can use this token to connect service.
generating the config file to /Users/ruki/.xmake/service/client.conf ..
<remote_cache_server>: listening 0.0.0.0:9692 ..
```

然后，我们编辑它，修正服务器的监听端口（可选）。

```sh
$ cat ~/.xmake/service/server.conf
{
    distcc_build = {
        listen = "0.0.0.0:9692",
        workdir = "/Users/ruki/.xmake/service/server/remote_cache"
    },
    known_hosts = { },
    logfile = "/Users/ruki/.xmake/service/server/logs.txt",
    tokens = {
        "590234653af52e91b9e438ed860f1a2b"
    }
}
```

### 配置客户端 {#configure-client}

客户端配置文件在 `~/.xmake/service/client.conf`，我们可以在里面配置客户端需要连接的服务器地址。

我们可以在 hosts 列表中配置多个服务器地址，以及对应的 token。

```sh
$cat ~/.xmake/service/client.conf
{
    remote_cache = {
            connect = "127.0.0.1:9692,
            token = "590234653af52e91b9e438ed860f1a2b"
        }
    }
}
```

#### 配置超时 {#configure-timeout}

默认情况下，客户端连接、收发数据都是无限等待不超时的，但是如果访问服务端的网络不稳定，有可能会导致访问卡死，这时可以配置超时来解决。

如果发生超时异常，就会自动退化到本地缓存，不会永远卡死。

我们可以配置 `send_timeout`、`recv_timeout` 和 `connect_timeout` 三种超时，如果在根节点设置，那么所有客户端服务都会生效。

```sh
$ cat ~/.xmake/service/client.conf
{
    send_timeout = 5000,
    recv_timeout = 5000,
    connect_timeout = 5000
}
```

我们也可以仅针对当前远程缓存服务配置超时，其他服务还是默认超时。

```sh
$ cat ~/.xmake/service/client.conf
{
    distcc_build = {
        send_timeout = 5000,
        recv_timeout = 5000,
        connect_timeout = 5000,
    }
}
```

::: tip 注意
服务端配置同样支持超时配置。
:::

### 用户认证和授权 {#user-authorization}

关于用户认证和授权，可以参考 [远程编译/用户认证和授权](/zh/guide/extras/remote-compilation#user-authorization) 里面的详细说明，用法是完全一致的。

### 连接服务器 {#connect-server}

配置完认证和服务器地址后，就可以输入下面的命令，将当前工程连接到配置的服务器上。

我们需要在连接时，输入 `--ccache`，指定仅连接远程编译缓存服务。

```sh
$ cd projectdir
$ xmake service --connect --ccache
<client>: connect 127.0.0.1:9692 ..
<client>: 127.0.0.1:9692 connected!
```

我们也可以同时连接多个服务，比如分布式编译和远程编译缓存服务。

```sh
$ xmake service --connect --distcc --ccache
```

::: tip 注意
如果不带任何参数，默认连接的是远程编译服务。
:::

### 断开连接 {#disconnect}

```sh
$ xmake service --disconnect --ccache
```

### 清理服务器缓存 {#clean-server-cache}

我们也可以通过下面的命令，清理当前工程对应的远程服务器上的缓存。

```sh
$ xmake service --clean --ccache
```

而如果我们执行 `xmake clean --all`，在连接了远程服务的状态下，也会自动清理所有的缓存。

### 一些内部优化 {#optimizations}

1. 拉取远程缓存的快照，通过 bloom filter + lz4 回传本地后，用于快速判断缓存是否存在，避免频繁查询服务端缓存信息。
2. 配合本地缓存，可以避免频繁地请求远程服务器，拉取缓存。
3. 内部状态维护，相比 sscache 等独立工具，避免了频繁的独立进程加载耗时，也避免了与守护进程额外的通信。
