# TDSQL 开发指南（V2R130）
TDSQL通过Proxy接口提供和mysql兼容的连接方式，用户通过IP地址、端口号以及用户名、密码连接TDSQL系统：

	mysql -h10.231.136.34 -P3306 -utest12 -ptest123
	
TDSQL系统兼容mysql的语法，但是不支持3. 4.0以下的版本以及压缩协议等，SSL加密传输在下个版本（V2R140）支持

同时对于group-sharding版本有一些区别和限制，用户在使用中必须注意，具体如下：

## 总体概括

在GS模式下，TDSQL实现水平分表，业务只需在创建表的时候指定一个分表字段，后续操作对业务透明，由TDSQL负责数据的路由以及汇总。

	提供了灵活的读写分离模式
	支持sum，count，avg，min，max等聚合函数，全局的order by，group by操作；
	支持单set的join，基于分组以及小表在一定程度上实现跨set的join；
	支持预处理协议；
	支持全局唯一字段；
	提供特定的sql查询整个集群的配置和状态；
	支持分布式事务
	支持两级分区

具体参考使用场景。

---

用户权限相关限制：

	1：目前版本不能通过proxy进行用户权限相关的设置，需要使用单独的工具


不支持MySQL的一些特性：

	1: 自定义数据类型、自定义函数
	2：视图、存储过程、触发器，游标
	3：外键，自建分区
	4：复合语句，如BEGIN END，LOOP等
	4：子查询，having字句，（支持带shardkey的derived table）

数据库状态信息限制：

	1：对于一般的查询数据库状态信息的sql，proxy会随机发到某个set，这样如果查询统计信息的话，就会不准确
	2：查询系统表也是随机发到某个set


## 使用场景

### 建表

普通的分表创建时必须在最后面指定shardkey的值，该值为表中的一个字段名字，会用于后续sql的路由选择：

	mysql> create table test1 ( a int key , b int, c char(20) ) shardkey=a;
	Query OK, 0 rows affected (0.07 sec)

shardkey字段有如下限定要求：

	1.如存在主键或者唯一索引，则shardkey字段必须是主键以及所有唯一索引的一部分
	2.shardkey字段的类型必须是int,bigint,smallint/char/varchar
	3.shardkey字段的值不应该有中文，网关不会转换字符集，所以不同字符集可能会路由到不同的分区
	4.不要update shardkey字段的值
	5.shardkey=a 放在sql的最后面
	6.访问数据尽量都能带上shardkey字段

groupshard支持建小表（广播表），此时该表在所有set中都是全量数据，这个主要方便于跨set的join操作，同时通过分布式事务保证修改操作的原子性，使得所有set的数据是完全一致的

	mysql> create table global_table ( a int, b int key) shardkey=noshardkey_allset;
	Query OK, 0 rows affected (0.06 sec)

groupshard同时支持建立普通的表，语法和mysql完全一样，此时该表的数据全量存在第一个set中，所有该类型的表都放在第一个set中：

	mysql> create table noshard_table ( a int, b int key);
	Query OK, 0 rows affected (0.02 sec)


### 普通sql
select：最好带上shardkey字段，否则就需要全表扫描，然后网关进行结果集聚合，对性能影响很大：

	mysql> select * from test1 where a=2;
	+------+------+---------+
	| a    | b    | c       |
	+------+------+---------+
	|    2 |    3 | record2 |
	|    2 |    4 | record3 |
	+------+------+---------+
	2 rows in set (0.00 sec)
	
insert/replace：字段必须包含shardkey，否则会拒绝执行该sql，因为Proxy不知道把该sql发往哪个后端数据库：

	mysql> insert into test1 (b,c) values(4,"record3");
	ERROR 1105 (07000): Proxy Warning - sql have no shardkey
	mysql> insert into test1 (a,c) values(4,"record3");
	Query OK, 1 row affected (0.01 sec)
	
delete/update：为了安全考虑，执行该类sql的时候必须带有where条件，否则拒绝执行该sql命令：

	mysql> delete from test1;
	ERROR 1005 (07000): Proxy Warning - sql is not legal,tokenizer_gram went wrong
	mysql> delete from test1 where a=1;
	Query OK, 1 row affected (0.01 sec)

