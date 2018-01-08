http://www.cnblogs.com/roverliang/p/4817820.html
《MySQL核心技术与最佳实践》


# 简介
一句话介绍: 精炼强悍的开源的关系型数据库

# 存储引擎

最常用的两种: MyISAM和InnoDB(其中InnoDB为默认)

## 查看当前表的存储引擎

```
MariaDB [none]> show table status from test where name='mytable2'\G
*************************** 1. row ***************************
         Name: mytable2
         Engine: InnoDB

```

## 修改存储引擎

```
MariaDB [test]> alter table mytable2 engine=innodb;
```

看到这里,你会想,这两种引擎有毛线区别啊?我使用默认的不就好了,下面为你介绍一下

## MyISAM和InnoDB的区别

### 功能区别

- **MyISAM类型的表强调的是性能，其执行数度比InnoDB类型更快，但是不提供事务支持**
- **InnoDB提供事务支持事务，外部键等高级数据库功能**
- InnoDB支持行锁（某些情况下还是锁整表，如 update table set a=1 where user like '%lee%'

也就是说,如果你想执行`rollback`命令回滚数据的时候，如果你的表是MyISAM类型，你会悲催的发现，回滚不了

### 构成上的区别

每个MyISAM在磁盘上存储成三个文件。第一个文件的名字以表的名字开始，扩展名指出文件类型。

- .frm文件存储表定义。
- 数据文件的扩展名为.MYD (MYData)。
- 索引文件的扩展名是.MYI (MYIndex)。

InnoDB 表的大小只受限于操作系统文件的大小，一般为 2GB

### 工程实践

如果大量的select操作，可以考虑使用MyISAM，写操作多使用InnoDB


# 索引

简单来说就类似于hash表里面的key

- 索引会占用磁盘空间，是典型的“以空间换时间”
- 一张表可以有多个索引，也可以是字段的组合（复合索引），但是不能跨表建索引
- 索引不是越多越好，更新数据的时候，过多索引会导致IO上升从而降低性能

## 索引选取原则

- 1、表中的某字段离散度越高，该字段越适合作为选作索引的关键字。
> 数据库用户在创建主键约束的同时，MySQL会自动的创建主索引（primary index）且索引名为primary ；在创建唯一约束的同时，MySQL会自动创建唯一性约束（unique index）默认情况下，索引名为唯一约束性约束的字段名。

- 2、占用存储空间少的字段更适合选作索引的关键字。
- 3、存储空间固定的字段更适合选作索引的关键字。
- 4、 where字句中经常使用的字段应该创建索引，分组字段或者排序字段应该创建索引，两个表的连接字段应该创建索引。
> 引入索引的目的是为了提高检索的效率，因此索引关键字的选择与select语句息息相关。这句话有两个方面的含义，select语句的设计可以决定索引的设计；索引的设计也同样影响着select的设计。select语句中的where字句、group by 字句，以及order by字句，又可以影响索引的设计。两个表的链接字段应该创建索引，外键约束一经创建，mysql便会自动地创建与外键相对应的索引，这是由于外键字段通常是两个表的连接字段。
- 原则5、更新频繁的字段不适合创建索引，不会出现在where字句中的字段不应该创建索引。
- 原则6、最左前缀的原则。
> 复合索引还有另外一个有点，他通过被称为“最左前缀”（leftmost prefixing）的概念体现出来，假设向一个表的多个字段（例如 firstName，lastName、address）创建复合索引（索引名：firstname_lastname_address）。当where查询的条件是以下各种字段的组合时，mysql将使用fname_lastname_address索引。其他情况将无法使用fname_lastname_address索引firstname,lastname,address firstname,;lastnamefirstname

- 7、 尽量使用前缀索引
> 仅仅在姓名中的姓氏部分创建索引，从而可以节省索引的存储空间，提高检索效率。与数据库的设计一样，索引的设计同样需要数据库开发人员经验的积累，以及智慧的沉淀，同时需要依据系统各自的特点设计出更好的索引，在“加快检索效率”与“降低更新速度”之间做好平衡。从而大幅提升数据库的整体性能。

