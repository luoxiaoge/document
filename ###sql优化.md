###sql优化

1，分析SQL执行频率

MySQL 客户端连接成功后，通过 show [session|global]status 命令可以提供服务器状态信 息，也可以在操作系统上使用 mysqladmin extended-status 命令获得这些消息。show [session|global] status 可以根据需要加上参数“session”或者“global”来显示 session 级（当 前连接）的统计结果和 global 级（自数据库上次启动至今）的统计结果。如果不写，默认使 用参数是“session”    

命令： show status like 'Com_%';    

Com_xxx 表示每个 xxx 语句执行的次数，我们通常比较关心的是以下几个统计参数。  Com_select：执行 select 操作的次数，一次查询只累加 1。 

 Com_insert：执行 INSERT 操作的次数，对于批量插入的 INSERT 操作，只累加一次。 

 Com_update：执行 UPDATE 操作的次数。  Com_delete：执行 DELETE 操作的次数。 上面这些参数对于所有存储引擎的表操作都会进行累计。下面这几个参数只是针对 InnoDB 存储引擎的，累加的算法也略有不同。  Innodb_rows_read：select 查询返回的行数。

  Innodb_rows_inserted：执行 INSERT 操作插入的行数。 

 Innodb_rows_updated：执行 UPDATE 操作更新的行数。

  Innodb_rows_deleted：执行 DELETE 操作删除的行数。 通过以上几个参数，可以很容易地了解当前数据库的应用是以插入更新为主还是以查询 操作为主，以及各种类型的 SQL 大致的执行比例是多少。对于更新操作的计数，是对执行 次数的计数，不论提交还是回滚都会进行累加。 对于事务型的应用，通过 Com_commit 和 Com_rollback 可以了解事务提交和回滚的情况， 对于回滚操作非常频繁的数据库，可能意味着应用编写存在问题。 此外，以下几个参数便于用户了解数据库的基本情况。

  Connections：试图连接 MySQL 服务器的次数。 

 Uptime：服务器工作时间。 

 Slow_queries：慢查询的次数。    

2，定位执行效率较低的 SQL 语句    

①通过慢查询日志定位那些执行效率较低的 SQL 语句，用--log-slow-queries[=file_name]选 项启动时，mysqld 写一个包含所有执行时间超过 long_query_time 秒的 SQL 语句的日志 文件    

②慢查询日志在查询结束以后才纪录，所以在应用反映执行效率出现问题的时候查询慢查 询日志并不能定位问题，可以使用 show processlist 命令查看当前 MySQL 在进行的线程， 包括线程的状态、是否锁表等，可以实时地查看 SQL 的执行情况，同时对一些锁表操 作进行优化    

3，通过 EXPLAIN 分析低效 SQL 的执行计划    

可以通过 EXPLAIN 或者 DESC 命令获取 MySQL 如何执行 SELECT 语句的信息，包括在 SELECT 语句执行过程中表如何连接和连接的顺序， 

每个列的简单解释如下： 

 select_type：表示 SELECT 的类型，常见的取值有 SIMPLE（简单表，即不使用表连接 或者子查询）、PRIMARY（主查询，即外层的查询）、UNION（UNION 中的第二个或 者后面的查询语句）、SUBQUERY（子查询中的第一个 SELECT）等。 

 table：输出结果集的表。 

 type：表示表的连接类型，性能由好到差的连接类型为 system（表中仅有一行，即 常量表）、const（单表中最多有一个匹配行，例如 primary key 或者 unique index）、 eq_ref（对于前面的每一行，在此表中只查询一条记录，简单来说，就是多表连接 中使用 primary key或者 unique index）、ref （与 eq_ref 类似，区别在于不是使用 primary key 或者 unique index，而是使用普通的索引）、ref_or_null（与 ref 类似，区别在于 条件中包含对 NULL 的查询）、index_merge(索引合并优化)、unique_subquery（in 的后面是一个查询主键字段的子查询）、index_subquery（与 unique_subquery 类似， 区别在于 in 的后面是查询非唯一索引字段的子查询）、range（单表中的范围查询）、 index（对于前面的每一行，都通过查询索引来得到数据）、all（对于前面的每一行，都通过全表扫描来得到数据）。         

 possible_keys：表示查询时，可能使用的索引。 

 key：表示实际使用的索引。 

 key_len：索引字段的长度。 

 rows：扫描行的数量。 

 Extra：执行情况的说明和描述    

