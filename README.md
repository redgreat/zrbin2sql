# zrbin2sql工具介绍
zrbin2sql 是一个用于解析和转换 MySQL 二进制日志（binlog）的工具。它可以将二进制日志文件中记录的数据库更改操作（如插入、更新、删除）转换为反向的 SQL 语句，以便进行数据恢复。其运行模式需二进制日志设置为 ROW 格式。

主要功能和特点包括：

 - 解析二进制日志：能够解析 MySQL 的二进制日志文件，并还原出其中的 SQL 语句；

 - 生成可读的 SQL：生成原始 SQL 和反向 SQL；

 - 支持过滤和筛选：可以根据时间范围、表、DML操作等条件来过滤出具体的误操作 SQL 语句；

 - 支持多线程并发解析binlog事件；

 - 支持UPDATE语句反向生成REPLACE语句，支持将UPDATE语句中SET为NULL值的单独还原。

### 原理

调用官方 https://python-mysql-replication.readthedocs.io/ 库来实现，通过指定的时间范围，转换为timestamp时间戳，将整个时间范围平均分配给每个线程。

由于 BinLogStreamReader 并不支持指定时间戳来进行递增解析，固在每个任务开始之前，使用上一个任务处理过的 binlog_file 和 binlog_pos，这样后续的线程就可以获取到上一个线程处理过的 binlog 文件名和 position，然后进行后续的并发处理。

假设开始时间戳 start_timestamp 是 1625558400，线程数量 num_threads 是 4，整个时间范围被平均分配给每个线程。那么，通过计算可以得到以下结果：

    对于第一个线程（i=0），start_time 是 1625558400。
    
    对于第二个线程（i=1），start_time 是 1625558400 + time_range。
    
    对于第三个线程（i=2），start_time 是 1625558400 + 2 * time_range。
    
    对于最后一个线程（i=3），start_time 是 1625558400 + 3 * time_range。
    
这样，每个线程的开始时间都会有所偏移，确保处理的时间范围没有重叠，并且覆盖了整个时间范围。最后，将结果保存在一个列表里，并对列表做升序排序，取得最终结果。


### 使用

##### 当出现误操作时，只需指定误操作的时间段，其对应的binlog文件（通常你可以通过show master status得到当前的binlog文件名）以及刚才误操作的表，和具体的DML命令，比如update或者delete。

工具运行时，首先会进行MySQL的环境检测（if binlog_format != 'ROW' and binlog_row_image != 'FULL'），如果不同时满足这两个条件，程序直接退出。

工具运行后，会在当前目录下生成一个{db}_{table}_recover.sql文件，保存着原生SQL（原生SQL会加注释） 和 反向SQL，如果想将结果输出到前台终端，可以指定--print选项。

如果你想把update操作转换为replace，指定--replace选项即可，同时会在当前目录下生成一个{db}_{table}_recover_replace.sql文件。

![图片](https://github.com/hcymysql/reverse_sql/assets/19261879/b06528a6-fbff-4e00-8adf-0cba19737d66)

MySQL 最小化用户权限：

```
> GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO `yourname`@`%`;

> GRANT SELECT ON `test`.* TO `yourname`@`%`;
```

### 恢复

在{db}_{table}_recover.sql文件中找到你刚才误操作的DML语句，然后在MySQL数据库中执行逆向工程后的 SQL 以恢复数据。

如果{db}_{table}_recover.sql文件的内容过多，也可以通过awk命令进行分割，以便更容易进行排查。
```
shell> awk '/^-- SQL执行时间/{filename = "output" ++count ".sql"; print > filename; next} {print > filename}' test_t1_recover.sql
```

不支持drop和truncate操作，因为这两个操作属于物理性删除，需要通过历史备份进行恢复。

参照，感谢：

[reverse_sql](https://github.com/hcymysql/reverse_sql)

[binlog2sql](https://github.com/michael-liumh/binlog2sql)

[binlog2sql](https://github.com/danfengcao/binlog2sql)