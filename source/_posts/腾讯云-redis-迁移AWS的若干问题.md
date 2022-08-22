title: 腾讯云 redis 迁移AWS的若干问题
author: tusimo
tags:
  - redis
  - 腾讯云
  - redis-shake
  - riot-redis
categories:
  - redis
date: 2022-08-22 15:09:00
---
# redis 迁移

## 全量迁移

  全量迁移比较简单,可以采用以下两种方式进行迁移.

  1 rdb 导入导出

	2 scan keys 进行同步

  以上方式可以通过 [`redis-shake`](https://github.com/alibaba/RedisShake)实现. 
  使用比较简单,可以参考官方文档进行配置.
## 增量迁移

  增量迁移可以通过以下方式进行

  1 通过`PSYNC`进行主从同步

  2 通过 redis 的 pub/sub 监听所有 `keys`的变化进行修改增量数据
    `notify-keyspace-events` 可以开启 键值变化通知.通知通过 pub/sub 输出.
    该方法需要开启配置 `notify-keyspace-events`的值为`KEA`
    可以使用 `redis-cli`工具登录控制台并使用命令`config set notify-keyspace-events KEA`设置.
    云厂商也有后台可以修改这些配置来开启该功能.

  `psync` 也可以通过 `redis-shake`实现增量同步.

  但是云厂商的`redis`一般都是禁用了同步相关的 `SYNC` `PSYNC` 等命令.无法通过第一种方式进行同步.

  第二种方式可以使用 [`riot-redis`](https://developer.redis.com/riot/riot-redis/index.html) 进行同步

## riot-redis 介绍

### 安装 - mac

```sh
brew install redis-developer/tap/riot-redis
```

### 命令介绍

#### replicate

`replicate` 使用 redis `dump`和`restore`命令进行同步


参数:

`--scan-count` 表示每次扫描的 `key` 的数量,默认值: `1000`
`--scan-match` 表示要同步的`key`的匹配模式,默认是`*`,即同步所有的 `key`
`--scan-type` 表示扫描的值类型,默认值 全部类型
`--reader-threads` 表示开启多少个读线程, 默认值: `1`
`--reader-batch` 表示单个线程 在一次`pipeline`里 `dump`的值数量 默认值:`50`
`--reader-queue` 是内部共享队列的最大大小. 默认值: `10000` ,该值最小要大于`--reader-threads`*`--reader-batch` 
`--reader-pool` 表示读连接池的大小.可以小于`--reader-threads`.默认值:`8`

`replicate-ds` 使用 对应的读写命令进行还原.

比如读取到的是 `string`, 那么还原的时候就使用`set`命令还原.

### 命令示例

```

//使用默认值全量同步
riot-redis -h redis.database.svc.cluster.local -p 6379 -a test replicate -h redis2.database.svc.cluster.local -p 6379 -a test


//使用默认值全量同步并增量同步
riot-redis -h redis.database.svc.cluster.local -p 6379 -a test replicate -h redis2.database.svc.cluster.local -p 6379 -a test --mode live


//仅使用增量同步
riot-redis -h redis.database.svc.cluster.local -p 6379 -a test replicate -h redis2.database.svc.cluster.local -p 6379 -a test --mode liveonly

//使用自定义值全量同步并增量同步
riot-redis -h redis.database.svc.cluster.local -p 6379 -a test replicate-ds -h redis2.database.svc.cluster.local -p 6379 -a test --reader-threads 4 --scan-count 2000 --reader-batch 200 --reader-queue 50000 --reader-pool 20 --mode live

```

## 问题

如果命令运行执行出现问题.

可以增加 `--info` 或者 `-d`输出详细信息调试.

``` sh
riot-redis --info -h redis.database.svc.cluster.local -p 6379 -a test replicate-ds -h redis2.database.svc.cluster.local -p 6379 -a test --reader-threads 4 --scan-count 2000 --reader-batch 200 --reader-queue 50000 --reader-pool 20 --mode live

riot-redis -d -h redis.database.svc.cluster.local -p 6379 -a test replicate-ds -h redis2.database.svc.cluster.local -p 6379 -a test --reader-threads 4 --scan-count 2000 --reader-batch 200 --reader-queue 50000 --reader-pool 20 --mode live
```


### 连接 AWS 的 redis 报 `Client set AUTH, but no password is set`.

  这个是由于`AWS`默认创建的`redis`没有设置密码,但是运维给了一个`on ~* +@all`的字符串说是密码.实际上这个是`AWS`的权限控制字符串.并不是密码.
  所以使用空密码就不报错了.
### 连接腾讯云 的 redis 报 ` NOAUTH Authentication required`.
  1 首先确认 `redis` 的密码是否正确
  2 查看 `redis` 版本. 我在腾讯云上 使用 `redis` 4.0 版本的时候在密码正确的情况下,会报这个错误.升级到 5.0 版本就可以正常同步了.
### 同步报 `syntax error`
   可以将同步方式 修改为 `replicate-ds`试试看.