### 透传sql

在groupshard中，proxy会对sql进行语法解析，会有比较严格的限制，如果用户想在某个set中执行sql，可以使用透传sql的功能：

	mysql> select * from test1 where a in (select a from test1);
	ERROR 7009 (HY000): Proxy ERROR:proxy do not support such use yet
	mysql> /*set_1*/select * from test1 where a in (select a from test1);
	Empty set (0.00 sec)

具体语法：
	
	sets:set_1,set_2（set名字可以通过/*proxy*/show status查询）   
	sets:allsets   
	shardkey:10,支持透传sql到对应的一个或者多个set，透传到shardkey对应的set，

### JOIN
Group-sharding支持单个set内的join操作，单个set意味着在一个事务内的所有sql必须操作同一个set，因此必须指定shardkey字段

	mysql> create table test1 ( a int key, b int, c char(20) ) shardkey=a;
	Query OK, 0 rows affected (1.56 sec)
	
	mysql> create table test2 ( a int key, d int, e char(20) ) shardkey=a;
	Query OK, 0 rows affected (1.46 sec)
	
	mysql> insert into test1 (a,b,c) values(1,2,"record1"),(2,3,"record2");
	Query OK, 2 rows affected (0.02 sec)
	
	mysql> insert into test2 (a,d,e) values(1,3,"test2_record1"),(2,3,"test2_record2");
	Query OK, 2 rows affected (0.02 sec)
	
	mysql>  select * from test1 left join test2 on test1.a=test2.a;
	ERROR 7015 (HY000): Proxy ERROR:sql should send to only one backend
	mysql>  select * from test1 left join test2 on test1.a=test2.a where test1.a=1;
	+---+------+---------+---+------+---------------+
	| a | b    | c       | a | d    | e             |
	+---+------+---------+---+------+---------------+
	| 1 |    2 | record1 | 1 |    3 | test2_record1 |
	+---+------+---------+---+------+---------------+
	1 row in set (0.00 sec)


支持带条件的跨set inner join

	mysql>  select * from test1 join test2 on test1.a=test2.a;
	+---+------+---------+---+------+---------------+
	| a | b    | c       | a | d    | e             |
	+---+------+---------+---+------+---------------+
	| 1 |    2 | record1 | 1 |    3 | test2_record1 |
	| 2 |    3 | record2 | 2 |    3 | test2_record2 |
	+---+------+---------+---+------+---------------+
	2 rows in set (0.03 sec)


但有如下前置条件：
- 	必须是inner join
- 	inner join的条件必须是所有表的shardkey字段相等，支持on，using以及where格式


对于小表和noshard表相关的join，对shardkey没有限制：

	mysql> create table noshard_table ( a int, b int key);
	Query OK, 0 rows affected (0.02 sec)
	
	mysql> create table noshard_table_2 ( a int, b int key);
	Query OK, 0 rows affected (0.00 sec)
	
	mysql> select * from noshard_table,noshard_table_2;
	Empty set (0.00 sec)
	
	mysql> insert into noshard_table (a,b) values(1,2),(3,4);
	Query OK, 2 rows affected (0.00 sec)
	Records: 2  Duplicates: 0  Warnings: 0
	
	mysql> insert into noshard_table_2 (a,b) values(10,20),(30,40);
	Query OK, 2 rows affected (0.00 sec)
	Records: 2  Duplicates: 0  Warnings: 0
	
	mysql> select * from noshard_table,noshard_table_2;
	+------+---+------+----+
	| a    | b | a    | b  |
	+------+---+------+----+
	|    1 | 2 |   10 | 20 |
	|    3 | 4 |   10 | 20 |
	|    1 | 2 |   30 | 40 |
	|    3 | 4 |   30 | 40 |
	+------+---+------+----+
	4 rows in set (0.00 sec)



### 事务
Group-sharding支持分布式事务

为了使用分布式事务功能，在创建完集群之后连接任意一个proxy运行如下sql进行初始化：

	mysql> xa init;
	Query OK, 0 rows affected (0.03 sec)

	注意：该sql会创建xa.gtid_log_t，用户在后续使用中万勿对其进行任何操作。
	