# 备份与恢复

## 备份

如果你需要一次性备份所有的数据库,可以使用如下命令

```
mysqldump -uroot -p --all-databases > backup.sql
```

如果只需要备份单个的数据库

```
mysqldump -uroot -p cinder > cinder.sql   # 其中cinder为数据库名
```

## 恢复

进入数据库,然后执行下面命令

```
MariaDB [test]> source /home/czs/backup.sql;
```

## 自动备份脚本

保存7天的备份

```
#!/bin/sh

db_user="root"   
db_passwd="root"

# the directory for story your backup file.   
backup_dir="/home/mysql_sql/backup"

Now=$(date +"%d-%m-%Y") 
File=bakmysql-$Now.sql

mysqldump -u$db_user  -p$db_passwd --all-databases > "$backup_dir/$File"
echo "Your database backup successfully completed"

SevenDays=$(date -d -7day  +"%d-%m-%Y")

if [ -f $backup_dir/bakmysql-$SevenDays.sql ] 
then
  rm -rf $backup_dir/bakmysql-$SevenDays.sql 
  echo "You have delete 7days ago bak file "
else
  echo "7days ago bak file not exist "
fi
```

```
# 前面略,下面这个有压缩,并且根据创建时间来删除文件

mysqldump -u$db_user  -p$db_passwd --all-databases | gzip > $backup_dir/$File.gz
echo "Your database backup successfully completed"

find $backup_dir -name "bakmysql*.sql.gz" -type f -mtime +$db_cycle -exec rm {} \; > /dev/null 2>&1
echo "You have delete $db_cycle days ago bak file "
```


# 远程访问权限



# 表修复

有时候访问表时候，会提示如下错误：

```
MariaDB [mysql]> select * from user;
ERROR 1194 (HY000): Table 'user' is marked as crashed and should be repaired
```

可用如下两个方法修复:

1.进入数据表所在的目录,执行如下命令:(结合使用)

```
myisamchk -r tblName  #注意修复之后表文件所属不要变成root
```

2.登录数据库,选择对应数据库，使用如下命令

```
[mysql]> REPAIR TABLE user;
+------------+--------+----------+------------------------------------+
| Table      | Op     | Msg_type | Msg_text                           |
+------------+--------+----------+------------------------------------+
| mysql.user | repair | warning  | Number of rows changed from 7 to 6 |
| mysql.user | repair | status   | OK                                 |
+------------+--------+----------+------------------------------------+
```

# 事务

注意,要InnoDB才支持事务

## 事务控制语句

> - BEGIN或START TRANSACTION: 显式地开启一个事务；
- COMMIT: 也可以使用COMMIT WORK，不过二者是等价的。COMMIT会提交事务，并使已对数据库进行的所有修改称为永久性的；
- ROLLBACK: 有可以使用ROLLBACK WORK，不过二者是等价的。回滚会结束用户的事务，并撤销正在进行的所有未提交的修改；
- SAVEPOINT identifier: SAVEPOINT允许在事务中创建一个保存点，一个事务中可以有多个SAVEPOINT；
- RELEASE SAVEPOINT identifier: 删除一个事务的保存点，当没有指定的保存点时，执行该语句会抛出一个异常；
- ROLLBACK TO identifier: 把事务回滚到标记点；
- SET TRANSACTION: 用来设置事务的隔离级别。InnoDB存储引擎提供事务的隔离级别有READ UNCOMMITTED、READ COMMITTED、REPEATABLE READ和SERIALIZABLE。

## 方法

1、用 BEGIN, ROLLBACK, COMMIT来实现

- BEGIN 开始一个事务
- ROLLBACK 事务回滚
- COMMIT 事务确认

