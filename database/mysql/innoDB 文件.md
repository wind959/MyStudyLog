# 文件

这里主要介绍 MySQL 和 InnoDB 中所使用到的文件类型，具体包含了下面的六种文件：

* 参数文件
* 日志文件
* socket 文件
* pid 文件
* 表结构文件
* 存储引擎文件

## 参数文件

MySQL 在启动的时候需要读一个配置文件，该文件用来寻找数据库的各种文件所在位置，以及某些初始化的参数。默认情况下 MySQL 实例会按照一定的顺序在指定位置读取(使用 `mysql --help |grep my.cnf`)。当然在 MySQL 中如果没有参数文件的话，参数值就是根据 MySQL 编译时的默认值来给的。

在 MySQL 中参数就是一个个键值对，我们查看每一个参数都可以用命令 SHOW VARIABLES 来查看，**并且 MySQL 是没有所谓隐藏参数的**。MySQL 中有两种类型的参数：动态参数和静态参数。动态参数的话意味着在 MySQL 运行实例的过程中是可以被更改的，而静态的参数在实例生命周期中是不能被修改的。在实例中使用 SET 命令就可以进行修改了，其中还有 global 和 session 限制设置的参数是全局的还是在会话期的。不过中途修改的参数并不会修改最终的参数文件。

## 日志文件

在 innoDB 中一般有一下四种日志：

* 错误日志
* 二进制日志
* 慢查询日志
* 查询日志

### 错误日志

错误日志对 MySQL 执行过程都有记录。它记录了所有的错误信息，也记录了警告信息或者正确信息，在默认情况下错误文件名为服务器的主机名。所以如果 MySQL 执行出问题了，那么就必须查找错误日志文件。

### 慢查询日志

慢查询日志是为了定位某些可能存在问题的 SQL 语句，从 SQL 语句层面优化。慢查询日志在默认情况下并不开启，需要手工调整该参数。MySQL 启动时设置一个阈值，将运行事件超过这个阈值的语句都记录在了慢查询日志中，这就方便未来查看是否有可以优化的语句。

但是慢查询日志可能会出现大小不断增加的情况，这样就很难直接查找，所以 MySQL 数据库提供的 mysqldumpslow 命令进行快速查询。

### 查询日志

查询日志中记录了所有 MySQL 数据库的请求信息，无论这些请求是否得到了执行。这个文件的默认文件名是：主机名.log。

### 二进制日志

二进制日志记录的是对 MySQL 数据库执行更改的所有操作，注意一些并不会对数据库数据有影响的操作是不会记录的，而有影响但是并没有修改数据的操作也会被记录到二进制日志中。

二进制日志主要有如下的作用：

* 恢复：数据库的数据恢复要用到二进制日志；
* 复制：当  MySQL 从服务器需要复制主服务器上的数据时，需要同步数据；
* 审计：判断是否对数据库进行了注入攻击；

默认的二进制日志文件名时主机名，后缀名是二进制日志的序列号，并且这个功能在默认状态下是没有启动的。影响系统中的二进制日志的参数有很多，其中有一些参数值得说一下：
binlog_cache_size 表示二进制日志缓存，当使用事务表引擎时，所有未提交的二进制日志文件都会被记录到一个缓存中，等待提交后写入二进制文件中。sync_binlog 参数表示每写缓冲多少次就同步到磁盘，这是因为二进制日志不会每次写都同步到磁盘。binlog_format 参数比较关键，它影响了记录二进制日志的格式，该参数的取值包括 STATEMENT、ROW 和 MIXED。STATEMENT 记录的是逻辑 SQL 语句，ROW 记录的是表的行更改信息(可能需要更大的磁盘空间)，MIXED 则默认使用 STATEMENT，只有部分情况下会使用 ROW。

那么如果需要查看二进制日志，则必须使用 mysqlbinlog 工具来查看。

### 套接字文件

在 UNIX 系统中本地连接 MySQL 可以使用套接字方式，这会需要一个 socket 文件

### pid 文件

当 MySQL 实例启动时会将自己的进程 ID 写入一个文件中，这个文件默认在数据库目录下，名字是主机名.pid

### 表结构定义

无论是哪一种存储引擎，MySQL 都有一个以 frm 为后缀名的文件，这个文件记录了某个表的表结构。当然这种类型的文件还存储了视图的定义。

### InnoDB 存储引擎文件

这是和 InnoDB 相关的一些文件，包括 InnoDB 使用的表空间文件、重做日志文件。

InnoDB 采用了将存储的数据按照表空间来进行存放的设计，在默认下目录中都会有一个 ibdata1 的文件。这个文件是默认的表空间，假设设置了参数 innodb_data_file_path 参数，那么所有数据都会存到这个共享表空间中，如果设置了参数 innodb_file_per_table，则每一个表都会有一个独立表空间，名字是表名.ibd。这些单独表空间文件仅存储该表的数据、索引和插入缓冲等，其他的信息还是会存放在共享表空间中。

在默认的情况下，在 InnoDB 存储引擎中的数据目录下都会有两个名为 ib_logfile0 和 ib_log_file1 的文件，这些文件记录了事务日志。主要的作用就是为了保证事务的完整性。每个引擎至少有 1 个重做日志文件组，每个文件组下至少有两个重做日志文件。日志组中的每个重做日志文件的大小一致，并且以循环写入的方式运行。注意这个重做日志只包含 InnoDB 存储引擎中的操作记录。而且重做日志记录的是每个页的物理状况，而不是逻辑日志。