新增sql命令：

	1)	select gtid()，获取当前分布式事务的gtid(事务的全局唯一性标识)，如果该事务不是分布式事务则返回空；
		gtid的格式：
			‘网关id’-‘网关随机值’-‘序列号’-‘时间戳’-‘分区号’，例如 c46535fe-b6-dd-595db6b8-25

	2)	select gtid_state(“gtid”)，获取“gtid”的状态，可能的结果有：
		a)	“COMMIT”，标识该事务已经或者最终会被提交
		b)	“ABORT”，标识该事务最终会被回滚
		c)  空，由于事务的状态会在一个小时之后清楚，因此有以下两种可能：
				1) 一个小时之后查询，标识事务状态已经清除
				2) 一个小时以内查询，标识事务最终会被回滚
		
	3) 运维命令：
		xa recover：向后端set发送xa recover命令，并进行汇总
		xa lockwait：显示当前分布式事务的等待关系（可以使用dot命令将输出转化为等待关系图）
		xa show：当前网关上正在运行的分布式事务
		
建议的sql编程方式：
	
	(python程序)			
		db = pymysql.connect(host=testHost, port=testPort, user=testUser, password=testPassword, database=testDatabase)
		cursor = db.cursor()
        try:
            cursor.execute("begin")
			
            #为一个账户Bob的余额减1
            query = "update t_user_balance set balance = balance - 1  where user='Bob' and balance>1)
            affected = cursor.execute(query)
			if affected == 0: #余额不足，回滚事务
				cursor.execute("rollback")
				return 
				
            #为一个账户John的余额加1
            query = "update t_user_balance set balance = balance + 1  where user='John')
            cursor.execute(query)
			
            # 为了安全起见，建议在这里执行‘select gtid()’获取当前事务的id值，便于后续跟踪事务的执行情况
			
			#提交事务
            cursor.execute("commit")
        except pymysql.err.MySQLError as e:
			# 发生故障，回滚事务
            cursor.execute("rollback")
				

	
### 全局唯一字段

Group-sharding支持一定意义上的自增字段，保证某个字段全局唯一，但是不保证单调递增，具体使用方法如下：

创建：

	mysql> create table auto_inc (a int,b int,c int auto_increment,d int,key auto(c),primary key p(a,d)) shardkey=d;
	Query OK, 0 rows affected (0.12 sec)

插入：

	mysql>  insert into shard.auto_inc ( a,b,d,c) values(1,2,3,0),(1,2,4,0);
	Query OK, 2 rows affected (0.05 sec)
	Records: 2  Duplicates: 0  Warnings: 0
	
	mysql> select * from shard.auto_inc;
	+---+------+---+---+
	| a | b    | c | d |
	+---+------+---+---+
	| 1 |    2 | 2 | 4 |
	| 1 |    2 | 1 | 3 |
	+---+------+---+---+
	2 rows in set (0.03 sec)


值得说明的是，如果在proxy调度切换、重启等过程中，自增长字段中间会有空洞，例如：

    mysql> insert into shard.auto_inc ( a,b,d,c) values(11,12,13,0),(21,22,23,0);
    Query OK, 2 rows affected (0.03 sec)
    mysql> select * from shard.auto_inc;
    +‐‐‐‐‐‐+‐‐‐‐‐‐+‐‐‐‐‐‐+‐‐‐‐‐‐+
    | a | b | c | d |
    +‐‐‐‐‐‐+‐‐‐‐‐‐+‐‐‐‐‐‐+‐‐‐‐‐‐+
    | 21 | 22 | 2002 | 23 |
    | 1 | 2 | 2 | 4 |
    | 1 | 2 | 1 | 3 |
    | 11 | 12 | 2001 | 13 |
    +‐‐‐‐‐‐+‐‐‐‐‐‐+‐‐‐‐‐‐+‐‐‐‐‐‐+
    4 rows in set (0.01 sec)
    
更改当前值

	alter table auto auto_increment=100

### 状态查询