4，确定问题并采取相应的优化措施    

① 索引问题 ：

如果索引正在工作，Handler_read_key 的值将很高，这个值代表了一个行被索引值读的 次数，很低的值表明增加索引得到的性能改善不高，因为索引并不经常使用。 Handler_read_rnd_next 的值高则意味着查询运行低效，并且应该建立索引补救。这个值 的含义是在数据文件中读下一行的请求数。如果正进行大量的表扫描， Handler_read_rnd_next 的值较高，则通常说明表索引不正确或写入的查询没有利用索引，具 体如下。    

命令：show status like 'Handler_read%';    





######优化具体类型语句

1，优化insert 语句

当进行数据 INSERT 的时候，可以考虑采用以下几种优化方式。 

 如果同时从同一客户插入很多行，尽量使用多个值表的 INSERT 语句，这种方式将大大 缩减客户端与数据库之间的连接、关闭等消耗，使得效率比分开执行的单个 INSERT 语 句快(在一些情况中几倍)。下面是一次插入多值的一个例子： insert into test values(1,2),(1,3),(1,4)… 

 如果从不同客户插入很多行，能通过使用 INSERT DELAYED 语句得到更高的速度。 DELAYED 的含义是让 INSERT 语句马上执行，其实数据都被放在内存的队列中，并没有 真正写入磁盘，这比每条语句分别插入要快的多；LOW_PRIORITY 刚好相反，在所有其 他用户对表的读写完后才进行插入； 

 将索引文件和数据文件分在不同的磁盘上存放（利用建表中的选项）；  如果进行批量插入，可以增加 bulk_insert_buffer_size 变量值的方法来提高速度，但是， 这只能对 MyISAM 表使用；

  当从一个文本文件装载一个表时，使用 LOAD DATA INFILE。这通常比使用很多 INSERT 语 句快 20 倍。    

2，优化 GROUP BY 语句    

默认情况下，MySQL 对所有 GROUP BY col1，col2....的字段进行排序。这与在查询中指定 ORDER BY col1，col2...类似。因此，如果显式包括一个包含相同的列的 ORDER BY 子句，则 对 MySQL 的实际执行性能没有什么影响。 如果查询包括 GROUP BY 但用户想要避免排序结果的消耗，则可以指定 ORDER BY NULL 禁止排序    



3,优化orderBy 语句

但是在以下几种情况下则不使用索引： SELECT * FROM t1 ORDER BY key_part1 DESC, key_part2 ASC； --order by 的字段混合 ASC 和 DESC SELECT * FROM t1 WHERE key2=constant ORDER BY key1； --用于查询行的关键字与 ORDER BY 中所使用的不相同 SELECT * FROM t1 ORDER BY key1, key2； --对不同的关键字使用 ORDER BY：    



4，优化嵌套查询    

MySQL 4.1 开始支持 SQL 的子查询。这个技术可以使用 SELECT 语句来创建一个单列的查 询结果，然后把这个结果作为过滤条件用在另一个查询中。使用子查询可以一次性地完成很 多逻辑上需要多个步骤才能完成的 SQL 操作，同时也可以避免事务或者表锁死，并且写起 来也很容易。但是，有些情况下，子查询可以被更有效率的连接（JOIN）替代    



5，MySQL 如何优化 OR 条件    

对于含有 OR 的查询子句，如果要利用索引，则 OR 之间的每个条件列都必须用到索引； 如果没有索引，则应该考虑增加索引。    



小结：

SQL 优化问题是数据库性能优化最基础也是最重要的一个问题，实践表明很多数据库性能问 题都是由不合适的 SQL 语句造成。本章通过实例描述了 SQL 优化的一般过程，从定位一个 有性能问题的 SQL 语句到分析产生性能问题的原因，最后到采取什么措施优化 SQL 语句的 性能。另外还介绍了优化 SQL 语句经常需要考虑的几个方面，比如索引、表分析、排序等    



