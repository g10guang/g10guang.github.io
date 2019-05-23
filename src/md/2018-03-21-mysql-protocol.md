---
layout: post
title:  "MySQL 协议学习笔记"
date:   2018-03-21 09:00:05 +0800
categories: MySQL
---

采用 tcpdump 可以获得本机网络通过 TCP 协议通信的数据详情，但是无法监听 unix socket，所以本地不能够采用 **localhost**，而应该采用 **127.0.0.1**。

## 编码方式

MySQL 协议中有两种编码方式：
+ 固定长度的编码
+ 第一个字节说明长度，其后跟着指定长度的字节

字符串有两种编码方式：
1. 字符串长度+内容
2. 字符串以(0x00)结束

## 通用请求数据包 Client-->Server

| Type | Name | Description |
| ---- | ---- | ----- |
| int<3> | payload_length | 负载的长度，数据包中除了初始4字节的长度 |
| int<1> | sequence_id | 序列号 |
| string<var> | payload | [len=payload_length] 数据包中的内容 |

Example: `01 00 00 00 01` 代表 `payload_length = 1, sequence_id = 0x00, payload=0x01`

payload_length 由3位无符号整形表示，所以一次最多传输 `2^24-1` 字节。如果需要传输的数据大于等于 `2^24-1` 字节，那么会有额外的数据包发送，直到有一个数据包的长度小于 `2^24-1` 字节。sequence_id 由 0 开始，然后依次自增。当客户端向服务端发送新的命令时，sequence_id 会被重置为 0。

payload 的第一个字节表明客户端携带的命令类型 command-type：