通过sql可以查看proxy的配置以及状态信息，目前支持如下命令：

	mysql> /*proxy*/help;
	+-----------------------+-------------------------------------------------------+
	| command               | description                                           |
	+-----------------------+-------------------------------------------------------+
	| show config           | show config from conf                                 |
	| show status           | show proxy status,like route,shardkey and so on       |
	| set sys_log_level=N   | change the sys debug level N should be 0,1,2,3        |
	| set inter_log_level=N | change the interface debug level N should be 0,1      |
	| set inter_time_open=N | change the interface time debug level N should be 0,1 |
	| set sql_log_level=N   | change the sql debug level N should be 0,1            |
	| set slow_log_level=N  | change the slow debug level N should be 0,1           |
	| set slow_log_ms=N     | change the slow ms                                    |
	| set log_clean_time=N  | change the log clean days                             |
	| set log_clean_size=N  | change the log clean size in GB                       |
	+-----------------------+-------------------------------------------------------+
	10 rows in set (0.00 sec)
	
	mysql> /*proxy*/show config;
	+-----------------+--------------------+
	| config_name     | value              |
	+-----------------+--------------------+
	| version         | V2R120D001         |
	| mode            | group shard        |
	| rootdir         | /shard_922         |
	| sys_log_level   | 0                  |
	| inter_log_level | 0                  |
	| inter_time_open | 0                  |
	| sql_log_level   | 0                  |
	| slow_log_level  | 0                  |
	| slow_log_ms     | 1000               |
	| log_clean_time  | 1                  |
	| log_clean_size  | 1                  |
	| rw_split        | 1                  |
	| ip_pass_through | 0                  |
	+-----------------+--------------------+
	14 rows in set (0.00 sec)

	mysql> /*proxy*/show status;
	+-----------------------------+------------------------------------------------------------------------------+
	| status_name                 | value                                                                        |
	+-----------------------------+------------------------------------------------------------------------------+
	| cluster                     | group_1499858910_79548                                                       |
	| set_1499859173_1:ip         | 10.49.118.165:5025;10.175.98.109:5025@1@IDC_4@0,10.231.23.241:5025@1@IDC_2@0 |
	| set_1499859173_1:hash_range | 0---31                                                                       |
	| set_1499911640_3:ip         | 10.49.118.165:5026;10.175.98.109:5026@1@IDC_4@0,10.231.23.241:5026@1@IDC_2@0 |
	| set_1499911640_3:hash_range | 32---63                                                                      |
	| set                         | set_1499859173_1,set_1499911640_3                                            |
	| db_table:pavel.ff           | shardkey:a                                                                   |
	


同时proxy增强了explain的返回结果，显示proxy修改后的sql

	mysql> explain select * from test1;
	+------+-------------+-------+------+---------------+------+---------+------+------+-------+-----------------------------------------+
	| id   | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra | info                                    |
	+------+-------------+-------+------+---------------+------+---------+------+------+-------+-----------------------------------------+
	|    1 | SIMPLE      | test1 | ALL  | NULL          | NULL | NULL    | NULL |   16 |       | set_2,explain select * from shard.test1 |
	|    1 | SIMPLE      | test1 | ALL  | NULL          | NULL | NULL    | NULL |   16 |       | set_1,explain select * from shard.test1 |
	+------+-------------+-------+------+---------------+------+---------+------+------+-------+-----------------------------------------+
	2 rows in set (0.03 sec)

### 数据导入导出

支持mysqldump导出数据：

	mysqldump  --no-tablespaces --default-character-set=utf8 --hex-blob --skip-add-drop-table -c 
				--no-create-info  --skip-lock-tables -v --master-data=1 --single-transaction 
				shard sbtest1  -utest -h10.231.136.34 -P3336  -ptest123


如果要导入数据的话，提供专门的导入工具，完成load data outfile对应数据的导入：

	[tdengine@TENCENT64 ~/]$./load_data
	
	format:./load_data mode0/mode1 proxy_host proxy_port user password db_table shardkey_index file field_terminate filed_enclosed
	
	    example:./load_data mode1 10.231.136.34 3336 test test123 shard.table  1 '/tmp/datafile'  ' ' ''
	
	
	
	format:./load_data mode2 routers_file shardkey_index file field_terminate filed_enclosed
	
	    example:./load_data mode2 '/data/3336.xml' 1 '/tmp/datafile'  ' ' ''
	
	note:lines should terminated by \n
	note:field_terminate may have more than one char,filed_enclosed must have only one char,all can not have ',do not support escape char
	

