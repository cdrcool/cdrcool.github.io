---
title: Redis 安全注意事项
date: 2020-02-19 21:16:00
categories: Redis
---
## 指令安全
Redis 中有一些非常危险的指令，这些指令会对 Redis 的稳定以及数据安全造成非常严重的影响。比如 **keys** 指令会导致 Redis 卡顿，**flushdb** 和 **flushall** 会让 Redis 中的所有数据全部清空。

Redis 在配置文件中提供了 **rename-command** 指令用于将某些危险的指令改成特别的名称，用来避免人为误操作。比如在配置文件的 security 块增加以下内容：
```
rename-command keys abckeyabc
```

如果还想执行 keys 指令，那就不能再敲 keys 指令了，而需要键入 abckeyabc。如果想完全封杀某条命令，可以将指令 rename 成空串，就无法通过任何字符串指令来执行这条指令了。
```
rename-command flushall ""
```

## 端口安全
Redis 默认会监听 6379 端口，如果当前的服务器主机有外网地址，Redis 的服务将会直接暴露再公网上，任何一个初级黑客使用适当的工具对 IP 地址进行端口扫描就可以探测出来。

所以，运维人员必须在 Redis 的配置文件中指定监听的 IP 地址，从而避免这样的惨剧发生。
```
bind 10.100.20.13
```

更进一步，还可以增加 Redis 的密码访问限制，客户端必须使用 auth 指令传入正确的密码才可以访问 Redis，这样即使地址暴露出去了，普通黑客也无法对 Redis 进行任何指令操作。
```
requirepass yoursecurepasswordhereplease
```

密码控制也会影响到从库复制，从库必须在配置文件里使用 masterauth 指令配置相应的密码才可以进行复制操作。
```
masterauth yoursecurepasswordhereplease
```

## Lua 脚本安全
开发者必须禁止 Lua 脚本由用户输入的内容（UGC）生成，这可能会被黑客利用以植入恶意的攻击代码来得到 Redis 的主机权限。

同时，我们应该让 Redis 以普通用户的身份启动，这样即使存在恶意代码黑客也无法拿到 root 权限。

## SSL 代理
Redis 并不支持 SSL 链接，意味着客户端和服务器之间交互的数据不应该直接暴露在公司网上传输，否则会有被窃听的风险。如果必须要用在公网上，可以考虑使用 SSL 代理。

SSL 代理比较常见的有 ssh，不过 Redis 官方推荐使用 spiped 工具，可能是因为 spiped 的功能相对比较单一，使用也比较简单，易于理解。
