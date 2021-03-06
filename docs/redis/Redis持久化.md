---
title: Redis 持久化
date: 2018/06/11
categories:
- database
tags:
- database
- nosql
- key-value
---

# Redis 持久化

> Redis 支持持久化，即把数据存储到硬盘中。
>
> Redis 提供了两种持久化方式：
>
> **RDB 快照（snapshot）** - 将存在于某一时刻的所有数据都写入到硬盘中。
>
> **只追加文件（append-only file，AOF）** - 它会在执行写命令时，将被执行的写命令复制到硬盘中。
>
> 这两种持久化方式既可以同时使用，也可以单独使用。
>
> 将内存中的数据存储到硬盘的一个主要原因是为了在之后重用数据，或者是为了防止系统故障而将数据备份到一个远程位置。另外，存储在 Redis 里面的数据有可能是经过长时间计算得出的，或者有程序正在使用 Redis 存储的数据进行计算，所以用户会希望自己可以将这些数据存储起来以便之后使用，这样就不必重新计算了。

<!-- TOC depthFrom:2 depthTo:3 -->

- [快照](#快照)
    - [快照的原理](#快照的原理)
    - [快照的配置](#快照的配置)
    - [快照的优点](#快照的优点)
    - [快照的缺点](#快照的缺点)
- [AOF](#aof)
    - [AOF 的原理](#aof-的原理)
    - [AOF 的配置](#aof-的配置)
    - [重写/压缩 AOF](#重写压缩-aof)
    - [AOF 的优点](#aof-的优点)
    - [AOF 的缺点](#aof-的缺点)
- [选择持久化方式](#选择持久化方式)
    - [怎样从快照方式切换为 AOF 方式](#怎样从快照方式切换为-aof-方式)
    - [AOF 和快照之间的相互作用](#aof-和快照之间的相互作用)
- [备份](#备份)
    - [容灾备份](#容灾备份)
    - [Redis 复制的启动过程](#redis-复制的启动过程)
- [资料](#资料)

<!-- /TOC -->

Redis 提供了两种持久方式：RDB 和 AOF。你可以同时开启两种持久化方式。在这种情况下, 当 redis 重启的时候会优先载入 AOF 文件来恢复原始的数据，因为在通常情况下 AOF 文件保存的数据集要比 RDB 文件保存的数据集要完整。

## 快照

Redis 可以通过创建快照来获得存储在内存里面的数据在某个时间点上的副本。在创建快照之后，用户可以对快照进行备份，可以将快照复制到其他服务器从而创建具有相同数据的服务器副本，还可以将快好留在原地以便重启服务器时使用。

根据配置，快照将被写入 `dbfilename` 选项指定的文件里面，并存储在 `dir` 选项指定的路径上面。

创建快照的方法：

- 客户端可以通过向 Redis 发送 BGSAVE 命令来创建一个快照。对于支持 BGSAVE 命令的平台来说，Redis 会创建一个子进程，然后子进程负责将快照写入到硬盘，而父进程则继续处理命令请求。
- 客户端还可以通过向 Redis 发送 SAVE 命令来创建一个快照。接到 SAVE 命令的 Redis 服务器在快照创建完毕之前将不再响应任何其他命令。
- 如果用户设置了 save 配置选项，比如 save 60 10000，当这个条件被满足时，Redis 就会自动触发 BGSAVE 命令。如果用户设置了多个 save 配置选项所设置的条件被满足时，Redis 就会触发一次 BGSAVE 命令。
- 当 Redis 通过 SHUTDOWN 命令接受到关闭服务器的请求时，或者接收到标准 TERM 信号时，会执行一个 SAVE 命令，阻塞所有客户端，不再执行客户端发送的任何命令，并在 SAVE 命令执行完毕之后关闭服务器。
- 当一个 Redis 服务器连接另一个 Redis 服务器，并向对方发送 SYNC 命令来开始一次复制操作的时候，如果主服务器目前没有在执行 BGSAVE 操作，或者主服务器并非刚刚执行完 BGSAVE 操作，那么主服务器就会执行 BGSAVE 命令。

快照持久化方式能够在指定的时间间隔能对整个数据进行快照存储。

使用快照持久化来保存数据是，需要记住：**如果系统真的发生崩溃，用户将丢失最近一次生成快照之后更改的所有数据。**

快照配置：

```
save 60 1000
stop-writes-on-bgsave-error no
rdbcompression yes
dbfilename dump.rdb
```

### 快照的原理

在默认情况下，Redis 将数据库快照保存在名字为 dump.rdb 的二进制文件中。你可以对 Redis 进行设置， 让它在“N 秒内数据集至少有 M 个改动”这一条件被满足时， 自动保存一次数据集。你也可以通过调用 SAVE 或者 BGSAVE，手动让 Redis 进行数据集保存操作。这种持久化方式被称为快照。

当 Redis 需要保存 dump.rdb 文件时， 服务器执行以下操作:

- Redis 创建一个子进程。
- 子进程将数据集写入到一个临时快照文件中。
- 当子进程完成对新 快照文件的写入时，Redis 用新快照文件替换原来的快照文件，并删除旧的快照文件。

这种工作方式使得 Redis 可以从写时复制（copy-on-write）机制中获益。

### 快照的配置

比如说， 在 redis.conf 中添加如下配置，表示让 Redis 在满足“ 60 秒内有至少有 1000 个键被改动”这一条件时， 自动保存一次数据集:

```
save 60 10000
```

### 快照的优点

- RDB 是一个非常紧凑的文件，它保存了某个时间点的数据集，非常适用于数据集的备份。比如你可以在每个小时报保存一下过去 24 小时内的数据，同时每天保存过去 30 天的数据，这样即使出了问题你也可以根据需求恢复到不同版本的数据集。
- RDB 是一个紧凑的单一文件，很方便传送到另一个远端数据中心或者亚马逊的 S3（可能加密），非常适用于灾难恢复。
- 快照在保存 RDB 文件时父进程唯一需要做的就是 fork 出一个子进程，接下来的工作全部由子进程来做，父进程不需要再做其他 IO 操作，所以快照持久化方式可以最大化 redis 的性能。
- 与 AOF 相比，在恢复大的数据集的时候，DB 方式会更快一些。

### 快照的缺点

- 如果你希望在 redis 意外停止工作（例如电源中断）的情况下丢失的数据最少的话，那么 快照不适合你。虽然你可以配置不同的 save 时间点(例如每隔 5 分钟并且对数据集有 100 个写的操作)，是 Redis 要完整的保存整个数据集是一个比较繁重的工作，你通常会每隔 5 分钟或者更久做一次完整的保存，万一在 Redis 意外宕机，你可能会丢失几分钟的数据。
- 快照需要经常 fork 子进程来保存数据集到硬盘上。当数据集比较大的时候，fork 的过程是非常耗时的，可能会导致 Redis 在一些毫秒级内不能响应客户端的请求。如果数据集巨大并且 CPU 性能不是很好的情况下，这种情况会持续 1 秒。AOF 也需要 fork，但是你可以调节重写日志文件的频率来提高数据集的耐久度。

## AOF

AOF 持久化方式记录每次对服务器执行的写操作。当服务器重启的时候会重新执行这些命令来恢复原始的数据。

AOF 命令以 redis 协议追加保存每次写的操作到文件末尾。Redis 还能对 AOF 文件进行后台重写。使得 AOF 文件的体积不至于过大。

```
appendonly no
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

### AOF 的原理

- Redis 创建一个子进程。
- 子进程开始将新 AOF 文件的内容写入到临时文件。
- 对于所有新执行的写入命令，父进程一边将它们累积到一个内存缓存中，一边将这些改动追加到现有 AOF 文件的末尾，这样样即使在重写的中途发生停机，现有的 AOF 文件也还是安全的。
- 当子进程完成重写工作时，它给父进程发送一个信号，父进程在接收到信号之后，将内存缓存中的所有数据追加到新 AOF 文件的末尾。
- 搞定！现在 Redis 原子地用新文件替换旧文件，之后所有命令都会直接追加到新 AOF 文件的末尾。

### AOF 的配置

AOF 持久化通过在 `redis.conf` 中的 `appendonly yes` 配置选项来开启。

可以通过 `appendfsync` 配置选项来设置同步频率：

- **always** - 每个 Redis 写命令都要同步写入硬盘。这样做会严重降低 Redis 的速度。
- **everysec** - 每秒执行一次同步，显示地将多个写命令同步到硬盘。
- **no** - 让操作系统来决定应该何时进行同步。

为了兼顾数据安全和写入性能，推荐使用 `appendfsync everysec` 选项。Redis 每秒同步一次 AOF 文件时的性能和不使用任何持久化特性时的性能相差无几。

### 重写/压缩 AOF

随着 Redis 不断运行，AOF 的体积也会不断增长，这将导致两个问题：

1.  AOF 耗尽磁盘可用空间。
2.  Redis 重启后需要执行 AOF 文件记录的所有写命令来还原数据集，如果 AOF 过大，则还原操作执行的时间就会非常长。

这个问题的解决方法：

执行 `BGREWRITEAOF` 命令，这个命令会通过移除 AOF 中的冗余命令来重写 AOF 文件，使 AOF 文件的体积尽可能地小。

`BGREWRITEAOF` 命令与 `BGSAVE` 原理类似：通过创建一个子进程，然后由子进程负责对 AOF 文件进行重写。

可以通过设置 `auto-aof-rewrite-percentage` 和 `auto-aof-rewrite-min-size`，使得 Redis 在满足条件时，自动执行 `BGREWRITEAOF`。

假设配置如下：

```
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

表明，当 AOF 大于 64MB，且 AOF 体积比上一次重写后的体积大了至少 100% 时，执行 `BGREWRITEAOF`。

### AOF 的优点

- 使用 AOF 会让你的 Redis 更加耐久: 你可以使用不同的 fsync 策略：无 fsync；每秒 fsync；每次写的时候 fsync。使用默认的每秒 fsync 策略，Redis 的性能依然很好(fsync 是由后台线程进行处理的,主线程会尽力处理客户端请求)，一旦出现故障，你最多丢失 1 秒的数据。
- AOF 文件是一个只进行追加的日志文件，所以不需要写入 seek，即使由于某些原因(磁盘空间已满，写的过程中宕机等等)未执行完整的写入命令，你也也可使用 redis-check-aof 工具修复这些问题。
- Redis 可以在 AOF 文件体积变得过大时，自动地在后台对 AOF 进行重写：重写后的新 AOF 文件包含了恢复当前数据集所需的最小命令集合。整个重写操作是绝对安全的，因为 Redis 在创建新 AOF 文件的过程中，会继续将命令追加到现有的 AOF 文件里面，即使重写过程中发生停机，现有的 AOF 文件也不会丢失。而一旦新 AOF 文件创建完毕，Redis 就会从旧 AOF 文件切换到新 AOF 文件，并开始对新 AOF 文件进行追加操作。
- AOF 文件有序地保存了对数据库执行的所有写入操作，这些写入操作以 Redis 协议的格式保存。因此 AOF 文件的内容非常容易被人读懂，对文件进行分析（parse）也很轻松。 导出（export） AOF 文件也非常简单。举个例子，如果你不小心执行了 FLUSHALL 命令，但只要 AOF 文件未被重写，那么只要停止服务器，移除 AOF 文件末尾的 FLUSHALL 命令，并重启 Redis ，就可以将数据集恢复到 FLUSHALL 执行之前的状态。

### AOF 的缺点

- 对于相同的数据集来说，AOF 文件的体积通常要大于 RDB 文件的体积。
- 根据所使用的 fsync 策略，AOF 的速度可能会慢于快照。在一般情况下，每秒 fsync 的性能依然非常高，而关闭 fsync 可以让 AOF 的速度和快照一样快，即使在高负荷之下也是如此。不过在处理巨大的写入载入时，快照可以提供更有保证的最大延迟时间（latency）。

## 选择持久化方式

如果你只希望你的数据在服务器运行的时候存在，你可以不使用任何持久化方式。

如果你非常关心你的数据，但仍然可以承受数分钟以内的数据丢失，那么你可以只使用 快照持久化。

如果你不能承受数分钟以内的数据丢失，那么你可以同时使用快照持久化和 AOF 持久化。

有很多用户都只使用 AOF 持久化， 但我们并不推荐这种方式： 因为定时生成 RDB 快照（snapshot）非常便于进行数据库备份，并且快照恢复数据集的速度也要比 AOF 恢复的速度要快，除此之外，使用快照还可以避免之前提到的 AOF 程序的 bug 。

### 怎样从快照方式切换为 AOF 方式

在 Redis 2.2 或以上版本，可以在不重启的情况下，从快照切换到 AOF ：

- 为最新的 dump.rdb 文件创建一个备份。
- 将备份放到一个安全的地方。
- 执行以下两条命令:
- redis-cli config set appendonly yes
- redis-cli config set save “”
- 确保写命令会被正确地追加到 AOF 文件的末尾。
- 执行的第一条命令开启了 AOF 功能： Redis 会阻塞直到初始 AOF 文件创建完成为止， 之后 Redis 会继续处理命令请求， 并开始将写入命令追加到 AOF 文件末尾。

执行的第二条命令用于关闭快照功能。 这一步是可选的， 如果你愿意的话， 也可以同时使用快照和 AOF 这两种持久化功能。

重要:别忘了在 redis.conf 中打开 AOF 功能！ 否则的话， 服务器重启之后， 之前通过 CONFIG SET 设置的配置就会被遗忘， 程序会按原来的配置来启动服务器。

### AOF 和快照之间的相互作用

在版本号大于等于 2.4 的 Redis 中， BGSAVE 执行的过程中， 不可以执行 BGREWRITEAOF 。 反过来说， 在 BGREWRITEAOF 执行的过程中， 也不可以执行 BGSAVE。这可以防止两个 Redis 后台进程同时对磁盘进行大量的 I/O 操作。

如果 BGSAVE 正在执行， 并且用户显示地调用 BGREWRITEAOF 命令， 那么服务器将向用户回复一个 OK 状态， 并告知用户， BGREWRITEAOF 已经被预定执行： 一旦 BGSAVE 执行完毕， BGREWRITEAOF 就会正式开始。 当 Redis 启动时， 如果快照持久化和 AOF 持久化都被打开了， 那么程序会优先使用 AOF 文件来恢复数据集， 因为 AOF 文件所保存的数据通常是最完整的。

## 备份

**务必确保你的数据有完整的备份。**

磁盘故障、节点失效，诸如此类的问题都可能让你的数据消失不见，不进行备份是非常危险的。

备份 Redis 数据建议采用如下策略：

备份 Redis 数据建议采用快照方式。RDB 文件一旦创建，就不会进行任何修改，所以十分安全。

Redis 快照备份过程：

- 创建一个定期任务（cron job），每小时将一个 RDB 文件备份到一个文件夹，并且每天将一个 RDB 文件备份到另一个文件夹。
- 确保快照的备份都带有相应的日期和时间信息，每次执行定期任务脚本时，使用 find 命令来删除过期的快照：比如说，你可以保留最近 48 小时内的每小时快照，还可以保留最近一两个月的每日快照。
- 至少每天一次，将 RDB 备份到你的数据中心之外，或者至少是备份到你运行 Redis 服务器的物理机器之外。

### 容灾备份

Redis 的容灾备份基本上就是对数据进行备份，并将这些备份传送到多个不同的外部数据中心。

容灾备份可以在 Redis 运行并产生快照的主数据中心发生严重的问题时，仍然让数据处于安全状态。

以下是一些实用的容灾备份方法：

- Amazon S3 ，以及其他类似 S3 的服务，是一个构建灾难备份系统的好地方。 最简单的方法就是将你的每小时或者每日 RDB 备份加密并传送到 S3 。对数据的加密可以通过 gpg -c 命令来完成（对称加密模式）。记得把你的密码放到几个不同的、安全的地方去（比如你可以把密码复制给你组织里最重要的人物）。同时使用多个储存服务来保存数据文件，可以提升数据的安全性。
- 传送快照可以使用 SCP 来完成（SSH 的组件）。 以下是简单并且安全的传送方法： 买一个离你的数据中心非常远的 VPS ，装上 SSH ，创建一个无口令的 SSH 客户端 key ，并将这个 key 添加到 VPS 的 authorized_keys 文件中，这样就可以向这个 VPS 传送快照备份文件了。为了达到最好的数据安全性，至少要从两个不同的提供商那里各购买一个 VPS 来进行数据容灾备份。
- 需要注意的是，这类容灾系统如果没有小心地进行处理的话，是很容易失效的。最低限度下，你应该在文件传送完毕之后，检查所传送备份文件的体积和原始快照文件的体积是否相同。如果你使用的是 VPS ，那么还可以通过比对文件的 SHA1 校验和来确认文件是否传送完整。

另外， 你还需要一个独立的警报系统， 让它在负责传送备份文件的传送器（transfer）失灵时通知你。

### Redis 复制的启动过程

<div align="center">
<img src="https://raw.githubusercontent.com/dunwu/database/master/images/redis/Redis复制启动过程.png" width="400"/>
</div>

当多个从服务器尝试连接同一个主服务器时：

- 上图步骤 3 尚未执行：所有从服务器都会接收到相同的快照文件和相同的缓冲区写命令。
- 上图步骤 3 正在执行或已经执行完毕：当主服务器与较早进行连接的从服务器执行完复制所需的 5 个步骤之后，主服务器会与新连接的从服务器执行一次新的步骤 1 至步骤 5。

## 资料

- [Redis 官网](https://redis.io/)
- [Redis 实战](https://item.jd.com/11791607.html)
- [Redis Persistence](https://redis.io/topics/persistence)