### 预处理

Sql类型的支持：

- PREPARE Syntax
- EXECUTE Syntax

二进制协议的支持：

- COM_STMT_PREPARE
- COM_STMT_EXECUTE

例子：

	mysql> select * from test1;
	+---+------+
	| a | b    |
	+---+------+
	| 5 |    6 |
	| 3 |    4 |
	| 1 |    2 |
	+---+------+
	3 rows in set (0.03 sec)
	
	mysql> prepare ff from "select * from test1 where a=?";
	Query OK, 0 rows affected (0.00 sec)
	Statement prepared
	
	mysql> set @aa=3;
	Query OK, 0 rows affected (0.00 sec)
	
	mysql> execute ff using @aa;
	+---+------+
	| a | b    |
	+---+------+
	| 3 |    4 |
	+---+------+
	1 row in set (0.06 sec)

### 主备同步

支持通过proxy建主备同步，目前支持mariadb的GTID模式，下个版本（V2R140）支持mysql的GTID模式和文件偏移量模式

	set global gtid_slave_pos='0-1-xxxx';
	change master to master_host='10.187.143.218',master_port=24001, 
		master_user='repl',master_password='repl_pass', master_use_gtid=slave_pos; 
	start slave;

其中ip和端口指向对应的proxy


### 读写分离

TDSQL支持三种模式的读写分离。

- 网关开启语法解析的配置，通过语法解析过滤出用户的select读请求，默认把读请求直接发给备机；
- 通过增加slave注释标记，将指定的sql发往备机。即在sql中添加/*slave*/这样的标记，该sql会发送给备机；
- 由只读帐号发送的请求会根据配置的属性发给备机
	

### 子查询

TDSQL对于子查询这块支持比较有限，目前只支持带shardkey的derived table

	mysql> select a from (select * from test1) as t;
	ERROR 7012 (HY000): Proxy ERROR:sql should has one shardkey
	mysql> select a from (select * from test1 where a=1) as t;
	+---+
	| a |
	+---+
	| 1 |
	+---+
	1 row in set (0.00 sec)


### 两级分区

TDSQL只用HASH方式进行数据拆分，不利于删除特定条件的数据，如流水类型，删除旧的数据，为了解决这个问题，可以使用两级分区。

TDSQL支持range格式的两级分区，具体建表语法和mysql分区语法类似：

	CREATE TABLE employees (
	    id INT NOT NULL,
	    fname VARCHAR(30),
	    lname VARCHAR(30),
	    hired DATE NOT NULL DEFAULT '1970-01-01',
	    separated DATE NOT NULL DEFAULT '9999-12-31',
	    job_code INT,
	    store_id INT
	)  shardkey=id 	PARTITION BY RANGE ( YEAR(hired) ) (
	    PARTITION p0 VALUES LESS THAN (1991),
	    PARTITION p1 VALUES LESS THAN (1996),
	    PARTITION p2 VALUES LESS THAN (2017)，
	    PARTITION p2 VALUES LESS THAN (2018)
	);

注意：分区使用的是小于符号，因此如果要存储当年的数据的话（2017），需要创建<2018的分区，用户只需创建到当前的时间分区，TDSQL会自动增加后续分区，默认往后创建3个分区，以YEAR为例，TDSQL会自动创建2018，2019，2020的分区，后续也会自动增减

目前支持按年月日三种range 方式： RANGE ( YEAR(hired) ) ， RANGE ( MONTH(hired) ) ， RANGE ( DAY(hired) ) ，后续会支持LIST二级分区


### 函数支持


#### Control Flow Functions

Name | Description 
---|---|---
CASE	|Case operator 
IF()	|If/else construct 
IFNULL()	|Null if/else construct
NULLIF()	|Return NULL if expr1 = expr2


#### String Functions