2、直接用 SET 来改变 MySQL 的自动提交模式:

- SET AUTOCOMMIT=0 禁止自动提交
- SET AUTOCOMMIT=1 开启自动提交

## 实例测试

```
mysql> CREATE TABLE runoob_transaction_test( id int(5)) engine=innodb;  # 创建数据表
Query OK, 0 rows affected (0.04 sec)
 
mysql> select * from runoob_transaction_test;
Empty set (0.01 sec)
 
mysql> begin;  # 开始事务
Query OK, 0 rows affected (0.00 sec)
 
mysql> insert into runoob_transaction_test value(5);
Query OK, 1 rows affected (0.01 sec)
 
mysql> insert into runoob_transaction_test value(6);
Query OK, 1 rows affected (0.00 sec)
 
mysql> commit; # 提交事务
Query OK, 0 rows affected (0.01 sec)
 
mysql>  select * from runoob_transaction_test;
+------+
| id   |
+------+
| 5    |
| 6    |
+------+
2 rows in set (0.01 sec)
 
mysql> begin;    # 开始事务
Query OK, 0 rows affected (0.00 sec)
 
mysql>  insert into runoob_transaction_test values(7);
Query OK, 1 rows affected (0.00 sec)
 
mysql> rollback;   # 回滚
Query OK, 0 rows affected (0.00 sec)
 
mysql>   select * from runoob_transaction_test;   # 因为回滚所以数据没有插入
+------+
| id   |
+------+
| 5    |
| 6    |
+------+
2 rows in set (0.01 sec)
```

# binlog日志

binlog日志为记录数据库操作的DML日志,需要打开才能记录,但是要注意记录文件会增长

## 打开方法

```
vim /etc/my.cnf

[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock

log-bin=mysql-bin

```

增加 `log-bin=mysql-bin`,然后重启数据库进程

可以在mysql中查看是否打开

```
MariaDB [(none)]> show variables like '%log_bin%';
+---------------------------------+-------+
| Variable_name                   | Value |
+---------------------------------+-------+
| log_bin                         | ON    |
| log_bin_trust_function_creators | OFF   |
| sql_log_bin                     | ON    |
+---------------------------------+-------+
```

## 查看日志文件方法

### 从客户端中查看

```
mysql> show binlog events;  # 查看第一个binlog文件

mysql> show binlog events in 'mysql-bin.000002'; # 查看指定文件

mysql> show master status\G  # 查看当前正在写入的binlog文件

mysql> show binary logs;  # 获取binlog文件列表
```

### 用mysqlbinlog工具查看

```
mysqlbinlog --start-position=2098 --stop-position=2205 -d hadoop /var/lib/mysql/mysql-bin.000001  #hadoop是库名

mysqlbinlog --start-datetime='2016-08-02 00:00:00' --stop-datetime='2016-08-03 23:01:01' -d hadoop /var/lib/mysql/mysql-bin.000001
```


# 常用定位

## 查看正在运行的SQL

```
mysql> select * from PROCESSLIST where info is not null
```

## 查看tmp table大小

```
mysql> show variables like '%table_size%';
+---------------------+-----------+
| Variable_name       | Value     |
+---------------------+-----------+
| max_heap_table_size | 16777216 |
| tmp_table_size      | 16777216 |
+---------------------+-----------+
```

## 查看整个状态

```
mysql> show status;
```

## 查看连接数

查看当前所有连接的详细资料:

```
./mysqladmin -uadmin -p -h10.140.1.1 processlist
```

只查看当前连接数(Threads就是连接数.):

```
./mysqladmin  -uadmin -p -h10.140.1.1 status
```

## 忘记密码

先stop数据库进程,然后采用安全模式启动,进入数据库再重新设置密码

```
systemctl stop mariadb
mysqld_safe --skip-grant-tables &
```

```
use mysql;
update user set password=PASSWORD("newpass")where user="root";
flush privileges;
```

然后重启mariadb服务
