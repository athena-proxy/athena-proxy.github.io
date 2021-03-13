---
title: C++语言接入DAL
permalink: /docs/dal-cpp/
---

## 现状

MySQL官方提供了2个版本的connector/C++, 1.1和8.0版本。8.0版本面向MySQL 5.7及以后的Server版本，考虑到生产中大部分MySQL 5.6版本的Server,所以推荐使用Connector/C++ 1.1版本。

## 注意事项

- 连接的时候必须指定Schema以及用户名密码，Schema即为申请DAL时返回的DALGroup.
- 不能使用sq::Connection::prepareStatement()服务端prepareStatement