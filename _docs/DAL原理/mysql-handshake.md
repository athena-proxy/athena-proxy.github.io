---
title: MySQL连接协议
permalink: /docs/mysql-handshake/
---

# 连接过程

1. client通过tcp协议连接上server的监听端口建立TCP连接。
2. server发送Handshake给client告知协议版本，分配给client的connectio_id, server的兼容性flag，给client加密密码的盐,加密算法
3. client通过接收的Handshake包中的信息计算要传输给server的client端兼容性flag，登录用户名，使用server端给的盐加密密码的密文，要登录的database等信息拼装为HandshakeResponse包发送给server
4. server接收到HandshakeResponse进行校验，认证成功给client回OK包，认证失败给client回ERR包  

官网链接及最新协议介绍请参考： https://dev.mysql.com/doc/internals/en/connection-phase.html  