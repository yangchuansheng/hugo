---
keywords:
- 米开朗基杨
- coredns
- chinadns
- gfw
title: "使用 CoreDNS 来应对 DNS 污染"
subtitle: "在 MacOS 上部署轻量级高性能的 CoreDNS"
description: 在 MacOS 上部署轻量级高性能的 CoreDNS
date: 2019-03-06T13:41:11+08:00
draft: false
toc: true
categories: "gfw"
tags: ["coredns", "gfw"]
img: "https://hugo-picture.oss-cn-beijing.aliyuncs.com/images/DNS.jpg"
bigimg: [{src: "https://hugo-picture.oss-cn-beijing.aliyuncs.com/blog/2019-04-27-080627.jpg"}]
---

[CoreDNS](https://github.com/coredns/coredns) 是新晋的 [CNCF 孵化项目](https://coredns.io/2017/03/02/why-cncf-for-coredns/)，前几天已经从 CNCF 正式毕业，并正式成为 Kubernetes 的 DNS 服务器。CoreDNS 的目标是成为 cloud-native 环境下的 DNS 服务器和服务发现解决方案，即：

> Our goal is to make CoreDNS the cloud-native DNS server and service discovery solution.

它有以下几个特性：

+ 插件化（Plugins）

   基于 Caddy 服务器框架，CoreDNS 实现了一个插件链的架构，将大量应用端的逻辑抽象成 plugin 的形式（如 Kubernetes 的 DNS 服务发现，Prometheus 监控等）暴露给使用者。CoreDNS 以预配置的方式将不同的 plugin 串成一条链，按序执行 plugin 的逻辑。从编译层面，用户选择所需的 plugin 编译到最终的可执行文件中，使得运行效率更高。CoreDNS 采用 Go 编写，所以从具体代码层面来看，每个 plugin 其实都是实现了其定义的 interface 的组件而已。第三方只要按照 CoreDNS Plugin API 去编写自定义插件，就可以很方便地集成于 CoreDNS。

+ 配置简单化

   引入表达力更强的 [DSL](https://www.wikiwand.com/zh/%E9%A2%86%E5%9F%9F%E7%89%B9%E5%AE%9A%E8%AF%AD%E8%A8%80)，即 `Corefile` 形式的配置文件（也是基于 Caddy 框架开发）。

+ 一体化的解决方案

   区别于 `kube-dns`，CoreDNS 编译出来就是一个单独的二进制可执行文件，内置了 cache，backend storage，health check 等功能，无需第三方组件来辅助实现其他功能，从而使得部署更方便，内存管理更为安全。
  
其实从功能角度来看，CoreDNS 更像是一个通用 DNS 方案（类似于 BIND），然后通过插件模式来极大地扩展自身功能，从而可以适用于不同的场景（比如 Kubernetes）。正如官方博客所说：

> CoreDNS is powered by plugins.

## <span id="inline-toc">1.</span> Corefile 介绍

----

`Corefile` 是 CoreDNS 的配置文件（源于 Caddy 框架的配置文件 Caddyfile），它定义了：

+ **`server` 以什么协议监听在哪个端口（可以同时定义多个 server 监听不同端口）**
+ **server 负责哪个 `zone` 的权威（authoritative）DNS 解析**
+ **server 将加载哪些插件**

常见地，一个典型的 Corefile 格式如下所示：

```bash
ZONE:[PORT] {
	[PLUGIN] ...
}
```

+ <span id=inline-purple>ZONE</span> : 定义 server 负责的 zone，`PORT` 是可选项，默认为 53；
+ <span id=inline-purple>PLUGIN</span> : 定义 server 所要加载的 plugin。每个 plugin 可以有多个参数；

比如：

```bash
. {
    chaos CoreDNS-001
}
```

上述配置文件表达的是：server 负责根域 `.` 的解析，其中 plugin 是 `chaos` 且没有参数。

### 定义 server

一个最简单的配置文件可以为：

```bash
.{}
```

即 server 监听 53 端口并不使用插件。**如果此时在定义其他 server，要保证监听端口不冲突；如果是在原来 server 增加 zone，则要保证 zone 之间不冲突，**如：

```bash
.    {}
.:54 {}
```

另一个 server 运行于 54 端口并负责根域 `.` 的解析。

又如：

```bash
example.org {
    whoami
}
org {
    whoami
}
```

同一个 server 但是负责不同 zone 的解析，有不同插件链。

### 定义 Reverse Zone

跟其他 DNS 服务器类似，Corefile 也可以定义 `Reverse Zone`（反向解析 IP 地址对应的域名）：

```bash
0.0.10.in-addr.arpa {
    whoami
}
```

或者简化版本：

```bash
10.0.0.0/24 {
    whoami
}
```

可以通过 `dig` 进行反向查询：

```bash
$ dig -x 10.0.0.1
```

### 使用不同的通信协议

CoreDNS 除了支持 DNS 协议，也支持 `TLS` 和 `gRPC`，即 [DNS-over-TLS](https://www.wikiwand.com/zh/DNS_over_TLS) 和 DNS-over-gRPC 模式：

```bash
tls://example.org:1443 {
#...
}
```

## <span id="inline-toc">2.</span> 插件的工作模式

----

当 CoreDNS 启动后，它将根据配置文件启动不同 server ，每台 server 都拥有自己的插件链。当有 DNS 请求时，它将依次经历如下 3 步逻辑： 

1. 如果有当前请求的 server 有多个 zone，将采用贪心原则选择最匹配的 zone；
2. 一旦找到匹配的 server，按照 [plugin.cfg](https://github.com/coredns/coredns/blob/master/plugin.cfg) 定义的顺序执行插件链上的插件；
3. 每个插件将判断当前请求是否应该处理，将有以下几种可能：


+ **请求被当前插件处理**

  插件将生成对应的响应并回给客户端，此时请求结束，下一个插件将不会被调用，如 whoami 插件；
  
+ **请求被当前插件以 Fallthrough 形式处理**

  如果请求在该插件处理过程中有可能将跳转至下一个插件，该过程称为 fallthrough，并以关键字 `fallthrough` 来决定是否允许此项操作，例如 host 插件，当查询域名未位于 /etc/hosts，则调用下一个插件；
  
+ **请求在处理过程被携带 Hint**

  请求被插件处理，并在其响应中添加了某些信息（hint）后继续交由下一个插件处理。这些额外的信息将组成对客户端的最终响应，如 `metric` 插件；

## <span id="inline-toc">3.</span> CoreDNS 如何处理 DNS 请求

----

如果 Corefile 为：

```bash
coredns.io:5300 {
    file db.coredns.io
}

example.io:53 {
    log
    errors
    file db.example.io
}

example.net:53 {
    file db.example.net
}

.:53 {
    kubernetes
    proxy . 8.8.8.8
    log
    health
    errors
    cache
}
```

从配置文件来看，我们定义了两个 server（尽管有 4 个区块），分别监听在 `5300` 和 `53` 端口。其逻辑图可如下所示：

{{< figure link="https://hugo-picture.oss-cn-beijing.aliyuncs.com/images/jYHoLN.jpg" >}}

每个进入到某个 server 的请求将按照 `plugin.cfg` 定义顺序执行其已经加载的插件。

从上图，我们需要注意以下几点：

+ 尽管在 `.:53` 配置了 `health` 插件，但是它并为在上面的逻辑图中出现，原因是：该插件并未参与请求相关的逻辑（即并没有在插件链上），只是修改了 server 配置。更一般地，我们可以将插件分为两种：
  + **Normal 插件**：参与请求相关的逻辑，且插入到插件链中；
  + **其他插件**：不参与请求相关的逻辑，也不出现在插件链中，只是用于修改 server 的配置，如 `health`，`tls` 等插件；
  
## <span id="inline-toc">4.</span> 在 MacOS 上部署 CoreDNS

----

既然 CoreDNS 如此优秀，我用它来抵御伟大的防火长城岂不美哉？研究了一圈，发现技术上还是可行的，唯一的一个缺点是不支持使用代理，不过你可以通过 [proxychians-ng](https://github.com/rofl0r/proxychains-ng) 或 [proxifier](https://github.com/yangchuansheng/love-gfw#%E7%95%AA%E5%A4%96%E7%AF%87) 来强制使用代理。下面开始折腾。

### 安装

CoreDNS 是 golang 写的，所以只需要下载对应操作系统的二进制文件，到处拷贝，就可以运行了。

下面统统以 MacOS 为例作讲解。

```bash
$ cd ~/Downloads
$ wget https://github.com/coredns/coredns/releases/download/v1.4.0/coredns_1.4.0_darwin_amd64.tgz
$ tar zxf coredns_1.4.0_darwin_amd64.tgz
$ mv ./coredns /usr/local/bin/
```

这里补充一句，CoreDNS 的二进制版本已经安装了所有的插件（plugins），不需要你自己编译。推荐下载二进制版本。

### 配置

要深入了解 CoreDNS，请查看其[文档](https://coredns.io/manual/toc)，及 [plugins 的介绍](https://coredns.io/plugins/)。下面是我的配置文件：

```bash
$ cat <<EOF > /usr/local/etc/Corefile
. {
  hosts {
    fallthrough
  }

  forward . tls://8.8.8.8 tls://8.8.4.4 {
    tls_servername dns.google
    force_tcp
    max_fails 3
    expire 10s
    health_check 5s
    policy sequential
    except www.baidu.com
  }

  proxy . 117.50.11.11 117.50.22.22 {
    policy round_robin
  }

  cache 120
  reload 6s
  log . "{local}:{port} - {>id} '{type} {class} {name} {proto} {size} {>do} {>bufsize}' {rcode} {>rflags} {rsize} {duration}"
  errors
}
EOF
```

+ **hosts** : `hosts` 是 CoreDNS 的一个 plugin，这一节的意思是加载 `/etc/hosts` 文件里面的解析信息。hosts 在最前面，则如果一个域名在 hosts 文件中存在，则优先使用这个信息返回；
+ **fallthrough** : 如果 `hosts` 中找不到，则进入下一个 plugin 继续。缺少这一个指令，后面的 plugins 配置就无意义了；
+ **forward** ：这是另外一个 plugin。`.` 代表所有域名，后面的 IP 代表上游 DNS 服务器的列表。按照什么顺序溯源，由下面的 policy 指令决定；
+ **tls://8.8.8.8** : 这里表示使用 DNS-over-TLS 协议访问 8.8.8.8。同时还需要通过 `tls_servername` 指定 DNS 名称。
+ **force_tcp** ： 强制使用 TCP 协议溯源。这要求上游 DNS 必须支持 TCP 协议；
+ **expect** ：指定哪些域名不按照本 plugin 配置溯源。这里主要用来排除国内的域名，然后通过下面的 proxy 转到国内的 DNS 进行解析。这里被排除的域名只填了一个 `www.baidu.com`，后面我们再通过脚本填上所有的国内域名；
+ **proxy** : 解析 `forward` 中被排除的域名。`.` 代表所有域名，后面的 IP 代表上游 DNS 服务器的列表，这里我选择的是 [onedns](https://www.onedns.net/)。按照什么顺序溯源，由下面的 policy 指令决定；
+ **cache** : 溯源得到的结果，缓存指定时间。类似 TTL 的概念；
+ **reload** : 多久扫描配置文件一次。如有变更，自动加载；
+ **log** : 打印/存储访问日志。日志格式参考：[https://coredns.io/plugins/log/](https://coredns.io/plugins/log/)；
+ **errors** : 打印/存储错误日志；

讲一下我自己的理解：

1. 配置文件类似于 nginx 配置文件的格式；
2. 最外面一级的大括号，对应『服务』的概念。多个服务可以共用一个端口；
3. 往里面一级的大括号，对应 plugins 的概念，每一个大括号都是一个 plugin。这里可以看出，plugins 是 CoreDNS 的一等公民；
4. 服务之间顺序有无关联没有感觉，但 plugins 之间是严重顺序相关的。某些 plugin 必须用 `fallthrough` 关键字流向下一个 plugin；
5. plugin 内部的配置选项是顺序无关的；
6. 从 [plugins](https://coredns.io/plugins/) 页面的介绍看，CoreDNS 的功能还是很强的，既能轻松从 bind 迁移，还能兼容 old-style dns server 的运维习惯；
7. 从 CoreDNS 的性能指标看，适合做大型服务。

当然了，上面的配置文件还可以升级一下，具体我就不解释了：

```bash
. {
  hosts {
    fallthrough
  }

  forward . 127.0.0.1:5301 127.0.0.1:5302 {
    max_fails 3
    expire 10s
    health_check 5s
    policy sequential
    except www.baidu.com
  }

  proxy . 117.50.11.11 117.50.22.22 {
    policy round_robin
  }

  cache 120
  reload 6s
  log . "{local}:{port} - {>id} '{type} {class} {name} {proto} {size} {>do} {>bufsize}' {rcode} {>rflags} {rsize} {duration}"
  errors
}

.:5301 {
  forward . tls://8.8.8.8 tls://8.8.4.4 {
    tls_servername dns.google
    force_tcp
    max_fails 3
    expire 10s
    health_check 5s
    policy sequential
  }
}

.:5302 {
  forward . tls://1.1.1.1 tls://1.0.0.1 {
    tls_servername 1dot1dot1dot1.cloudflare-dns.com
    force_tcp
    max_fails 3
    expire 10s
    health_check 5s
    policy sequential
  }
}
```

### 定时更新国内域名列表

编写一个 shell 脚本，用来更新 Corefile 中排除的国内域名列表：

```bash
$ brew install gnu-sed
$ cat <<EOF > /usr/local/bin/update_coredns.sh
#!/bin/bash

chinadns=$(curl -sL https://raw.githubusercontent.com/felixonmars/dnsmasq-china-list/master/accelerated-domains.china.conf|awk -F "/" '{print $2}')
touch update_coredns.sed && echo "" > update_coredns.sed
for i in $chinadns; do echo "/except/ s/$/ $i/" >> update_coredns.sed; done
gsed -i "s/\(except\).*/\1/" /usr/local/etc/Corefile
gsed -i -f update_coredns.sed /usr/local/etc/Corefile
EOF
$ sudo chmod +x /usr/local/bin/update_coredns.sh
```

先执行一遍该脚本，更新 Corefile 的配置：

```bash
$ /usr/local/bin/update_coredns.sh
```

然后通过 `Crontab` 制作定时任务，每隔两天下午两点更新域名列表：

```bash
$ crontab -l
0 14 */2 * * /usr/local/bin/update_coredns.sh
```

### 开机自启

MacOS 可以使用 launchctl 来管理服务，它可以控制启动计算机时需要开启的服务，也可以设置定时执行特定任务的脚本，就像 Linux crontab 一样, 通过加装 `*.plist` 文件执行相应命令。Launchd 脚本存储在以下位置, 默认需要自己创建个人的 `LaunchAgents` 目录：

+ `~/Library/LaunchAgents` : 由用户自己定义的任务项
+ `/Library/LaunchAgents` : 由管理员为用户定义的任务项
+ `/Library/LaunchDaemons` : 由管理员定义的守护进程任务项
+ `/System/Library/LaunchAgents` : 由 MacOS 为用户定义的任务项
+ `/System/Library/LaunchDaemons` : 由 MacOS 定义的守护进程任务项

我们选择在 `/Library/LaunchAgents/` 目录下创建 `coredns.plist` 文件，内容如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <key>Label</key>
    <string>coredns</string>
    <key>ProgramArguments</key>
    <array>
      <string>/usr/local/bin/coredns</string>
      <string>-conf</string>
      <string>/usr/local/etc/Corefile</string>
    </array>
    <key>StandardOutPath</key>
    <string>/var/log/coredns.stdout.log</string>
    <key>StandardErrorPath</key>
    <string>/var/log/coredns.stderr.log</string>
    <key>KeepAlive</key>
    <true/>
    <key>RunAtLoad</key>
    <true/>
  </dict>
</plist>
```

设置开机自动启动 coredns：

```bash
$ sudo launchctl load -w /Library/LaunchAgents/coredns.plist
```

查看服务：

```bash
$ sudo launchctl list|grep coredns

61676	0	coredns
```

```bash
$ sudo launchctl list coredns

{
	"StandardOutPath" = "/var/log/coredns.stdout.log";
	"LimitLoadToSessionType" = "System";
	"StandardErrorPath" = "/var/log/coredns.stderr.log";
	"Label" = "coredns";
	"TimeOut" = 30;
	"OnDemand" = false;
	"LastExitStatus" = 0;
	"PID" = 61676;
	"Program" = "/usr/local/bin/coredns";
	"ProgramArguments" = (
		"/usr/local/bin/coredns";
		"-conf";
		"/usr/local/etc/Corefile";
	);
};
```

查看端口号：

```bash
$ sudo ps -ef|egrep -v grep|grep coredns

    0 81819     1   0  2:54下午 ??         0:04.70 /usr/local/bin/coredns -conf /usr/local/etc/Corefile
    
$ sudo lsof -P -p 81819|egrep "TCP|UDP"

coredns 81819 root    5u    IPv6 0x1509853aadbdf853      0t0     TCP *:5302 (LISTEN)
coredns 81819 root    6u    IPv6 0x1509853acd2f39ab      0t0     UDP *:5302
coredns 81819 root    7u    IPv6 0x1509853aadbdc493      0t0     TCP *:53 (LISTEN)
coredns 81819 root    8u    IPv6 0x1509853acd2f5a4b      0t0     UDP *:53
coredns 81819 root    9u    IPv6 0x1509853ac63bfed3      0t0     TCP *:5301 (LISTEN)
coredns 81819 root   10u    IPv6 0x1509853acd2f5d03      0t0     UDP *:5301
```

大功告成，现在你只需要将系统的 DNS IP 设置为 `127.0.0.1` 就可以了。

### 验证

```bash
$ dig www.google.com

; <<>> DiG 9.10.6 <<>> www.google.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 49942
;; flags: qr rd ra; QUERY: 1, ANSWER: 6, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.google.com.			IN	A

;; ANSWER SECTION:
www.google.com.		64	IN	A	108.177.97.147
www.google.com.		64	IN	A	108.177.97.105
www.google.com.		64	IN	A	108.177.97.106
www.google.com.		64	IN	A	108.177.97.103
www.google.com.		64	IN	A	108.177.97.104
www.google.com.		64	IN	A	108.177.97.99

;; Query time: 0 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Wed Mar 06 13:23:31 CST 2019
;; MSG SIZE  rcvd: 223
```

搞定。

## <span id="inline-toc">5.</span> 参考资料

----

+ [CoreDNS 使用与架构分析](https://zhengyinyong.com/coredns-basis.html)
