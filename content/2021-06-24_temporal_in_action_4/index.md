+++
title = "Temporal实战 (4) TiDB兼容性"
description = ""
draft = false

[taxonomies]
tags = ["Temporal", "Workflow"]
categories = ["Temporal in Action"]
+++

Temporal原生支持Cassandra, MySQL, PostgreSQL作为后端存储. 考虑到TiDB与MySQL高度兼容, 以及公司的技术栈, 我们尝试使用TiDB作为Temporal的后端存储. 然而在使用过程中还是遇到了一些问题, 在此进行记录 (持续更新中).

<!-- more -->

## 兼容性问题

### 乐观事务

低版本TiDB默认使用乐观事务, Temporal在乐观事务模式下甚至无法启动. 报错如下:

```text
Unable to start server. Error: unable to initialize system namespace: unable to register system namespace: CreateNamespace operation failed. Failed to commit transaction. Error: Error 1062: Duplicate entry '54321-2�hxr@��c��Y�j�' for key 'PRIMARY'
```

Temporal Server在启动时, 会开启事务, 向`temporal.namespaces`中插入一条记录系统namespace的记录, 同时更新`temporal.namespace_metadata`表中的notification_version字段, 将其加1, 然后提交事务. 每次插入的系统namespace记录的`partition_id`与`id`均相同, 而`namespaces`表以`partition_id`和`id`作为联合主键, 因此每次启动Server时都会报`ERROR 1062: Duplicate entry`主键重复错误. 源码中会处理INSERT时报错出现的`ERROR 1062`, 并封装成特定类型错误`serviceerror.NewNamespaceAlreadyExists`返回给上层, 上层对error类型进行判断, 如果是该类型, 则忽略错误.

对悲观事务模式 (MySQL, 以及TiDB开启悲观事务模式) 来说, 这样的处理没有问题. 然而在乐观事务模式下, `ERROR 1062`并不是在INSERT时返回的, 而是在COMMIT时返回的. 而Temporal并没有对Commit error中的`ERROR 1062`进行特殊处理 (感觉也没法处理, 因为不好判定到底是执行哪条语句报的错). 上层并没有捕捉到该错误, 并认为是系统错误, 导致进程退出. 以上就是问题的原因.

用TiDB (乐观事务模式) 测试一下对应的SQL:

```sql
mysql> begin;
Query OK, 0 rows affected (0.04 sec)

mysql> insert into namespaces values (54321, 0x32049B68787240948E63D0DD59896A83, 'temporal-system', 1, '12345', 'abc', 0);
Query OK, 1 row affected (0.04 sec)

mysql> commit;
ERROR 1062 (23000): Duplicate entry '54321-2�hxr@��c��Y�j�' for key 'PRIMARY'
```

再用MySQL测试一下:

```sql
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into namespaces values (54321, 0x32049B68787240948E63D0DD59896A83, 'temporal-system', 1, '12345', 'abc', 0);
ERROR 1062 (23000): Duplicate entry '54321-2�hxr@��c��Y�j�' for key 'namespaces.PRIMARY'
```

将TiDB切换成悲观事务模式后, 该问题解决. 所以如果要使用TiDB作为Temporal作为后端存储, 建议开启悲观事务.

以上问题已经提交issue [temporal/issues/1684](https://github.com/temporalio/temporal/issues/1684) 和 [helm-charts/issues/204](https://github.com/temporalio/helm-charts/issues/204) 反馈给官方, 感兴趣的话可以跟进.

## 总结

使用TiDB作为Temporal后端存储时, 需要注意:

- 将TiDB开启悲观事务