Name | Description 
---|---|---
ASCII()|	Return numeric value of left-most character
BIN()|	Return a string containing binary representation of a number
BIT_LENGTH()|	Return length of argument in bits
CHAR()	|Return the character for each integer passed
CHAR_LENGTH()|	Return number of characters in argument
CHARACTER_LENGTH()|	Synonym for CHAR_LENGTH()
CONCAT()	|Return concatenated string
CONCAT_WS()	|Return concatenate with separator
ELT()	|Return string at index number
EXPORT_SET()|	Return a string such that for every bit set in the value bits, you get an on string and for every unset bit, you get an off string
FIELD()	|Return the index (position) of the first argument in the subsequent arguments
FIND_IN_SET()	|Return the index position of the first argument within the second argument
FORMAT()	|Return a number formatted to specified number of decimal places
FROM_BASE64()	|Decode to a base-64 string and return result
HEX()	|Return a hexadecimal representation of a decimal or string value
INSERT()	|Insert a substring at the specified position up to the specified number of characters
INSTR()	|Return the index of the first occurrence of substring
LCASE()	|Synonym for LOWER()
LEFT()	|Return the leftmost number of characters as specified
LENGTH()|	|Return the length of a string in bytes
LIKE	|Simple pattern matching
LOAD_FILE()|	Load the named file
LOCATE()|	Return the position of the first occurrence of substring
LOWER()	|Return the argument in lowercase
LPAD()	|Return the string argument, left-padded with the specified string
LTRIM()	|Remove leading spaces
MAKE_SET()|	Return a set of comma-separated strings that have the corresponding bit in bits set
MATCH	|Perform full-text search
MID()|	Return a substring starting from the specified position
NOT LIKE|	Negation of simple pattern matching
NOT REGEXP	|Negation of REGEXP
OCT()	|Return a string containing octal representation of a number
OCTET_LENGTH()|	Synonym for LENGTH()
ORD()|	Return character code for leftmost character of the argument
POSITION()	|Synonym for LOCATE()
QUOTE()|	Escape the argument for use in an SQL statement
REGEXP|	Pattern matching using regular expressions 
REPEAT()	|Repeat a string the specified number of times
REPLACE()	|Replace occurrences of a specified string
REVERSE()|	Reverse the characters in a string
RIGHT()|	Return the specified rightmost number of characters
RLIKE|	Synonym for REGEXP
RPAD()|	Append string the specified number of times
RTRIM()|	Remove trailing spaces
SOUNDEX()|	Return a soundex string
SOUNDS LIKE|	Compare sounds 
SPACE()	|Return a string of the specified number of spaces
STRCMP()|	Compare two strings
SUBSTR()|	Return the substring as specified
SUBSTRING()|	Return the substring as specified
SUBSTRING_INDEX()|	Return a substring from a string before the specified number of occurrences of the delimiter
TO_BASE64()	|Return the argument converted to a base-64 string
TRIM()	|Remove leading and trailing spaces
UCASE()	|Synonym for UPPER()
UNHEX()	|Return a string containing hex representation of a number
UPPER()	|Convert to uppercase
WEIGHT_STRING()	|Return the weight string for a string


#### Numeric Functions and Operators

Name | Description 
---|---|---
ABS()	|Return the absolute value
ACOS()	|Return the arc cosine
ASIN()	|Return the arc sine
ATAN()	|Return the arc tangent
ATAN2(), ATAN()	|Return the arc tangent of the two arguments
CEIL()	|Return the smallest integer value not less than the argument
CEILING()	|Return the smallest integer value not less than the argument
CONV()	|Convert numbers between different number bases
COS()	|Return the cosine
COT()	|Return the cotangent
CRC32()	|Compute a cyclic redundancy check value
DEGREES()	|Convert radians to degrees
DIV	|Integer division
/	|Division operator
EXP()	|Raise to the power of
FLOOR()	|Return the largest integer value not greater than the argument
LN()	|Return the natural logarithm of the argument
LOG()	|Return the natural logarithm of the first argument
LOG10()	|Return the base-10 logarithm of the argument
LOG2()	|Return the base-2 logarithm of the argument
-	|Minus operator
MOD()	|Return the remainder
%, MOD	|Modulo operator
PI()	|Return the value of pi
+	|Addition operator
POW()	|Return the argument raised to the specified power
POWER()	|Return the argument raised to the specified power
RADIANS()	|Return argument converted to radians
RAND()	|Return a random floating-point value
ROUND()	|Round the argument
SIGN()	|Return the sign of the argument
SIN()	|Return the sine of the argument
SQRT()	|Return the square root of the argument
TAN()	|Return the tangent of the argument
*	|Multiplication operator
TRUNCATE()	|Truncate to specified number of decimal places
-	|Change the sign of the argument