| Hex | Constant Name |
| ---- | ------------ |
| 00 | [COM_SLEEP](https://dev.mysql.com/doc/internals/en/com-sleep.html) |
| 01 | [COM_QUIT](https://dev.mysql.com/doc/internals/en/com-quit.html) |
| 02 | [COM_INIT_DB](https://dev.mysql.com/doc/internals/en/com-init-db.html) |
| 03 | [COM_QUERY](https://dev.mysql.com/doc/internals/en/com-query.html) |
| 04 | [COM_FIELF_LIST](https://dev.mysql.com/doc/internals/en/com-field-list.html) |
| 05 | [COM_CREATE_DB](https://dev.mysql.com/doc/internals/en/com-create-db.html) |
| 06 | [COM_DROP_DB](https://dev.mysql.com/doc/internals/en/com-drop-db.html) |
| 07 | [COM_REFRESH](https://dev.mysql.com/doc/internals/en/com-refresh.html) |
| 08 | [COM_SHUTDOWN](https://dev.mysql.com/doc/internals/en/com-shutdown.html) |
| 09 | [COM_STATISTICS](https://dev.mysql.com/doc/internals/en/com-statistics.html) |
| 0a | [COM_PROCESS_INFO](https://dev.mysql.com/doc/internals/en/com-process-info.html) |
| 0b | [COM_CONNECT](https://dev.mysql.com/doc/internals/en/com-connect.html) |
| 0c | [COM_PROCESS_KILL](https://dev.mysql.com/doc/internals/en/com-process-kill.html) |
| 0d | [COM_DEBUG](https://dev.mysql.com/doc/internals/en/com-debug.html) |
| 0e | [COM_PING](https://dev.mysql.com/doc/internals/en/com-ping.html) |
| 0f | [COM_TIME](https://dev.mysql.com/doc/internals/en/com-time.html) |
| 10 | [COM_DELAYED_INSERT](https://dev.mysql.com/doc/internals/en/com-delayed-insert.html) |
| 11 | [COM_CHANGE_USER](https://dev.mysql.com/doc/internals/en/com-change-user.html) |
| 12 | COM_BINLOG_DUMP |
| 13 | COM_TABLE_DUMP |
| 14 | COM_CONNECT_OUT |
| 15 | COM_REGISTER_SLAVE |
| 16 | [COM_STMT_PREPARE](https://dev.mysql.com/doc/internals/en/com-stmt-prepare.html) |
| 17 | [COM_STMT_EXECUTE](https://dev.mysql.com/doc/internals/en/com-stmt-execute.html) |
| 18 | [COM_STMT_SEND_LONG_DATA](https://dev.mysql.com/doc/internals/en/com-stmt-send-long-data.html) |
| 19 | [COM_STMT_CLOSE](https://dev.mysql.com/doc/internals/en/com-stmt-close.html) |
| 1a | [COM_STMT_RESET](https://dev.mysql.com/doc/internals/en/com-stmt-reset.html) |
| 1b | COM_OPTION |
| 1c | [COM_STMT_FETCH](https://dev.mysql.com/doc/internals/en/com-stmt-fetch.html) |
| 1d | [COM_DAEMON](https://dev.mysql.com/doc/internals/en/com-daemon.html) |
| 1e | COM_BINLOG_DUMP_GTID |
| 1f | [COM_RESET_CONNECTION](https://dev.mysql.com/doc/internals/en/com-reset-connection.html) |

## 通用回复数据包 Server-->Client

+ OK_Packet
+ ERR_Packet
+ EOF_Packet
+ Status Flags

从 MySQL 5.7.5 开始，EOF_Packet 被 OK_Packet 替代。代表成功和结束都是返回 OK_Packet。

| Type | Name | Description |
| ---- | ---- | ----------- |
| int<1> | header | [00] for OK; [fe] for EOF |
| int<lenenc> | affected_rows | affected rows |
| int<lenenc> | last_inserted_id | last inserted id |

还有部分没有加入表中，更多参考 [https://dev.mysql.com/doc/internals/en/packet-OK_Packet.html](https://dev.mysql.com/doc/internals/en/packet-OK_Packet.html)

+ OK: header = 0 and length_of_packet > 7
+ EOF: header = 0xfe and length_of_packet < 9

ERR_Packet

| Type | Name | Description |
| ---- | ----- | ---------- |
| int<1> | header | [ff] for ERR_Packet |
| int<2> | error_code | error code |
| string<EOF> | error_message | human readable error message |

Status Flags

| Flag | Value |
| ---- | ----- |
| SERVER_STATUS_IN_TRANS | 0x0001 |
| SERVER_STATUS_AUTOCOMMIT | 0x0002 |
| SERVER_MORE_RESULTS_EXISTS | 0x0008 |
| SERVER_STATUS_NO_GOOD_INDEX_USED | 0x0010 |
| SERVER_STATUS_NO_INDEX_USED | 0x0020 |
| SERVER_STATUS_CURSOR_EXISTS | 0x0040 |
| SERVER_STATUS_LAST_ROW_SENT | 0x0080 |
| SERVER_STATUS_DB_DROPPED | 0x0100 |
| SERVER_STATUS_NO_BLACKSLASH_ESCAPES | 0x0200 |
| SERVER_STATUS_METADATA_CHANGED | 0x0400 |
| SERVER_QUERY_WAS_SLOW | 0x0800 |
| SERVER_PS_OUT_PARAMS | 0x1000 |
| SERVER_STATUS_IN_TRANS_READONLY | 0x2000 |
| SERVER_SESSION_STATE_CHANGED | 0x4000 |

字符集 character set

查看MySQL服务支持的 collations

character set 字符集是符号和编码的集合，collation 是如何**比较**编码中的字符的规则的集合。

例子：有 `A=0, B=1, a=2, b=3`，那么 A, B, a, b 是符号 symbol 的集合；0, 1, 2, 3 是编码方式 encoding；而 0<1 所以 A<B 是 collation 比较规则。

```sql
select id, collation_name from information_schema.collations;
```

选择不同的字符集不单单影响数据在 MySQL 服务中的存储方式，还会影响客户端和服务端的通信。

MySQL 认证采用 CHAP 协议。

MySQL 认证成功过程：
1. 客户端连接服务端
2. 服务端回复 Initial_Handshake_Packet 使用验证方法 M
3. 客户端发送 Handshake_Response_Packet 使用验证方法 M
4. 客户端和服务端后续有可能需要交换更多数据包来达到验证
5. 服务端发送 OK_Packet 表明验证成功

成功验证之后，服务端会返回客户端本次连接的 connection_id

MySQL 验证失败过程：
1. 客户端连接服务端
2. 服务端回复 Initial_Handshake_Packet
3. 客户端发送 Handshake_Response_Packet
4. 客户端和服务端后续有可能需要交换更多数据包来达到验证
5. 服务端发送 ERR_Packet 表明验证失败，断开连接

### Protocol::CapabilityFlags

客户端和服务端表明自身支持的功能

[https://dev.mysql.com/doc/internals/en/capability-flags.html](https://dev.mysql.com/doc/internals/en/capability-flags.html)

SQL 语句 prepared 与 非 prepared 有区别。

### Compressed Packet Header

如果传输的数据过长，那么需要对数据进行压缩，能够提升传输效率。

[压缩的格式](https://dev.mysql.com/doc/internals/en/compressed-packet-header.html)

| Type | Description |
| ---- |----------- |
| int<3> | length of compressed payload: 数据包长度-数据包头长度(7bytes) |
| int<1> | compressed sequence id: 压缩的数据太大，无法一次传输，所以需要序列号 |
| int<3> | length of payload before compression: 数据压缩前的大小 |

压缩算法是 defalte。

通常 payload 小于50Bytes不会进行压缩传输。
[发送没有压缩的数据](https://dev.mysql.com/doc/internals/en/uncompressed-payload.html)：
1. 将 length of payload before compression 设置为 0
2. 将没有压缩的数据直接存放在 compressed data 中

### Text Protocol

[https://dev.mysql.com/doc/internals/en/text-protocol.html](https://dev.mysql.com/doc/internals/en/text-protocol.html)



MySQL 协议采用小端方式，比如传输 int<3> 1 对应的是：`01 00 00`

同一个客户端不能够同时向服务端发送两个请求，必须完成一个请求后再进行另一个请求。

对于客户端的 COM_QUERY 的查询语句，服务端返回 [Response 格式](https://dev.mysql.com/doc/internals/en/com-query-response.html)

COM_STMT_PREPARE 会返回一个 prepared statement 对应的 ID。

在 COM_QUERY 中，允许客户端同时向服务端发送多个 sql 语句，他们之间通过 ; 隔开。

问题：
1. 如何处理不同字符集
2. 如何处理经过压缩
3. 捕获命令是否是可执行的 SQL

## binlog

以二进制的形式记录对数据库有修改的事件（insert / update / delete / create / drop），但是高版本的 binlog 会对事件进行修改，binlog 同时包含每个 statement 的执行时间。binlog 不会对 SELECT 等对数据不会产生副作用的事件进行记录。

binlog 目的：
- 增量备份数据库
- 恢复丢失数据
- 主从备份之间的同步

对于时间的 NOW() 等函数，MySQL 会自动跳转到日志生成的时间。

[Binlog Event](https://dev.mysql.com/doc/internals/en/binlog-event.html)


需要处理请求的类型：
- COM_QUERY
- COM_INIT_DB
- COM_STMT_PREPARE
- COM_STMT_EXECUTE
- COM_DROP_DB
- COM_CREATE_DB


## TCP

options for tcp: [mss 1412,sackOK,TS val 845532788 ecr 0,nop,wscale 7]

