# 底层文件缓存相关的内置函数

当用户选择使用对象存储作为AO/AOCS表底层存储时，相关的数据会通过Gopher文件缓存系统与数据库进行交互。

当用户将Gopher系统中的文件打开选项设置为需要缓存时，系统中会产生每次访问数据的缓冲数据。

Hashdata数据库中提供了相关的内置函数来获取缓存数据信息，并进行相关的清理工作。

## 获取指定AO/AOCS表的缓存大小

```sql
-- 根据表OID
SELECT gp_toolkit.__gopher_cache_size_relation_oid(TABLEOID);
-- 根据表名称
SELECT gp_toolkit.__gopher_cache_size_relation_name('TABLENAME');
```

## 释放全部Gopher文件系统的缓存信息

```sql
SELECT gp_toolkit.__gopher_free_all_cache();
```

## 释放指定AO/AOCS表的缓存

```sql
-- 根据表OID释放
SELECT gp_toolkit.__gopher_cache_free_relation_oid(TABLEOID);
-- 根据表名释放
SELECT gp_toolkit.__gopher_cache_free_relation_name('TABLENAME');
```

## 释放指定数据库的缓存

```sql
SELECT gp_toolkit.__gopher_cache_free_db_name('DBNAME');
```


