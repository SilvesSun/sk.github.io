---
title: redis持久化
date: 2018-09-05 10:09:25
tags:
- 数据库
- RDB
- AOF
categories:
- redis
---


redis 支持 RDB 和 AOF 两种持久化机制.

## RDB
在 Redis 运行时， RDB 程序将当前内存中的数据库快照保存到磁盘文件中， 在 Redis 重启动时， RDB 程序可以通过载入 RDB 文件来还原数据库的状态。

RDB 持久化的核心是 `rdbsave` 和 `rdbload` 两个函数, 分别用于生成 rdb 文件和将存储的文件加载到内存中

触发 RDB 持久化分为手动触发和自动触发. 手动触发对应的是 `save` 和 `bgsave` 命令

save 命令会阻塞当前 redis 服务器, 直到RDB过程完成. bgsave 会执行 fork 操作创建子进程, RDB 持久化过程由子进程完成, 阻塞发生在 fork 阶段. 

自动持久化发生的情况如下:
- 配置`save m n`, 这样当在m秒内发生n次修改时, 就会自动触发 bgsave
- 节点执行全量复制, 主节点会自动执行 bgsave 生成 RBD 文件发送给节点
- `debug reload` 命令会自动触发save操作
- 默认情况下执行 `shutdown`, 如果没有开启 AOF 持久化功能, 就会自动执行 bgsave

### bgsave 的基本原理
redis通过创建子进程来进行bgsave操作, 使用写时复制(copy-on-write)策略, 即fork函数发生的一刻父子进程共享同一内存数据，当父进程要更改其中某片数据时（如执行一个写命令 ），操作系统会将该片数据复制一份以保证子进程的数据不受影响，所以新的RDB文件存储的是执行fork那一刻的内存数据

Redis在进行快照的过程中不会修改RDB文件，只有快照结束后才会将旧的文件替换成新的，也就是说任何时候RDB文件都是完整的。这使得我们可以通过定时备份RDB文件来实 现Redis数据库备份。RDB文件是经过压缩（可以配置rdbcompression参数以禁用压缩节省CPU占用）的二进制格式，所以占用的空间会小于内存中的数据大小，更加利于传输。

## AOF(append only file)
以独立日志的方式记录每次写命令, 重启时再重新执行AOF文件中的命令达到恢复数据的目的.

AOF 主要是解决了数据持久化的实时性问题

AOF 的工作流程:

> 命令写入(append) -> 文件同步(sync) -> 文件重写(rewrite) -> 重启加载(load)


所有的写入命令会追加到 aof_buf 中, 然后根据对应的策略向硬盘中作同步操作, 随着AOF 文件增大, 定期对AOF进行重写, 压缩文件. 然后当重启服务器时, 可以加载 AOF 文件进行数据恢复.

appendfsync 参数指明了 aof_buf 同步到硬盘中的策略, 分为三种
- always	每一个redis写命令都会被写入到aof文件中，这样会严重降低redis的速度
- everysec	每秒钟执行一次同步，将这一秒钟之内接受到的命令写入aof文件中
- no	并不是不进行aof持久化，而是让操作系统决定什么时候将命令写入aof文件中，这样我们丢失的数据将不可控

## 总结
bgsave做镜像全量持久化，aof做增量持久化。因为bgsave会耗费较长时间，不够实时，在停机的时候会导致大量丢失数据，所以需要aof来配合使用。在redis实例重启时，会使用bgsave持久化文件重新构建内存，再使用aof重放近期的操作指令来实现完整恢复重启之前的状态。