#### Date and Time Functions

Name | Description | 
---|---|---
ADDDATE()	|Add time values (intervals) to a date value
ADDTIME()	|Add time
CONVERT_TZ()	|Convert from one time zone to another
CURDATE()	|Return the current date
CURRENT_DATE(), CURRENT_DATE	|Synonyms for CURDATE()
CURRENT_TIME(), CURRENT_TIME	|Synonyms for CURTIME()
CURRENT_TIMESTAMP(), CURRENT_TIMESTAMP|	Synonyms for NOW()
CURTIME()	|Return the current time
DATE()	|Extract the date part of a date or datetime expression
DATE_ADD()	|Add time values (intervals) to a date value
DATE_FORMAT()	|Format date as specified
DATE_SUB()	|Subtract a time value (interval) from a date
DATEDIFF()	|Subtract two dates
DAY()	|Synonym for DAYOFMONTH()
DAYNAME()	|Return the name of the weekday
DAYOFMONTH()	|Return the day of the month (0-31)
DAYOFWEEK()	|Return the weekday index of the argument
DAYOFYEAR()	|Return the day of the year (1-366)
EXTRACT()	|Extract part of a date
FROM_DAYS()	|Convert a day number to a date
FROM_UNIXTIME()	|Format Unix timestamp as a date
GET_FORMAT()	|Return a date format string
HOUR()	|Extract the hour
LAST_DAY	|Return the last day of the month for the argument
LOCALTIME(), LOCALTIME	|Synonym for NOW()
LOCALTIMESTAMP, LOCALTIMESTAMP()	|Synonym for NOW()
MAKEDATE()	|Create a date from the year and day of year
MAKETIME()	|Create time from hour, minute, second
MICROSECOND()	|Return the microseconds from argument
MINUTE()	|Return the minute from the argument
MONTH()	|Return the month from the date passed
MONTHNAME()	|Return the name of the month
NOW()	|Return the current date and time
PERIOD_ADD()	|Add a period to a year-month
PERIOD_DIFF()	|Return the number of months between periods
QUARTER()	|Return the quarter from a date argument
SEC_TO_TIME()	|Converts seconds to 'HH:MM:SS' format
SECOND()	|Return the second (0-59)
STR_TO_DATE()	|Convert a string to a date
SUBDATE()	|Synonym for DATE_SUB() when invoked with three arguments
SUBTIME()	|Subtract times
SYSDATE()	|Return the time at which the function executes
TIME()	|Extract the time portion of the expression passed
TIME_FORMAT()|	Format as time
TIME_TO_SEC()|	Return the argument converted to seconds
TIMEDIFF()	|Subtract time
TIMESTAMP()	|With a single argument, this function returns the date or datetime expression; with two arguments, the sum of the arguments
TIMESTAMPADD()|	Add an interval to a datetime expression
TIMESTAMPDIFF()|	Subtract an interval from a datetime expression
TO_DAYS()	|Return the date argument converted to days
TO_SECONDS()|	Return the date or datetime argument converted to seconds since Year 0
UNIX_TIMESTAMP()|	Return a Unix timestamp
UTC_DATE()|	Return the current UTC date
UTC_TIME()|	Return the current UTC time
UTC_TIMESTAMP()	|Return the current UTC date and time
WEEK()|	Return the week number
WEEKDAY()	|Return the weekday index
WEEKOFYEAR()|	Return the calendar week of the date (1-53)
YEAR()	|Return the year
YEARWEEK()|	Return the year and week

