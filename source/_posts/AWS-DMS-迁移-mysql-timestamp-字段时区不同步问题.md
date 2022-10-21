title: AWS DMS 迁移 mysql timestamp 字段时区不同步问题
author: tusimo
tags:
  - mysql
  - dms
  - cdc
  - fulload
  - timestamp
  - timezone
  - 时区
categories: []
date: 2022-10-21 18:31:00
---
# 问题

线上服务 使用`dms`迁移到 aws ,在迁移数据库的时候遇到 `timestamp` 字段迁移出现时区不同步问题.
在 `dms` 使用`fullload`全量 和`cdc`增量模式时.`fullload`全量 的时区正常.`cdc`增量模式时区相差 8 个小时.

# 调试

我们源数据库在腾讯云上.`mysql`版本 5.7.时区为`Asia/Shanghai`

```shell
mysql> show global variables like '%time_zone%';
+------------------+--------+
| Variable_name    | Value  |
+------------------+--------+
| system_time_zone | CST    |
| time_zone        | SYSTEM |
+------------------+--------+
2 rows in set (0.09 sec)

```
目标数据库在 aws 上,`mysql`版本为 5.7.时区为`Asia/Shanghai`

```shell
mysql> show global variables like '%time_zone%';
+------------------+---------------+
| Variable_name    | Value         |
+------------------+---------------+
| system_time_zone | UTC           |
| time_zone        | Asia/Shanghai |
+------------------+---------------+
2 rows in set (0.08 sec)

```

`DMS`的 `source endpoint` 设置 `ServerTimezone`为`Asia/Shanghai`

```
{
    "Port": 63642,
    "ServerName": "*****",
    "ServerTimezone": "Asia/Shanghai",
    "Username": "****"
}
```
`DMS`的`target endpoint` 没做额外的时区配置.

我们在启动`dms`同步时,发现在全量模式下,`timestamp`字段的值是正常的.全量同步完进行增量同步的时候时区就相差 8 个小时.然而数据的插入时区也是正常的.配置当时参考了这个[https://aws.amazon.com/premiumsupport/knowledge-center/dms-migrate-mysql-non-utc/](https://aws.amazon.com/premiumsupport/knowledge-center/dms-migrate-mysql-non-utc/)

# 解决

在尝试无果后,咨询了`AWS`的技术人员.得到了一个令人诧异的答案.当时就`emo`了.

```
with batch-optimization，you need to use serverTimezone=Asia/Shanghai at the source endpoint like what you have done before.
without batch-optimization，you need to use initstmt=SET time_zone='Asia/Shanghai in ECA at the source endpoint.
```

![upload successful](/images/pasted-13.png)

原来在`dms`里面存在一个 批量插入优化的选项.当我们在`source endpoint` ,使用`ServerTimezone`这个设置的时候,必须要开启这个批量优化的选项.不开启就会出现时区问题.开启了就正常了.
`dms`还有一个设置时区的方法.就是使用`Extra connection attributes`.

如下图,两种配置的设置位置,任选一种设置,但是不同的设置要使用不同的`batch-optimization`配置:

![upload successful](/images/pasted-14.png)

| 配置 | batch-optimization配置 | 结果 |
| :-----: | :----: | :----: |
|使用ServerTimezone=Asia/Shanghai| 开启|时区均正常|
|使用ServerTimezone=Asia/Shanghai| 不开启|全量正常,增量同步时区不正常|
|使用initstmt=SET time_zone='Asia/Shanghai'| 开启|全量正常,增量同步时区不正常|
|使用initstmt=SET time_zone='Asia/Shanghai'| 不开启|时区均正常|

由于`batch-optimization`默认是不开启的,所以建议配置使用initstmt=SET time_zone='Asia/Shanghai'.

由于腾讯云的数据库里面没有`Asia/Shanghai`这个时区的配置,我这里使用的是initstmt=SET time_zone='+08:00',效果是一样的.

不要两个一起都设置.你会发现时区都不对了.

# 总结
这个问题困扰了很久.在得知是由于开启关闭批量插入优化导致时区不正常就非常难受.
总结也不想写了.就这样吧.