---
title: "基本介绍"
draft: false
weight: 1
---

Trojan-Go的核心部分有

- tunnel 各个协议具体实现

- proxy 代理核心

- config 配置注册和解析模块

- redirector 主动检测欺骗模块

- statistics 用户认证和统计模块

可以在对应文件夹中找到相关源代码。

## tunnel.Tunnel隧道

Trojan-Go将所有协议（包括路由功能等）抽象为隧道(tunnel.Tunnel接口)，每个隧道可开启服务端（tunnel.Server接口）和客户端（tunnel.Client）。每个服务端可以从其底层隧道中，剥离并接受流（tunnel.Conn）和包（tunnel.PacketConn)。客户端可以向底层隧道，创建流和包。

每个隧道并不关心其下方的隧道是什么，但是每个隧道清楚知道这个它上方的其他隧道的相关信息。

所有隧道需要下层提供流或包传输支持，或两者都要求提供。所有隧道必须向上层隧道提供流传输支持，但不一定提供包传输。

隧道可能只有服务端，也可能只有客户端，也可能两者皆有。两者皆有的隧道，可被用于作为Trojan-Go客户端和服务端间的传输隧道。

注意，请区分Trojan-Go的服务端/客户端，和隧道的服务端/客户端的区别。下面是一个方便理解的图例。

```text

  入站                              GFW                                  出站
-------->隧道A服务端->隧道B客户端 ----------------> 隧道B服务端->隧道C客户端----------->
           (Trojan-Go客户端)                          (Trojan-Go服务端)

```

最底层的隧道为传输层，即不从其他隧道获取或者创建流和包的隧道，充当上图中隧道A或者C的角色。

- transport，TLS和可插拔传输层

- socks，socks5代理，仅隧道服务端
  
- tproxy，透明代理，仅隧道服务端

- dokodemo，反向代理，仅隧道服务端

- raw，原始TCP/UDP

这几个隧道直接从TCP/UDP Socket创建流和包，不接受为其底层添加的任何隧道。

其他隧道，只要下层能满足上层对包和流传输的需求，则原则上可以任何方式，任何数量进行组合和堆叠。这些隧道在上图中充当隧道B的角色，他们有

- trojan

- websocket

- mux

- simplesocks

- router，路由功能，仅隧道客户端

他们都不关心其下层隧道实现。但可以根据到来的流和包，将其分发给上层隧道。

# proxy.Proxy代理核心

代理核心的作用，监听上述隧道进行组合堆叠并形成的协议栈，将所有的入站协议栈（多个的隧道Server）中抽取的流和包，以及对应元信息，转送给出站协议栈（一个隧道Client）。

注意，这里的入站协议栈可以有多个，如客户端可以同时从Socks5和HTTP协议栈中抽取流和包，服务端可以同时从Websocket承载的Trojan协议，和TLS承载的Trojan协议中抽取流和包等。但是出站协议栈只能有一个。

为了描述入站协议栈（隧道服务端）的组合和堆叠方式，使用一棵多叉树对所有协议栈进行描述。你可以在client/forward/nat/server中看到构建树的过程。

而出站协议栈则比较简单，使用一个简单列表即可描述。
