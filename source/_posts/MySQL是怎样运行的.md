---
title: MySQL是怎样运行的
---
# MySQL是怎样运行的
@[TOC](目录)
# 字符集和比较规则
mysql中支持多种字符集，每种字符集支持几种比较规则；
==*查询的时候发现排序结果不符合预期，可以看下比较规则是否符合预期*==
![一些常用的字符集及比较规则](https://img-blog.csdnimg.cn/ea85e7be4cb149f7a4275960f6f5560c.png)其中default collection表示默认的比较规则；（后缀_ci表示比较时不区分大小写）
		maxlen表示所占字符大小；
		

如：

 - utf8mb3:使用1～3个字节表示字符；
 - utf8mb4:使用1～4个字节表示字符；
我们平时说的utf8是指utf8mb3；

对于字符集和比较规则的指定可以在四个级别进行指定：

 - 系统级别
 - 数据库级别
 - 表级别
 - 列级别

```bash
CHARACHER SET:  设定字符集
COLLATE：       设定比较规则
```

4个级别字符集和比较规则的联系：

 - 如果创建或修改列时没有显式的指定字符集和比较规则，则该列默认用表的字符集和比较规则
 - 如果创建或修改表时没有显式的指定字符集和比较规则，则该表默认用数据库的字符集和比较规则
 - 如果创建或修改数据库时没有显式的指定字符集和比较规则，则该数据库默认用服务器的字符集和比较规则

对于客户端与mysql服务器的交互过程中存在多次的字符转换，如下图所示：
![交互过程字符转换流程](https://img-blog.csdnimg.cn/f6cff671cd624accb6baef4371c7e34a.png)
要保证character_set_client与os的字符集编码一致；character_set_client与os的字符集编码一致；
==通常习惯将character_set_client、character_set_connection、character_set_client三个系统变量设置成一致，防止多次不必要的字符集转换==

# 行存储格式
mysql中内置了4种行存储格式，compact、redundant、dynamic、compressed；
## compact格式
![compact格式存储](https://img-blog.csdnimg.cn/bb368f0496fa41febd32550859c25d39.png)
==变长字段长度列表==记录了本行中存储变长字段的列的长度，按照==逆序==存储，每一项占==1～2字节==；
**允许存储的最大字节数（长度上限*每个字的最大长度）>255，且实际的存储数超过127字节时占2个字节，否则只占1个字节；**

==NULL值列表==记录改行中允许是NULL的列，==逆序存储==，该字段必须占整个字节，其余位补0，若没有可以是NULL的列，则无该结构；（这样NULL值的列就可以不在真实数据部分存储了，可以节省空间）

==记录头信息==，这部分用到的时候进行查阅～
![compact记录头信息](https://img-blog.csdnimg.cn/b9261258f6f34918af73db3cd7714494.png)
==隐藏列==
|name|是否必须|占用空间 |描述 |
|--|--|--|--|
|  row_id| 否 | 6字节| 行ID，唯一标识一条记录（表中无primary key、无unique key时自动添加）|
|  transaction_id|是  | 6字节| 事务ID|
|  roll_pointer|  是| 7字节| 回滚指针|

==行溢出==
mysql中除了TEXT、BLOBs这种大对象类型之外，一行所有的列（不包含隐藏列和记录头信息）占用的字节长度加起来不能超过65535个字节；
这个65535字节中不能全部都用来存储数据，其中包括三部分：

 1. 真实数据
 2. 真实数据长度（1～2字节）
 3. NULL值标识（0～1字节）

**exp**
前提：使用gdk字符集，有NOT NULL标识，该行中只有一列，VARCHAR(M)
gdk字符集中一个字符占2个字节，(65535-2)/2=32766.5，也就是说M最大取值是32766；

在mysql中规定==一个页中至少存放两行记录==，一个页的大小是16kb，每个页中除了存放真实数据之外，还需要存储136个字节的额外信息，同时每条记录需要的额外信息是27字节；
即：136+2*(n+27)>16384
解得n>8098；
也就是说当一个列中存储数据不大于8098字节时就不会发生溢出；
**注意以上情况讨论的是理想情况下，一行中只有一列的情况，真实情况会有不同；**

![compact行溢出](https://img-blog.csdnimg.cn/96938fcbafbd485d8f4e5b8e8b34ebb5.png)

发生行溢出的时候，compact会存储真实数据的前768字节，剩余的数据存储在溢出页中；

## redundant
![redundant存储格式](https://img-blog.csdnimg.cn/fdd675f101354bf694a685eaf985e434.png)

这种格式是mysql5.0前的格式，与compact不同的是，字段长度偏移列表存储的是每个字段的结束地址，没有NULL值列表；

溢出形式与compact相同；
## dynamic和compressed
这两种方式的格式与compact十分类似，只是在行溢出的时候处理方式不同；
对于dynamic的行溢出，不会存储前768字节的真实数据，而是直接存储溢出页的地址；
![dyanmic行溢出](https://img-blog.csdnimg.cn/22fe2129d905435b8f926696a4dbb1c5.png)
对于compressed，会使用压缩算法对页面进行压缩；

# innoDB数据页

页结构：
|名称|中文名  |占用空间大小|简单描述|
|--|--|--|--|
| File Header | 文件头部 |38字节|页的一些通用信息|
| Page Header | 页面头部 |56字节|数据页专有的一些信息|
| Infimum + Supremum |  最小记录和最大记录|26字节|两个虚拟的行记录|
| User Records | 用户记录 |不确定|实际存储的行记录内容|
| Free Space | 空闲空间 |不确定|页中尚未使用的空间|
| Page Directory | 页面目录 |不确定|页中的某些记录的相对位置|
|  Tile Trailer| 文件尾部 |8字节|校验页是否完整|


![在这里插入图片描述](https://img-blog.csdnimg.cn/5c9937594c6e4dc2ab961b44ba34ad62.png)
innoDB会把页中的记录进行分组，结构如图所示；

| 名称 | 中文名 | 含义|
|--|--|--|
| delete_mask | 删除标志位 |0:未删除  1:已删除|
| min_res_mask | 最小记录标识 |1:最小记录|
| n_owned | 该组有几条记录 |最小记录所在分组：1条记录； 最大记录所在分组：1～8条记录； 其余分组：4～8条记录；|
| heap_no | 记录在本页中的位置 |系统默认添加了最小记录和最大记录，如图上方所示|
| record_type | 记录类型 |0:普通记录 1:B+树非页节点记录 2:最小记录 3:最大纪录|
| next_record | 下条记录的偏移量 ||

查找时，会先使用二分法查找记录所在的分组；
再在分组中通过next_record遍历寻找具体记录；


# 索引讨论
## inndoDB索引
![B+树索引结构](https://img-blog.csdnimg.cn/06ba6a4095dc4fa78779b97efe619c6c.png)
innoDB的索引结构如上图所示（主键索引），其中record_type就是上面说到的记录结构中的record_type，有以下几种类型：

 - 0:普通的用户记录；
 - 1:目录项记录；
 -  2:最小记录；
 - 3:最大记录
其中只有叶子节点存储用户数据，其他节点存储主键值和页号，这种索引结构叫做**聚簇索引**，有以下两个特点：
 - 使用记录主键值的大小进行记录和页的排序；
 - B+树的叶子节点存储的是完整的用户记录；
innoDB会为每一张表都**自动建立**一个上述格式的**聚簇索引**；

而我们也可以通过语句建立二级索引，有以下几种索引：

 1. 普通索引；
 2. 联合索引；
 3. 唯一索引；

建立&删除索引的语句如下：

```sql
ALTER TABLE 表名 ADD [INDEX/KEY] 索引名 (cols...)

CREATE TABLE 表名（
	c1 INT,
	c2 INT,
	c3 CHAR(1),
	PRIMARY KEY(c1),
	INDEX idx_2(c2,c3)
）;

ALTER TABLE 表名 DROP [INDEX/KEY] 索引名
```

若为表中的c2、c3列建立联合索引，则索引结构如下图所示：
![联合索引结构](https://img-blog.csdnimg.cn/5e68f269342b40099a206fb3f7fe2121.png)可以看到，叶子节点存储c2、c3的值和主键值，其他节点则存储c2、c3、主键值和页号，在存储的时候会先按c2排序，若c2值相同则按c3排序，使用二级索引进行查询时就会涉及到回表操作；

接下来我们讨论一下**回表**操作：
可能出现回表操作的场景：

 1. 使用二级索引，查询的列不仅是索引覆盖的列；
 
由于访问二级索引是顺序IO，而访问聚簇索引是随机IO，因此需要回表的记录越多，性能就越低，所以查询优化器会对表中的记录计算一些统计数据，然后利用这些统计数据根据查询的条件计算一下需要回表的记录书，需要回表的记录越多，就越倾向于使用*全表扫描*，反之倾向于使用*二级索引+回表*的方式；
==我们在查询时尽量使用覆盖索引==

*tips：*
 - 一个B+树索引的根节点建立后，位置就不会再改变； 
 - 内节点中目录项是唯一的，通过存储索引列的值和主键值保证；
 - 一个页面中至少存储2条记录；

介绍完索引的结构，切记也不要建立许多不必要的索引，因为每建立一个索引会带来时间和空间上的浪费：

 - 空间：每建立一个索引，就会建立一个B+树，一个数据页又占16kb；
 - 时间：每次对表中的数据增删改查时，都要去修改索引，因为破坏了阶段和记录的排序，会对性能造成损耗；

索引适用的条件：
 - 全值匹配
 - 最佳左前缀匹配；
 - 匹配列前缀，如name like 'Ant%'，
 - 范围匹配，由于记录是按索引排好序的，所以可以匹配范围的两个端点，但是若对多个列同时进行范围查找，则只有对索引最左边的那个列进行范围查找的时候才能用到B+树索引；
 - 精确匹配某一列并范围匹配另一列；
 - 用于排序；（ASC、DESC混用时不能使用）
 - 用于分组；

如何挑选索引：

 - 只为用于搜索、排序或分组的列创建索引（WHERE、ORDER BY、GROUP BY）
 - 考虑列的基数，即不要包含太多的重复值；
 - 索引列的类型尽量小；
 - 建立字符串值前缀的索引；建立格式：[name(10)]；不支持使用索引排序；
 - 让索引在比较表达式中单独出现；
 - 主键索引要有自增策略，不要手动插入，然后innoDB自动生成，这样就避免了页面分裂和记录移位；
 - 删除冗余、重复的索引；

## MYISAM的索引结构
MYISAM将索引和数据分开存储，其建立的索引相当于全都是二级索引，主键索引的叶子节点存储的不是完整的用户记录，而是行号，也就是说每次查询都需要一次回表操作；
![MYISAM数据存储结构](https://img-blog.csdnimg.cn/af48204752e6427eb6ee931d452c2683.png)

# 数据存储结构（待完善）
系统变量==datadir==可以看数据存在那个路径下；

```sql
SHOW VARIABLES LIKE 'datadir'
```

MYSQL系统数据库：
| 数据库名| 存储信息 |备注|
|--|--|--|
|  mysql| MYSQL的用户账户和权限信息<br>一些存储过程、时间的定义信息<br>一些运行过程中产生的日志信息<br>一些帮助信息及时区信息 ||
| information_schema | MYSQL服务器维护的所有其他数据库信息，是一些描述信息，元数据 ||
| performance_schema |  MYSQL服务器运行过程中的状态信息，算是一个性能监控||
| sys | 通过视图的方式将information_schema和performance_schema结合起来，可以更方便的了解服务器的一些性能信息 ||

每个数据库都对应数据目录下的一个子目录，即一个文件夹，每个表的信息可以分成两种：

 1. 表结构的定义；
 2. 表中的数据；

## innoDB存储表数据
innoDB使用表空间的概念管理数据；
一个表空间最多可以拥有2³²个页，这是因为每个页都对应一个页号，也就是FIL_PAGE_OFFSET字段，这个字段占4个字节；
![表空间结构](https://img-blog.csdnimg.cn/9a1db5a4a33e45d9967651551423089d.png)
如上图所示，连续的64个页是一个区，每256个区划分成一组，每个组的最开始几个页面类型是固定的；
若表中数据大量的时候，为某个索引分配空间的时候就不按照页为单位分配了，而是按**区**为单位分配。
### 区的概念
<center>区的分类</center>

|状态名|类型| 含义 |所属空间|
|--|--|--|--|
|FREE| 空闲的区 | 还没有用到这个区中的任何页面 |直属表空间|
|FREE_FRAG| 有剩余空间的碎片区 | 碎片区还有可以用的页面 |直属表空间|
|FULL_FRAG| 没有剩余空间的碎片区 | 碎片区没有空闲页面 |直属表空间|
|FSEG| 附属于某个段的区 |  |附属于某个段|

每个区对应一个XDES Entry结构
![XDES](https://img-blog.csdnimg.cn/8cfbb687bebb457fadd3538da13d67ae.png)
其中Page State Bitmap共128个比特位，划分成64个部分，每个部分2个比特位，第一位表示对应的页是否空闲，第二个未使用；


### 表空间概念
#### 系统表空间
对应文件系统上一个或多个实际的文件，默认大小12M，是自扩展文件，即不够用的时候会自己增加文件大小；
MYSQL5.5.7～MYSQL5.6.6间各个版本表数据会默认存储到这个**系统表空间**；

#### 独立表空间
MYSQL5.6.6后，MYSQL为每一个表建立一个独立表空间，一张表对应两个文件：
test.frm：表结构数据
test.ibd：表数据，innoDB中索引即数据，数据即索引
## MYISAM存储表数据
MYISAM中没有表空间的概念，表数据都存放在对应的数据库子目录下，一张表对应三个文件：
test.frm：表结构数据
test.MYD：数据文件，插入的用户记录
test.MYI：索引文件

## 段的概念
一个索引会生成两个段，存放叶子节点的区的集合就算是一个段，存放非叶子节点的区的集合也是一个段；
段以区为单位申请空间，一个区默认大小1MB；

为了防止空间的浪费，引入<font color=red>**碎片区**</font>的概念，碎片区不属于任何一个段，直属于表空间，那么为某个段分配空间的策略如下：

 - 在刚开始向表中插入数据的时候，段是从某个碎片区以单个页面为单位来分配存储空间的；
 - 当某个段已经占用了32个碎片区页面后，会以完整的区为单位来分配存储空间；

段是一些零散的页面以及一些完整的区的集合；

每一个段对应一个INODE Entry结构，这个INODE Entry结构描述了这个段的各种信息，通过Segment Header结构来定位一个INODE Entry：
![在这里插入图片描述](https://img-blog.csdnimg.cn/6d3b498ab7434f2ebb48f09e7cbbb7c2.png)
- Space ID of the INODE Entry ： INODE Entry 结构所在的表空间ID。
- Page Number of the INODE Entry ： INODE Entry 结构所在的页面页号。
- Byte Offset of the INODE Ent ： INODE Entry 结构在该页面中的偏移量

# 访问方法

```sql
CREATE TABLE single_table (
	id INT NOT NULL AUTO_INCREMENT,
	key1 VARCHAR(100),
	key2 INT,
	key3 VARCHAR(100),
	key_part1 VARCHAR(100),
	key_part2 VARCHAR(100),
	key_part3 VARCHAR(100),
	common_field VARCHAR(100),
	PRIMARY KEY (id),
	KEY idx_key1 (key1),
	UNIQUE KEY idx_key2 (key2),
	KEY idx_key3 (key3),
	KEY idx_key_part(key_part1, key_part2, key_part3)
) Engine=InnoDB CHARSET=utf8;
```

### const
通过主键或者唯一二级索引列来定位一条记录的访问方法；
```sql
SELECT * FROM single_table WHERE id = 1438
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/c3003a44da464c9b90bf773fdda09d35.png)

### ref
搜索条件是二级索引列与常数等值比较，采用二级索引来执行查询的访问方法；
```sql
SELECT * FROM single_table WHERE key1 = 'abc'
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/b912ac44b0514d3bba2f8d92e505e6de.png)
### ref_or_null
使用二级索引而不是全表扫描的方式执行这个查询时，就是ref_or_null
```sql
SELECT * FROM single_demo WHERE key1 = 'abc' OR key1 IS NULL
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/2f552d75baa440f99bf32d8123e1042d.png)
### range

利用索引进行范围匹配的访问方法
```sql
SELECT * FROM single_table WHERE key2 IN (1438, 6328) OR (key2 >= 38 AND key2 <= 79)
```

### index
不进行回表操作，遍历二级索引记录的执行方式
```sql
SELECT key_part1, key_part2, key_part3 FROM single_table WHERE key_part2 = 'abc'
```
### all
全表扫描执行查询的方式

## 索引合并
一把情况下，mysql执行一个查询最多只会用到单个二级索引，有些特殊情况也会使用到多个，这种使用多个索引来完成一次查询的执行方法叫做：index merge

### Intersection合并
使用两个二级索引分别查询主键id，查询到结果后对两个主键id的集合进行merge然后再回表查询；
原因：
虽然读取多个二级索引比读取一个二级索引消耗性能，但是读取二级索引的操作是顺序I/O ，而回表操作是随机I/O ，所以如果只读取一个二级索引时需要回表的记录数特别多，而读取多个二级索引之后取交集的记录数非常少，当节省的因为回表而造成的性能损耗比访问多个二级索引带来的性能损耗更高时，读取多个二级索引后取交集比只读取一个二级索引的成本更低

 - 情况一：二级索引列是等值匹配的情况，对于联合索引来说，在联合索引中的每个列都必须等值匹配，不能出现只出现匹配部分列的情况
```sql
// 可以使用Intersection合并
SELECT * FROM single_table WHERE key1 = 'a' AND key_part1 = 'a' AND key_part2 = 'b'
AND key_part3 = 'c'

// 不可以使用Intersection合并
SELECT * FROM single_table WHERE key1 > 'a' AND key_part1 = 'a' AND key_part2 = 'b'
AND key_part3 = 'c';

// 不可以使用Intersection合并
SELECT * FROM single_table WHERE key1 = 'a' AND key_part1 = 'a'
```
	
 - 情况二：主键列可以是范围匹配
```sql
SELECT * FROM single_table WHERE id > 100 AND key1 = 'a'
```
情况一 和 情况二 只是发生 Intersection 索引合并的必要条件，不是充分条件;
优化器只有在单独根据搜索条件从某个二级索引中获取的记录数太多，导致回表开销太大，而通过 Intersection 索引合并后需要回表的记录数大大减少时才会使用 Intersection 索引合并

==**尽量使用联合索引替代Intersection索引合并**==
### union合并
 - 情况一：二级索引列是等值匹配的情况，对于联合索引来说，在联合索引中的每个列都必须等值匹配，不能出现只出现匹配部分列的情况
```sql
// 可以使用union合并
SELECT * FROM single_table WHERE key1 = 'a' OR ( key_part1 = 'a' AND key_part2 = 'b'
AND key_part3 = 'c')

// 不可以使用union合并
SELECT * FROM single_table WHERE key1 > 'a' OR (key_part1 = 'a' AND key_part2 = 'b'
AND key_part3 = 'c')

// 不可以使用union合并
SELECT * FROM single_table WHERE key1 = 'a' OR key_part1 = 'a'
```
 - 情况二：主键列可以是范围匹配
 - 情况三：使用 Intersection 索引合并的搜索条件
（搜索条件的某些部分使用 Intersection 索引合并的方式得到的主键集合和其他方式得到的主键集合取交集）
```sql
SELECT * FROM single_table WHERE key_part1 = 'a' AND key_part2 = 'b' AND key_part3 ='c' OR (key1 = 'a' AND key3 = 'b')
```

### Sort-Union合并
```sql
SELECT * FROM single_table WHERE key1 < 'a' OR key3 > 'z'
```
先按照二级索引记录的主键值进行排序，之后按照 Union 索引合并方式执行的方式称之为 SortUnion 索引合并，很显然，这种 Sort-Union 索引合并比单纯的 Union 索引合并多了一步对二级索引记录的主键值排序的过程；

# 连接讨论
## 相关概念
**笛卡尔积**：连接查询的结果集中包含一个表中的每一条记录与另一个表中的每一条记录相互匹配的组合，像这样的结果集就叫做笛卡尔积；
**驱动表**：第一个需要被查询的表；
**被驱动表**：根据第一个要查询的表查询出来的数据，匹配记录的表；
==驱动表只会被访问一次，而被驱动表可能被访问多次==
```sql
SELECT * FROM t1, t2 WHERE t1.m1 > 1 AND t1.m1 = t2.m2 AND t2.n2 < 'd'
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/ac20007193db442ab54f020247ffdef0.png)

**内连接**：
语法： 
```sql
SELECT * FROM t1 INNER JOIN t2 ON xxx WHERE xxx 
```
因为在内连接中，WHERE和ON的子句是等价的，所以对于内连接不强制要求是否有ON；
内连接中驱动表和被驱动表是可以互换的，不会影响查询结果；
内连接中驱动表中的记录在被驱动表中找不到对应的记录，则不会展示在结果中；
**外连接**：
语法：
```sql
// 左连接
SELECT * FROM t1 LEFT JOIN t2 ON xxx WHERE xxx
// 右连接
SELECT * FROM t1 RIGHT JOIN t2 ON xxx WHERE xxx
```
其中左连接的驱动表是左边的表，右连接的驱动表是右边的表；

外连接必须要有ON语句指出连接的条件；
外连接的驱动表和被驱动表的顺序不能调换，会对查询结果产生影响；
外连接中驱动表的记录在被驱动表中找不到对应的记录，也会展示在结果中展示；
## 连接的原理

### 嵌套循环连接
![在这里插入图片描述](https://img-blog.csdnimg.cn/696021aa83e0460fb74048c7f4337e71.png)
 - 步骤1：选取驱动表，使用与驱动表相关的过滤条件，选取代价最低的单表访问方法来执行对驱动表的单表查询；
 - 步骤2：对上一步骤中查询驱动表得到的结果集中每一条记录，都分别到被驱动表中查找匹配的记录；

### 基于块的嵌套匹配
由于每次匹配被驱动表中的记录都可能会产生磁盘IO，因此引入join buffer的概念：
![在这里插入图片描述](https://img-blog.csdnimg.cn/ad63a9c5a1df4b4586c16615ee5558d1.png)
join buffer 就是执行连接查询前申请的一块固定大小的内存，先把若干条驱动表结果集中的记录装在这个 join buffer 中，然后开始扫描被驱动表，每一条被驱动表的记录一次性和 join buffer 中的多条驱动表记录做匹配；


==同时对于被驱动表的匹配，其实也相当于单表扫描，因此也可以通过在被驱动表中添加索引，来提升连接速度，注意，在真实工作中最好不要使用 * 作为查询列表，最好把真实用到的列作为查询列表==


# mysql的成本讨论
正如我们所知道的，mysql中一条语句的执行，包含两部分成本：

 1. IO成本；
 2. CPU成本；
读取一个页面的成本默认是1.0，读取并检测一条记录的成本是0.2；

mysql的成本对比过程：
 1. 根据搜索条件，找出所有符合条件的索引；
 2. 计算全表扫描的代价；
 3. 计算使用不同索引的代价；
 4. 对比代价，找出成本最小的那一个；

按照上述的介绍，我们来讨论一下单表查询和连接查询的成本；
表信息如下：
```sql
CREATE TABLE single_table (
id INT NOT NULL AUTO_INCREMENT,
key1 VARCHAR(100),
key2 INT,
key3 VARCHAR(100),
key_part1 VARCHAR(100),
key_part2 VARCHAR(100),
key_part3 VARCHAR(100),
common_field VARCHAR(100),
PRIMARY KEY (id),
KEY idx_key1 (key1),
UNIQUE KEY idx_key2 (key2),
KEY idx_key3 (key3),
KEY idx_key_part(key_part1, key_part2, key_part3)
) Engine=InnoDB CHARSET=utf8;
```
## 单表查询
```sql
SELECT * FROM single_table WHERE
key1 IN ('a', 'b', 'c') AND
key2 > 10 AND key2 < 1000 AND
key3 > key2 AND
key_part1 LIKE '%hello%' AND
common_field = '123';
```
首先可以使用以下语句查看表状态：
```sql
SHOW TABLE STATUS;
*************************** 1. row ***************************
Name: single_table
Engine: InnoDB
Version: 10
Row_format: Dynamic
Rows: 9693
Avg_row_length: 163
Data_length: 1589248
Max_data_length: 0
Index_length: 2752512
Data_free: 4194304
Auto_increment: 10001
Create_time: 2018-12-10 13:37:23
Update_time: 2018-12-10 13:38:03
Check_time: NULL
Collation: utf8_general_ci
Checksum: NULL
Create_options:
Comment:
1 row in set (0.01 sec)
```
Data_length：
MYISAM：该值就是数据文件的大小；
INNODB：聚簇索引所占存储空间大小；即：Data_length = 聚簇索引的页面数量 x 每个页面的大小

所以该表中页面数量=1589248/16/1024=97；
### 全表扫描成本
#### IO成本
公式：页面数量✖️成本系数+微调系数
97✖️1.0+1.1=98.1
#### CPU成本
公式：行数✖️成本系数+微调系数
9693✖️0.2+1.0=1939.6
#### 总体成本
公式：IO成本+CPU成本
98.1+1939.6=2037.7

### 使用索引成本
可能使用到的索引有idx_key1和idx_key2；
MySQL 查询优化器先分析使用唯一二级索引的成本，再分析使用普通索引的成本；
#### idx_key1成本
##### IO成本
**统计区间范围数量**
范围数量：3
**统计需要回表的记录数**
==先找最左记录和最右记录，若相隔不远就直接统计区间内的记录数，否则只沿着区间最左记录 向右读10个页面，计算平均每个页面中包含多少记录，然后用这个平均值乘以区间最左记录和区间最右记录之间的页面数量就可以了；
对于**页面数量**可以统计最左记录和最右记录的父节点间的记录数量，就是页面数量==

经过统计区间的记录数量：118

二级索引回表：118✖️1=118
##### CPU成本
读取二级索引：118✖️0.2+0.01=23.61
对比记录是否满足搜索条件：118✖️0.2=23.6
#####  总体成本
IO：3.0 + 118 x 1.0 = 121.0 (范围区间的数量 + 预估的二级索引记录条数)
CPU：118 x 0.2 + 0.01 + 118 x 0.2 = 47.21 （读取二级索引记录的成本 + 读取并检测回表后聚簇索引记录的成本）
总体：121.0 + 47.21 = 168.21
#### idx_key2成本
##### IO成本
**统计区间范围数量**
范围数量：1
**统计需要回表的记录数**
经过统计区间的记录数量：95

二级索引回表：95✖️1=95
##### CPU成本
读取二级索引：95✖️0.2+0.01=19.01
对比记录是否满足搜索条件：95✖️0.2=19.0
#####  总体成本
IO：1.0 + 95 x 1.0 = 96.0 (范围区间的数量 + 预估的二级索引记录条数)
CPU：95 x 0.2 + 0.01 + 95 x 0.2 = 38.01 （读取二级索引记录的成本 + 读取并检测回表后聚簇索引记录的成本）
总体：96.0 + 38.01 = 134.01

综上，选取idex_key2作为使用的索引；

```sql
SHOW INDEX FROM table_name
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/426c798367dd4fdfa472303956b5d30f.png)
其中Cardinality是估计值：<font color=red>不精确</font>
一个值的重复次数 ≈ Rows ÷ Cardinality
## 连接查询
对驱动表进行查询后得到的记录条数称之为驱动表的<font color=red>扇出</font>
查询成本：

 - 单次查询驱动表的成本；
 - 多次查询被驱动表的成本；


连接查询总成本 = 单次访问驱动表的成本 + 驱动表扇出数 x 单次访问被驱动表的成本

所以优化重点：
- 尽量减少驱动表的扇出
- 对被驱动表的访问成本尽量低

即：尽量在被驱动表的连接列上建立索引，被驱动表的连接列最好是该表的主键或者唯一二级索引列；


一条sql语句的执行分成两层：
- server层
- 存储引擎层
一条语句在 server 层中执行的成本是和它操作的表使用的存储引擎是没关系的，所以关于这些操作对应的成本常数 就存储在了**server_cost**表中，而依赖于存储引擎的一些操作对应的成本常数就存储在了**engine_cost**表中；

更新表中的参数：
```sql
//更新值
UPDATE mysql.server_cost
SET cost_value = 0.4
WHERE cost_name = 'row_evaluate_cost';

//重新加载变量
FLUSH OPTIMIZER_COSTS;
```
# innoDB的统计数据收集
innoDB以表为单位统计数据，这些统计数据既可以是基于磁盘的永久性统计数据，也可以说基于内存的非永久性统计数据；
innodb_stats_persistent=1：磁盘
innodb_stats_persistent=0：内存
```sql
CREATE TABLE 表名 (...) Engine=InnoDB, STATS_PERSISTENT = (1|0);

ALTER TABLE 表名 Engine=InnoDB, STATS_PERSISTENT = (1|0);
```
innodb_stats_persistent_sample_pages 控制着永久性统计数据的采样页面数量；（默认值：20）
innodb_stats_transient_sample_pages 控制着非永久性统计数据的采样页面数量；(默认值：8)
数量越大，统计值越准确，同时性能损耗越严重；

对于永久存储统计数据，体现在以下两个表：
- innodb_table_stats表存储了关于表的统计数据，每一条记录对应着一个表的统计数据；
- innodb_index_stats表存储了关于索引的统计数据，每一条记录对应着一个索引的一个统计项的统计数据；
<center>innodb_table_stats</center>

|字段名|描述  | 备注|
|--|--|--|
|  database_name| 数据库名 ||
| table_name | 表名 ||
| last_update | 本条记录最后更新时间 ||
| n_rows | 表中记录的条数 ||
| clustered_index_size | 表的聚簇索引占用的页面数量 ||
| sum_of_other_index_sizes | 表的其他索引占用的页面数量 ||

<center>innodb_index_stats</center>

|字段名|描述  | 备注|
|--|--|--|
|  database_name| 数据库名 ||
| table_name | 表名 ||
| index_name | 索引名 ||
| last_update | 本条记录最后更新时间 ||
| stat_name | 统计项的名称 ||
| stat_value | 对应的统计项的值 ||
| sample_size | 为生成统计数据而采样的页面数量||
| stat_description |对应的统计项的描述|n_leaf_pages:表示该索引的叶子节点占用多少页面<br>n_diff_pfxNN:表示对应的索引列不重复的值有多少<br>(n_diff_pfx01 表示的是统计 key_part1 这单单一个列不重复的值有多少<br>n_diff_pfx02 表示的是统计 key_part1、key_part2 这两个列组合起来不重复的值有多少)|

对于统计数据的更新：
系统变量innodb_stats_auto_recalc决定是否自动重新计算统计数据；
若记录变更数量超过10%，且开启了自动更新，则会异步更新统计数据；
也可以手动执行以下语句，更新统计数据：
```sql
ANALYZE TABLE table_name
```
我们可以针对某个具体的表，在创建和修改表时通过指定 ：
- STATS_PERSISTENT（使用内存还是磁盘保存统计数据）
- STATS_AUTO_RECALC（自动更新开关）
- STATS_SAMPLE_PAGES（采样页面数量）
的值来控制相关统计数据属性。

innodb_stats_method 决定着在统计某个索引列不重复值的数量时如何对待 NULL 值
|参数 |含义 |备注|
|--|--|--|
| null_equal|所有NULL值都相等||
| null_unequal| 所有NULL值不相等||
| null_ignored| 直接忽略NULL值||

# MYSQL基于规则的优化

## 条件化简
| 策略|优化前| 优化后|备注|
|--|--|--|--|
|移除不必要的联系|((a = 5 AND b = c) OR ((a > c) AND (c < 5)))|(a = 5 and b = c) OR (a > c AND c < 5)| |
|常量传递| a = 5 AND b > a| a = 5 AND b > 5 | |
|等值传递|(a < 1 and b = b) OR (a = 6 OR 5 != 5)|a < 1 OR a = 6||
|表达式计算|a = 5 + 1|a = 6|如果某个列并不是以单独的形式作为表达式的操作数时，比如出现在函数中，出现在某个更复杂表达式中，如：ABS(a) > 5、-a < -8，优化器不会对其进行化简|
|HAVING语句和WHERE语句合并|||如果查询语句中没有出现诸如 SUM 、 MAX 等等的聚集函数以及 GROUP BY 子句，优化器就把 HAVING 子句和WHERE 子句合并起来|
|常量表检测|SELECT * FROM table1 INNER JOIN table2 ON table1.column1 = table2.column2 WHERE table1.primary_key = 1|SELECT table1表记录的各个字段的常量值, table2.* FROM table1 INNER JOIN table2ON table1表column1列的常量值 = table2.column2|优化器在分析一个查询语句时，先首先执行常量表查询，然后把查询中涉及到该表的条件全部替换成常数，最后再分析其余表的查询成本|

## 子查询介绍

<center>子查询位置分类</center>

|位置|语句|备注|
|--|--|--|
| SELECT子句| SELECT (SELECT m1 FROM t1 LIMIT 1)| |
| FROM子句| SELECT m, n FROM (SELECT m2 + 1 AS m, n2 AS n FROM t2 WHERE m2 > 2) AS t| |
| WHERE或ON子句| SELECT * FROM t1 WHERE m1 IN (SELECT m2 FROM t2)| |
| ORDER BY| | 意义不大～|
| GROUP BY| | 意义不大～|

<center>子查询返回结果集分类</center>

|类别|语句|备注|
|--|--|--|
|标量子查询|SELECT (SELECT m1 FROM t1 LIMIT 1)|返回单一值的子查询|
|列子查询|SELECT * FROM t1 WHERE m1 IN (SELECT m2 FROM t2)|返回一列的子查询|
|行子查询|SELECT * FROM t1 WHERE (m1, n1) = (SELECT m2, n2 FROM t2 LIMIT 1)|返回一条记录的子查询|
|表子查询|SELECT * FROM t1 WHERE (m1, n1) IN (SELECT m2, n2 FROM t2)|返回结果既包含很多条记录，也包含很多列|

<center>子查询按与外层查询关系集分类</center>

|类别|语句|备注|
|--|--|--|
|不相关子查询|SELECT (SELECT m1 FROM t1 LIMIT 1)|子查询可以单独运行出结果，而不依赖于外层查询的值|
|相关子查询|SELECT * FROM t1 WHERE m1 IN (SELECT m2 FROM t2 WHERE n1 = n2)|子查询的执行需要依赖于外层查询的值|

<center>子查询在布尔表达式中的使用</center>

|描述|语法|语句|备注|
|--|--|--|--|
|使用 = 、 > 、 < 、 >= 、 <= 、 <> 、 != 、 <=> 作为布尔表达式的操作符|操作数 comparison_operator (子查询)|SELECT * FROM t1 WHERE m1 < (SELECT MIN(m2) FROM t2)|这里的子查询只能是标量子查询或者行子查询，也就是子查询的结果只能返回一个单一的值或者只能是一条记录|
|IN 或者 NOT IN|操作数 [NOT] IN (子查询)|SELECT * FROM t1 WHERE (m1, n2) IN (SELECT m2, n2 FROM t2)|判断某个操作数在不在由子查询结果集组成的集合中|
|ANY/SOME|操作数 comparison_operator ANY/SOME(子查询)|SELECT * FROM t1 WHERE m1 > ANY(SELECT m2 FROM t2)|只要子查询结果集中存在某个值和给定的操作数做 comparison_operator 比较结果为 TRUE ，那么整个表达式的结果就为 TRUE ，否则整个表达式的结果就为 FALSE|
|ALL|操作数 comparison_operator ALL(子查询)|SELECT * FROM t1 WHERE m1 > ALL(SELECT m2 FROM t2)|子查询结果集中所有的值和给定的操作数做 comparison_operator 比较结果为 TRUE ，那么整个表达式的结果就为 TRUE ，否则整个表达式的结果就为 FALSE|
|EXISTS子查询|[NOT] EXISTS (子查询)|SELECT * FROM t1 WHERE EXISTS (SELECT 1 FROM t2)|判断子查询的结果集中是否有记录，而不在乎它的记录具体是个啥|

### 子查询注意事项：
- 子查询必须用小括号扩起来
- 在 SELECT 子句中的子查询必须是标量子查询
- 在想要得到标量子查询或者行子查询，但又不能保证子查询的结果集只有一条记录时，应该使用 LIMIT 1 语句来限制记录数量。
- 对于 [NOT] IN/ANY/SOME/ALL 子查询来说，子查询中不允许有 LIMIT 语句
- 子查询的结果其实就相当于一个集合，集合里的值排不排序一点儿都不重要
- 集合里的值去不去重也没啥意义
- 在没有聚集函数以及 HAVING 子句时， GROUP BY 子句就是个摆设
- 不允许在一条语句中增删改某个表的记录时同时还对该表进行子查询


## 子查询的执行步骤
表定义：
```sql
CREATE TABLE single_table (
id INT NOT NULL AUTO_INCREMENT,
key1 VARCHAR(100),
key2 INT,
key3 VARCHAR(100),
key_part1 VARCHAR(100),
key_part2 VARCHAR(100),
key_part3 VARCHAR(100),
common_field VARCHAR(100),
PRIMARY KEY (id),
KEY idx_key1 (key1),
UNIQUE KEY idx_key2 (key2),
KEY idx_key3 (key3),
KEY idx_key_part(key_part1, key_part2, key_part3)
) Engine=InnoDB CHARSET=utf8;
```
### 标量子查询、行子查询执行方式
#### 不相关子查询
```sql
SELECT * FROM s1 WHERE key1 = (SELECT common_field FROM s2 WHERE key3 = 'a' LIMIT 1)
```
**执行步骤：**
先执行子查询中的语句，再将上一步子查询得到的结果当作外层查询的参数再执行外层查询；
对于包含不相关的标量子查询或者行子查询的查询语句来说，MySQL会分别独立的执行外层查询和子
查询，就当作两个单表查询就好了。
#### 相关子查询
```sql
SELECT * FROM s1 WHERE key1 = (SELECT common_field FROM s2 WHERE s1.key3 = s2.key3 LIMIT 1);
```
**执行步骤：**
- 先从外层查询中获取一条记录，本例中也就是先从 s1 表中获取一条记录。
- 然后从上一步骤中获取的那条记录中找出子查询中涉及到的值，本例中就是从 s1 表中获取的那条记录中找出 s1.key3 列的值，然后执行子查询。
- 最后根据子查询的查询结果来检测外层查询 WHERE 子句的条件是否成立，如果成立，就把外层查询的那条记录加入到结果集，否则就丢弃。
- 再次执行第一步，获取第二条外层查询中的记录，依次类推～


### IN子查询优化

#### 不相关子查询
##### 物化表
**1.物化表概念**
```sql
SELECT * FROM s1 WHERE key1 IN (SELECT common_field FROM s2 WHERE key3 = 'a')
```

先执行子查询，但是若子查询中的数据很大，内存中装不下怎么办呢，in中参数太多会不会影响性能呢，这里引入`物化`概念，不直接将不相关子查询的结果集当作外层查询的参数，而是将该结果集写入一个临时表里。
这个将子查询结果集中的记录保存到临时表的过程称之为`物化`。
写入临时表的过程：
- 该临时表的列就是子查询结果集中的列
- 写入临时表的记录会被去重
- 一般情况下子查询结果集不会大的离谱，所以会为它建立基于内存的使用 Memory 存储引擎的临时表，而且
会为该表建立哈希索引
- 如果子查询的结果集非常大，超过了系统变量 tmp_table_size 或者 max_heap_table_size ，临时表会转而
使用基于磁盘的存储引擎来保存结果集中的记录，索引类型也对应转变为 B+ 树索引

**2.物化表转连接**
将原始sql语句的子查询用物化表进行存储，则sql语句就可以转化成以下形式的内连接：
```sql
SELECT s1.* FROM s1 INNER JOIN materialized_table ON key1 = m_val
```
这样，查询优化器就可以计算使用两张表分别作为驱动表的成本，从而选取成本较小的查询方案；
所以，整个查询过程的成本组成如下：（假设使用s1表作为驱动表）
- 物化子查询时需要的成本
- 扫描 s1 表时的成本
- s1表中的记录数量 × 通过 m_val = xxx 对 materialized_table 表进行单表访问的成本（前边说过物化表中的记录是不重复的，并且为物化表中的列建立了索引，所以这个步骤显然是非常快的）
##### 将子查询转化成semi-join
还是这个sql语句
```sql
SELECT * FROM s1 WHERE key1 IN (SELECT common_field FROM s2 WHERE key3 = 'a')
```
对于 s1 表的某条记录来说，我们只关心在 s2 表中是否存在与之匹配的记录是否存在，而不关心具体有多少条记录与之匹配，最终的结果集中只保留 s1 表的记录，这样就引出了`半连接(semi-join)`的概念。

<center>实现半连接的方式</center>

|名字|条件|方式|原sql|转化sql|备注|
|--|--|--|--|--|--|
|Table pullout<br>（子查询中的表上拉）|子查询的查询列表处只有主键或者唯一索引列|直接把子查询中的表上拉到外层查询的 FROM 子句中，并把子查询中的搜索条件合并到外层查询的搜索条件中|SELECT * FROM s1 WHERE key2 IN (SELECT key2 FROM s2 WHERE key3 = 'a')|SELECT s1.* FROM s1 INNER JOIN s2 ON s1.key2 = s2.key2 WHERE s2.key3 = 'a'|主键或者唯一索引列中的数据本身就是不重复的嘛！所以对于同一条 s1 表中的记录，不可能找到两条以上的符合 s1.key2 = s2.key2 的记录|
|DuplicateWeedout execution strategy （重复值消除）| | 建立一个临时表，每当s1的记录要加入结果集时，先把这条记录的id加入临时表，若能成功添加，则说明无重复，直接添加到结果集中，否则丢弃|SELECT * FROM s1 WHERE key1 IN (SELECT common_field FROM s2 WHERE key3 = 'a')|临时表：CREATE TABLE tmp (id PRIMARY KEY)||
|LooseScan execution strategy （松散索引扫描）||只取值相同的记录的第一条去做匹配操作|SELECT * FROM s1 WHERE key3 IN (SELECT key1 FROM s2 WHERE key1 > 'a' AND key1 < 'b')
|Semi-join Materialization execution strategy
|FirstMatch execution strategy （首次匹配）| | 先取一条外层查询的中的记录，然后到子查询的表中寻找符合匹配条件的记录，如果能找到一条，则将该外层查询的记录放入最终的结果集并且停止查找更多匹配的记录，如果找不到则把该外层查询的记录丢弃掉；然后再开始取下一条外层查询中的记录，重复上边这个过程| SELECT * FROM s1 WHERE key1 IN (SELECT common_field FROM s2 WHERE s1.key3 = s2.key3)| SELECT s1.* FROM s1 SEMI JOIN s2 ON s1.key1 = s2.common_field AND s1.key3 = s2.key3|接下来可以使用上述的几种方式来进行子查询`相关子查询并不是一个独立的查询，所以不能转换为物化表来执行查询`|

**semi-join适用条件：**
- 该子查询必须是和 IN 语句组成的布尔表达式，并且在外层查询的 WHERE 或者 ON 子句中出现。
- 外层查询也可以有其他的搜索条件，只不过和 IN 子查询的搜索条件必须使用 AND 连接起来。
- 该子查询必须是一个单一的查询，不能是由若干查询由 UNION 连接起来的形式。
- 该子查询不能包含 GROUP BY 或者 HAVING 语句或者聚集函数。

形式如下：
```sql
SELECT ... FROM outer_tables 
WHERE expr IN (SELECT ... FROM inner_tables ...) AND ...

SELECT ... FROM outer_tables
WHERE (oe1, oe2, ...) IN (SELECT ie1, ie2, ... FROM inner_tables ...) AND ...
```

**semi-join不适用条件：**
|条件|sql语句|备注|
|--|--|--|
|外层查询的WHERE条件中有其他搜索条件与IN子查询组成的布尔表达式使用`OR`连接起来|SELECT * FROM s1 WHERE key1 IN (SELECT common_field FROM s2 WHERE key3 = 'a') OR key2 > 100;|
|使用`NOT IN`而不是 IN 的情况|SELECT * FROM s1 WHERE key1 NOT IN (SELECT common_field FROM s2 WHERE key3 = 'a')|
|在`SELECT子句`中的IN子查询的情况|SELECT key1 IN (SELECT common_field FROM s2 WHERE key3 = 'a') FROM s1|
|子查询中包含`GROUP BY、HAVING或者聚集函数`的情况|ELECT * FROM s1 WHERE key2 IN (SELECT COUNT(*) FROM s2 GROUP BY key1)|
|子查询中包含`UNION`的情况|SELECT * FROM s1 WHERE key1 IN (SELECT common_field FROM s2 WHERE key3 = 'a' UNION SELECT common_field FROM s2 WHERE key3 = 'b')|

对于不能使用semi-join查询的`IN不相关子查询`，mysql会先尝试物化，若不能转化成物化表或物化的成本过高，则	会转换成EXISTS查询；
不管相关子查询还是不相关子查询，只要是IN子查询放在WHERE或ON语句中，那都可以转换成EXISTS子查询；
```sql
outer_expr IN (SELECT inner_expr FROM ... WHERE subquery_where)

EXISTS (SELECT inner_expr FROM ... WHERE subquery_where AND outer_expr=inner_exp

// 转化举例
SELECT * FROM s1
WHERE key1 IN (SELECT key3 FROM s2 where s1.common_field = s2.common_field)
OR key2 > 1000;
👇
SELECT * FROM s1
WHERE EXISTS (SELECT 1 FROM s2 where s1.common_field = s2.common_field AND s2.ke
y3 = s1.key1)
OR key2 > 1000
// 转化后可以使用索引
```
**1.IN子查询优化**
如果 IN 子查询符合转换为 semi-join 的条件，查询优化器会优先把该子查询为 semi-join ，然后再考虑下边5种执行半连接的策略中哪个成本最低：
- Table pullout
- DuplicateWeedout
- LooseScan
- Materialization
- FirstMatch
选择成本最低的那种执行策略来执行子查询。

如果 IN 子查询不符合转换为 semi-join 的条件，那么查询优化器会从下边两种策略中找出一种成本更低的方式执行子查询：
- 先将子查询物化之后再执行查询
- 执行 IN to EXISTS 转换

**2.ANY/ALL子查询优化**
![在这里插入图片描述](https://img-blog.csdnimg.cn/613cc51487444a73b39e581b401d07ed.png)
**3.【NOT】 EXISTS子查询优化**
- 如果 [NOT] EXISTS 子查询是不相关子查询，可以先执行子查询，得出该 [NOT] EXISTS 子查询的结果是 TRUE 还是 FALSE ，并重写原先的查询语句；
- 对于相关的 [NOT] EXISTS 子查询来说，只能按照上述的那种执行相关子查询的方式来执行。不过如果 [NOT] EXISTS 子查询中如果可以使用索引的话，那查询速度也会加快不少；

**4.派生表优化**
子查询放在外层查询的 FROM 子句后，那么这个子查询的结果相当于一个`派生表`
```sql
SELECT * FROM (
SELECT id AS d_id, key3 AS d_key3 FROM s2 WHERE key1 = 'a'
) AS derived_s1 WHERE d_key3 = 'a';
```
- 派生表物化；（延迟物化：在查询中真正用到派生表时才回去尝试物化派生表，而不是还没开始执行查询呢就把派生表物化掉）
- 将派生表和外层的表合并，也就是将查询重写为没有派生表的形式
派生表中有以下语句的时候不能合并：
	- 聚集函数，比如MAX()、MIN()、SUM()啥的
	- DISTINCT
	- GROUP BY
	- HAVING
	- LIMIT
	- UNION 或者 UNION ALL
	- 派生表对应的子查询的 SELECT 子句中含有另一个子查询

MySQL 在执行带有派生表的时候，优先尝试把派生表和外层查询合并掉，如果不行的话，再把派生表物化掉
执行查询。

# EXPLAIN讨论
```sql
mysql> explain select * from student where id < 10;
+----+-------------+---------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
| id | select_type | table   | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+---------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | student | NULL       | range | PRIMARY       | PRIMARY | 4       | NULL |    9 |   100.00 | Using where |
+----+-------------+---------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.24 sec)
```
<center>EXPLAIN字段</center>

![在这里插入图片描述](https://img-blog.csdnimg.cn/5365b7c09bad4cdba22e92e4ae69b5c5.png)

在EXPLAIN和查询语句中间加上`FORMAT=JSON`，可以显示查询成本。
```sql
mysql> EXPLAIN FORMAT=JSON SELECT * FROM s1 INNER JOIN s2 ON s1.key1 = s2.key2 WHERE s1.common_field = 'a'
```
这里面大部分的输出在EXPLAIN中已有体现，只是详细的介绍一遍，我们需要关注
```json
"cost_info": {
	"read_cost": "1840.84",
	"eval_cost": "193.76",
	"prefix_cost": "2034.60",
	"data_read_per_join": "1M"
}
```
- read_cost 是由下边这两部分组成的：
	- IO 成本
	- 检测 rows × (1 - filter) 条记录的 CPU 成本
- eval_cost 是这样计算的：
	- 检测 rows × filter 条记录的成本。
- prefix_cost 就是单独查询 s1 表的成本，也就是：
	- read_cost + eval_cost
- data_read_per_join 表示在此次查询中需要读取的数据量

我们要重点关注`prefix_cost`，这个是查询成本。

执行完EXPLAIN后还可以接着输入`SHOW WARNINGS`，当返回`code=1003`时，Message可以看出查询优化器将我们的查询语句重写后的语句；

## 使用optimizer trace 功能查看优化步骤
```sql
# 1. 打开optimizer trace功能 (默认情况下它是关闭的):
SET optimizer_trace="enabled=on";
# 2. 这里输入你自己的查询语句
SELECT ...;
# 3. 从OPTIMIZER_TRACE表中查看上一个查询的优化过程
SELECT * FROM information_schema.OPTIMIZER_TRACE;
# 4. 可能你还要观察其他语句执行的优化过程，重复上边的第2、3步
...
# 5. 当你停止查看语句的优化过程时，把optimizer trace功能关闭
SET optimizer_trace="enabled=off";
```
执行后的输出分成以下四个部分：
- QUERY ：表示我们的查询语句。
- TRACE ：表示优化过程的JSON格式文本。
- MISSING_BYTES_BEYOND_MAX_MEM_SIZE ：由于优化过程可能会输出很多，如果超过某个限制时，多余的文本
将不会被显示，这个字段展示了被忽略的文本字节数。
- INSUFFICIENT_PRIVILEGES ：表示是否没有权限查看优化过程，默认值是0，只有某些特殊情况下才会是1 ，暂时不关心这个字段的值。

其中我们关心trace部分，气包含以下三个阶段：
- prepare阶段
- optimize阶段
- execute阶段
我们主要看optimize阶段。


# InnoDB的Buffer
即使只需要访问一个页中的一条记录，也需要将整个页加载到内存中，而读取硬盘的IO操作是十分耗时的，这就需要在内存中建立缓存，下面来讨论一下InnoDB的缓存策略；

Buffer Pool：缓冲池
默认值：128MB
通过innodb_buffer_pool_size参数可以修改；

Buffer Pool中对页的管理和磁盘对页的管理一样，大小都是16KB，InnoDB中对缓存页的管理是通过`控制块`实现的，控制块和缓存页一一对应。存储结构如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/5b7aa04038464c6a8535a28c802ff777.png)
每个控制块大概占缓存页的5%。

## 缓存页的查找
使用hash表存储：
key：表空间号+页号
value：缓存页

## 对空闲缓存页的管理
内部维护了一个free链表：
![在这里插入图片描述](https://img-blog.csdnimg.cn/089e419ba95a4e6da4593ff2775e2913.png)
每当需要从磁盘中加载一个页到Buffer Pool中时，就从free链表中取一个空闲的缓存页，并且把该缓存页对应的控制块的信息填上（就是该页所在的表空间、页号之类的信息），然后把该缓存页对应的 free链表节点从链表中移除，表示该缓存页已经被使用了。
## 脏缓存页的处理
InnoDB将修改后的“脏数据页”不马上写入磁盘，内部维护flush链表进行记录，结构如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/9b6e5bc105dc4d808df1d6ab87456b70.png)

把在mtr执行过程中可能修改过的页面加入到Buffer Pool的flush链表：
- oldest_modification ：如果某个页面被加载到Buffer Pool后进行第一次修改，那么就将修改该页面的mtr开始时对应的lsn值写入这个属性。
- newest_modification ：每修改一次页面，都会将修改该页面的mtr结束时对应的lsn值写入这个属性。也就是说该属性表示页面最近一次修改后对应的系统lsn值。

flush链表中的脏页按照修改发生的时间顺序进行排序，也就是按照oldest_modification代表的LSN值进行排序，被多次更新的页面不会重复插入到flush链表中，但是会更新newest_modification属性的值。

## 真实环境中的Buffer Pool
### 存储结构
每次当我们要重新调整Buffer Pool 大小时，都需要重新向操作系统申请一块连续的内存空间，然后将旧的Buffer Pool中的内容复制到这一块新空间，这是极其耗时的；
因此引入`chunk`的概念，以`chunk`为单位申请内存，一个Buffer Pool实例其实是由若干个 chunk 组成的，一个 chunk 就代表一片连续的内存空间，里边儿包含了若干缓存页与其对应的控制块，画个图表示就是这样：
![在这里插入图片描述](https://img-blog.csdnimg.cn/6defd7ff362b4222b0794195719d79e5.png)Buffer Pool中的各种链表都需要加锁处理啥的，在Buffer Pool特别大而且多线程并发访问特别高的情况下，单一的Buffer Pool可能会影响请求的处理速度，所以在 Buffer Pool 特别大的时候，我们可以把它们拆分成若干个小的 Buffer Pool ，每个 Buffer Pool 都称为一个 实例 ，它们都是独立的，独立的去申请内存空间，独立的管理各种链表，结构图可以参考上图。
### 刷新策略
#### 冷热数据刷新
`提高缓存命中率`
InnoDB使用LRU链表进行缓存页的刷新，结构如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/411e7979f1ac4c3f92061b7b852ef00e.png)
InnoDB中的LRU链表分成old区和young区，系统变量innodb_old_blocks_pct的值来确定 old 区域在LRU链表中所占的比例；
- 当磁盘上的某个页面在初次加载到Buffer Pool中的某个缓存页时，该缓存页对应的控制块会被放到old区域的头部，在对某个处在 old 区域的缓存页进行第一次访问时就在它对应的控制块中记录下来这个访问时间，如果后续的访问时间与第一次访问的时间在某个时间间隔内，那么该页面就不会被从old区域移动到young区域的头部，否则将它移动到young区域的头部（时间间隔innodb_old_blocks_time控制）
==以上策略主要是为了防止预读页面后面kennel不进行后续访问==
- 只有被访问的缓存页位于 young 区域的 1/4 的后边，才会被移动到LRU链表头部；
==以上策略为了防止频繁的节点操作==
#### 脏数据刷新
- 从 LRU链表 的冷数据中刷新一部分页面到磁盘
后台线程会定时从 LRU链表 尾部开始扫描一些页面，扫描的页面数量可以通过系统变量innodb_lru_scan_depth 来指定，如果从里边儿发现脏页，会把它们刷新到磁盘
- 从 flush链表 中刷新一部分页面到磁盘
后台线程也会定时从 flush链表 中刷新一部分页面到磁盘，刷新的速率取决于当时系统是不是很繁忙

# 事务
ACID
A：原子性
C：一致性
I：隔离性
D：持久性

# redo日志
为了满足`持久性`的要求，需要在事务提交之前将操作记录存储到磁盘，方便下次重启的时候恢复数据，但是直接将修改的页面刷新到磁盘会面临两个问题，1、刷新一个完整的数据页很浪费；2、随机IO刷新起来比较慢；所以这里引入redo日志，记录用户的操作。
结构：
![在这里插入图片描述](https://img-blog.csdnimg.cn/681b7fc8ee6142c0b57a5262814ef53f.png)
- type：该条redo日志类型；
- sapce ID：表空间ID；
- page number：页号；
- data：该条redo日志的具体内容；

redo日志会把事务在执行过程中对数据库所做的所有修改都记录下来，在之后系统崩溃重启后可以把事务所做的任何修改都恢复出来；

## Mini-Transaction（MTR）
向某个索引对应的B+树中插入一条记录的这个过程必须是原子的，所以在执行新记录的插入的时候想要保证原子操作，就必须要以`组`的形式记录redo日志；
那么如何来实现`组`的概念呢？在组中的最后一条redo日志后边加上一条特殊类型的redo日志，该类型名称为 MLOG_MULTI_REC_END，type 字段对应的十进制数字为 31 ，该类型的redo日志结构很简单，只有一个type字段，那么一组的日志结构就如下所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/5d95a4d78b9243e6b02d7b4aa250602b.png)
但是会有一些特殊情况，比如一条redo日志就是一个原子的操作，不足以形成`组`，这时怎么办呢？
由于redo日志的类型只有几十种，是小于127的，而type字段占了8字节，这样第一个比特位就可以空出来表示这个概念，所以：
==如果type字段的第一个比特位为 1 ，代表该需要保证原子性的操作只产生了单一的一条redo日志，否则表示该需要保证原子性的操作产生了一系列的redo日志。==

`Mini-Transaction：对底层页面中的一次原子访问的过程`
一个事务可以包含若干条语句，每一条语句其实是由若干个mtr组成，每一个mtr又可以包含若干条redo日志:
![在这里插入图片描述](https://img-blog.csdnimg.cn/8ce7f1bc20044bc380a48af71021fa9e.png)
## redo日志写入过程
### log buffer
服务器启动时就向操作系统申请了一大片称之为redo log buffer的连续内存空间，也就是redo日志缓冲区，把通过mtr生成的redo日志都放在了大小是512字节的页中，结构如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/50308ffbc5aa4946a488e7a81bfdae29.png)
| 名字| 含义| 备注|
|--|--|--|
|LOG_BLOCK_HDR_NO|每一个block都有一个标号值，这个就是标号值|
|LOG_BLOCK_HDR_DATA_LEN|表示block使用了多少字节|
|LOG_BLOCK_FIRST_REC_GROUP |代表该block中第一个mtr生成的redo日志记录组的偏移量|
|LOG_BLOCK_CHECKPOINT_NO| checkpoint序号|
|LOG_BLOCK_CHECKSUM|block的校验值|

服务器在内存中申请一大块内存，log buffer，这片内存空间被划分成若干个连续的redo log block，log buffer大小通过参数innodb_log_buffer_size来设置，默认值16MB，结构如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/b00bc18f18794462a9c6f39f9e0311c0.png)
一个mtr执行过程中可能产生若干条redo日志，这些redo日志是一个不可分割的组，所以其实并不是每生成一条redo日志，就将其插入到log buffer中，而是每个mtr运行过程中产生的日志先暂时存到一个地方，当该mtr结束的时候，将过程中产生的一组redo日志再全部复制到log buffer中，示意图如上图所示。


Log Sequeue Number：日志序列号
buf_next_to_write：标记当前log buffer中已经有哪些日志被刷新到磁盘中来
flushed_to_disk_lsn：刷新到磁盘中的redo日志量

![在这里插入图片描述](https://img-blog.csdnimg.cn/58e8357d2ff54c66b0d56260df6a6873.png)


### 日志文件
磁盘上的redo日志不止一个，而是以一个`日志文件组`的形式出现，结构如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/2de009dfd5e2417bbb91c02208fafca0.png)
将log buffer中的redo日志刷新到磁盘的本质就是把block的镜像写入日志文件中，所以redo日志文件其实也是由若干个 512 字节大小的block组成。
redo日志文件组结构示意图如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/e30dfffb05e74c5cbc31ba773e33af39.png)
redo日志文件前4个block结构：
![在这里插入图片描述](https://img-blog.csdnimg.cn/a6d449c7316e44be8c055aca7fd2b790.png)

<center>log file header结构</center>

|属性名|长度（单位：字节）|描述| 
|:--:|:--:|:--| 
| LOG_HEADER_FORMAT | 4 | redo 日志的版本，在 MySQL5.7.21 中该值永远为1|
| LOG_HEADER_PAD1 | 4 |做字节填充用的，没什么实际意义，忽略～|
| LOG_HEADER_START_LSN | 8 |标记本 redo 日志文件开始的LSN值，也就是文件偏移量为2048字节初对应的LSN值（关于什么是LSN我们稍后再看哈，看不懂的先忽略）。|
| LOG_HEADER_CREATOR | 32 |一个字符串，标记本 redo 日志文件的创建者是谁。正常运行时该值为 MySQL 的版本号，比如： "MySQL 5.7.21" ，使用mysqlbackup 命令创建的 redo 日志文件的该值为 "ibbackup" 和创建时间。|
| LOG_BLOCK_CHECKSUM | 4 |本block的校验值，所有block都有，我们不关心|


<center>check point1&check point2结构</center>

|属性名|长度（单位：字节）|描述|
|:--:|:--:|:--|
| LOG_CHECKPOINT_NO | 8 |服务器做 checkpoint 的编号，每做一次 checkpoint ，该值就加1。|
| LOG_CHECKPOINT_LSN | 8 |服务器做 checkpoint 结束时对应的 LSN 值，系统奔溃恢复时将从该值开始。|
| LOG_CHECKPOINT_OFFSET | 8 |上个属性中的 LSN 值在 redo 日志文件组中的偏移量|
| LOG_CHECKPOINT_LOG_BUF_SIZE | 8 |服务器在做 checkpoint 操作时对应的 log buffer 的大小|
| LOG_BLOCK_CHECKSUM | 4 |本block的校验值，所有block都有，我们不关心|

由于redo日志文件在磁盘中是类似于环型链表的，如果对应的脏页已经写入了磁盘，那么日志文件就可以进行覆盖了，因此判断某些redo日志占用的磁盘空间是否可以覆盖的依据就是它对应的脏页是否已经刷新到磁盘里。
innodb使用checkpoint_lsn来代表当前系统中可以被覆盖的redo日志总量是多少，这个变量初始值也是8704。而计算一次checkpoint_lsn的过程称做一次check_point，其分成两步：
- 步骤一：计算一下当前系统中可以被覆盖的 redo 日志对应的 lsn 值最大是多少。凡是在系统lsn值小于该节点oldest_modification值时产生的redo日志都是可以被覆盖掉的
- 步骤二：将checkpoint_lsn和对应的 redo 日志文件组偏移量以及此次checkpoint的编号写到日志文件的管理信息（就是 checkpoint1或者checkpoint2）中当 checkpoint_no 的值是偶数时，就写到checkpoint1中，是奇数时，就写到checkpoint2中。

这样redo日志组文件中各个lsn值的关系如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/94e535876b6446778070464136ea77ba.png)
## redo日志恢复过程
### 恢复起点
选取最近发生的那次checkpoint的信息，把checkpoint1和checkpoint2这两个block中的checkpoint_no值读出来比一下大小，哪个的checkpoint_no值更大，说明哪个block存储的就是最近的一次checkpoint信息。这样我们就能拿到最近发生的checkpoint对应的checkpoint_lsn值以及它在redo日志文件组中的偏移量checkpoint_offset 。

### 恢复终点
普通block的 log block header 部分有一个称之为 LOG_BLOCK_HDR_DATA_LEN 的属性，该属性值记录了当前block里使用了多少字节的空间。对于被填满的block来说，该值永远为 512 。如果该属性的值不为 512 ，那么就是它了，它就是此次崩溃恢复中需要扫描的最后一个block。
![在这里插入图片描述](https://img-blog.csdnimg.cn/b85b94c586ee40aba9bd74adc2dc5c19.png)
### 怎么恢复
**使用哈希表**
根据 redo 日志的 space ID 和 page number 属性计算出散列值，把 space ID 和 page number 相同的 redo日志放到哈希表的同一个槽里，如果有多个 space ID 和 page number 都相同的 redo 日志，那么它们之间使用链表连接起来，按照生成的先后顺序链接起来的：
![在这里插入图片描述](https://img-blog.csdnimg.cn/a976729f0f44490b849501fbbbf0d282.png)
这样可以一次修复一个页面，避免了很多随机IO。

**跳过已经刷新的页面**
可能后台线程又不断的从 LRU链表 和 flush链表 中将一些脏页刷出 Buffer Pool 。这些在 checkpoint_lsn 之后的 redo 日志，如果它们对应的脏页在奔溃发生时已经刷新到磁盘，那在恢复时也就没有必要根据 redo 日志的内容修改该页面了。

每个页面都有一个称之为 File Header 的部分，在 File Header 里有一个称之为FIL_PAGE_LSN 的属性，该属性记载了最近一次修改页面时对应的 lsn 值（其实就是页面控制块中的newest_modification 值）。如果在做了某次 checkpoint 之后有脏页被刷新到磁盘中，那么该页对应的FIL_PAGE_LSN 代表的 lsn 值肯定大于 checkpoint_lsn 的值，凡是符合这种情况的页面就不需要重复执行lsn值小于 FIL_PAGE_LSN 的redo日志了。

# undo日志
为了保证`原子性`，引入undo日志，把回滚时所需的东西都记录下来，方便后续进行回滚。

## undo日志格式及类型
### 事务id
若某个事务执行过程中对某个表执行了增、删、改操作，那么InnoDB存储引擎会给这个事务分配一个`事务id`，具体的分配策略如下：
- 对于只读事务来说，只有在它第一次对某个用户创建的临时表执行增、删、改操作时才会为这个事务分配一个事务id，否则的话是不分配事务id的；
- 对于读写事务来说，只有在它第一次对某个表（包括用户创建的临时表）执行增、删、改操作时才会为这个事务分配一个事务id，否则的话也是不分配事务id的

事务id的生成策略如下：
- 服务器会在内存中维护一个全局变量，每当需要为某个事务分配一个 事务id 时，就会把该变量的值当作 事务id 分配给该事务，并且把该变量自增1。
- 每当这个变量的值为 256 的倍数时，就会将该变量的值刷新到系统表空间的页号为 5 的页面中一个称之为Max Trx ID 的属性处，这个属性占用 8 个字节的存储空间。
- 当系统下一次重新启动时，会将上边提到的 Max Trx ID 属性加载到内存中，将该值加上256之后赋值给我们前边提到的全局变量（因为在上次关机时该全局变量的值可能大于 Max Trx ID 属性值）

### 日志结构
#### insert
使用TRX_UNDO_INSERT_REC的undo日志，结构如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/fa75bf55ed7441b39a8fbc195596427d.png)
前文中说过有三个隐藏列：
- row_id（用户没有在表中定义主键以及UNIQUE键）
- trx_id
- roll_pointer

roll_pointer的结构如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/a2233f7ca09f4cfbbe4ec137c223952d.png)
本质上是一个指针，指向记录对应的undo日志；
#### delete
使用DELETE操作删除记录，整个删除过程分成两个阶段：
- 阶段一：`delete mark阶段`仅仅将记录的delete_mask标识位设置为 1 ，其他的不做修改；
![在这里插入图片描述](https://img-blog.csdnimg.cn/2718317a0f1d48d7ae425b2a30ec548a.png)
- 阶段二：`purge阶段`当删除语句所在的事务提交之后，有专门的线程将该记录从正常记录链表中移除，并且加入到垃圾链表的头部；
![在这里插入图片描述](https://img-blog.csdnimg.cn/d05ac3bc42d34d868063833a3fec4aa0.png)
使用TRX_UNDO_DEL_MARK_REC类型的undo日志进行记录，结构如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/032f0d569fe74109b1f5ebec62affd9e.png)
其他字段都不难理解，这里介绍一下“索引列各列信息”：
- pos：这个列在记录中的位置
- len：占用存储空间大小
- value：实际值

在删除语句所在的事务提交之前，只会经历delete mark阶段，在delete mark前会记录undo日志。
#### update
##### 不更新主键
- 更新记录与原记录所占存储空间相同，则就地更新

- 更新记录与原记录所占空间不一致，则先删除就记录，再插入新记录
注意这里的删除操作并不是delete mark操作，而是将这条记录从正常记录链表中移除并加入到垃圾链表中，如果新创建的记录占用的存储空间大小不超过旧记录占用的空间，那么可以直接重用被加入到垃圾链表中的旧记录所占用的存储空间，否则的话需要在页面中新申请一段空间以供新记录使用，如果本页面内已经没有可用的空间的话，那就需要进行页面分裂操作，然后再插入新记录。

使用TRX_UNDO_UPD_EXIST_REC类型的undo日志进行记录，结构如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/858f3129cca04361891c7719cf9e03ef.png)
##### 更新主键
InnoDB在聚簇索引中分两步处理：
- 将旧记录进行delete mark操作
- 根据更新后各列的值创建一条新记录，并将其插入到聚簇索引中

针对 UPDATE 语句更新记录主键值的这种情况，
在对该记录进行delete mark操作前，会记录一条类型为TRX_UNDO_DEL_MARK_REC 的undo日志 ；
之后插入新记录时，会记录一条类型为TRX_UNDO_INSERT_REC的undo日志 ；
也就是说每对一条记录的主键值做改动时，会记录2条undo日志。

## undo日志写入过程
### undo日志结构
InnoDB使用FIL_PAGE_UNDO_LOG类型的页面存储undo日志，其结构如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/f12f5951c71e482ea8ae19151b6f68cd.png)
|名字|大小|功能|备注|
|--|--|--|--|
|TRX_UNDO_PAGE_TYPE|2字节|记录存储什么类型的undo日志|1️⃣TRX_UNDO_INSERT(使用十进制1表示)：类型为TRX_UNDO_INSERT_REC的undo日志属于此大类<br>2️⃣TRX_UNDO_UPDATE （使用十进制 2 表示）：除了类型为 TRX_UNDO_INSERT_REC 的 undo日志 ，其他类型的 undo日志 都属于这个大类
|TRX_UNDO_PAGE_START|2字节|表示当前页面从什么位置开始存储undo日志|
|TRX_UNDO_PAGE_FREE|2字节|表示当前页面中存储的最后一条undo日志结束时的偏移量|
|TRX_UNDO_PAGE_NODE|12字节|表示一个List Node结构|

**TRX_UNDO_PAGE_NODE字段详解：**
看上表中的TRX_UNDO_PAGE_TYPE字段，undo日志格式有两种，所以一个事务的执行可能需要2个undo页面的链表，一个称作insert undo链表，一个称作update链表，同时，innoDB对于普通表和临时表的undo日志使用两条链表进行存储，因此innoDB共有四条链表记录undo日志，如下图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/819b55f00839450e94b431636054128c.png)
这四个链表并不是从一开始就存在的，而是：按需分配，啥时候需要啥时候再分配，不需要就不分配。
undo链表的第一个页面的类型是first undo page，其余的页面都是normal undo page，first undo page的结构如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/10a6cb0b8f3045c68667cc52dc0566c5.png)
undo页面链表示意图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/2e533d88bd1d4fcaab8838ea30e1b797.png)
### undo页面重用
如果为每一个事务都分配四条链表，那对存储空间会比较浪费，在某些事务提交之后可以对undo页面链表进行重用，一个undo页面链表是否可以重用条件如下：
- 该链表只包含一个undo页面；
- 该undo页面已经使用的空间小于整个页面空间的3/4；
	- insert undo链表
insert undo链表 中只存储类型为 TRX_UNDO_INSERT_REC 的 undo日志 ，这种类型的undo日志在事务提交之后就没用了，就可以被清除掉。所以在某个事务提交后，重用这个事务的insert undo链表 （这个链表中只有一个页面）时，可以直接把之前事务写入的一组 undo日志覆盖掉，从头开始写入事务的一组 undo日志 ，如下图所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/6e3351bebc194e14915f784d1422de84.png)
	- update undo链表
在一个事务提交后，它的 update undo链表 中的 undo日志 也不能立即删除掉（这些日志用于MVCC）。所以如果之后的事务想重用update undo链表时，就不能覆盖之前事务写入的 undo日志 。这样就相当于在同一个Undo页面 中写入了多组的undo日志，效果看起来就是这样：
![在这里插入图片描述](https://img-blog.csdnimg.cn/75903b63b1b94199bfd51420d8b96657.png)
### 回滚段
Rollback Segment Header：回滚段，同一时刻系统里面会有多个undo页面链表存在，为了管理这些链表，引入`回滚段`的概念。每一个 Rollback Segment Header 页面都对应着一个段，这个段就称为 Rollback Segment ，翻译过来就是`回滚段`。
结构如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/e4c8948dd73f41e5941d68695c2276e4.png)
Rollback Segment Header中的TRX_RSEG_UNDO_SLOTS存储undo链表的first page的页号。

#### 从回滚段中申请undo链表
初始条件下，各个 undo slot 都被设置成了一个特殊的值： FIL_NULL （对应的十六进制就是 0xFFFFFFFF ），表示该undo slot 不指向任何页面。

**分配**
有事务需要分配Undo页面链表时，就从回滚段的第一个undo slot开始，看看该undo slot的值是不是FIL_NULL：
- 如果是 FIL_NULL ，那么在表空间中新创建一个段（也就是 Undo Log Segment ），然后从段里申请一个页面作为Undo页面链表的 first undo page ，然后把该 undo slot 的值设置为刚刚申请的这个页面的地址，这样也就意味着这个undo slot被分配给了这个事务。
- 如果不是 FIL_NULL ，说明该 undo slot 已经指向了一个 undo链表 ，也就是说这个 undo slot 已经被别的事务占用了，那就跳到下一个 undo slot ，判断该 undo slot 的值是不是 FIL_NULL ，重复上边的步骤。

一个 Rollback Segment Header 页面中包含 1024 个undo slot，如果这1024个undo slot的值都不为FIL_NULL ，这就意味着这1024个 undo slot 都已经名花有主（被分配给了某个事务），此时由于新事务无法再获得新的Undo页面 链表，就会回滚这个事务并且给用户报错；

**事务提交**
那么说完分配，事务提交之后它所占用的undo slot怎么处理呢，分成以下两种情况：
- 若该undo slot指向的undo页面链表符合重用条件；
	- 如果对应的 Undo页面 链表是 insert undo链表 ，则该 undo slot 会被加入 insert undo cached链表
	- 如果对应的 Undo页面 链表是 update undo链表 ，则该 undo slot 会被加入 update undo cached链表

这样该undo slot就处于被缓存的状态，TRX_UNDO_STATE会被设置成TRX_UNDO_CACHED，一个回滚段就对应着上述两个 cached链表 ，如果有新事务要分配undo slot 时，先从对应的cached链表中找。如果没有被缓存的 undo slot ，才会到回滚段的Rollback Segment Header页面中再去找。

- 若该undo slot指向的undo页面链表不符合重用条件；
	- 如果对应的 Undo页面 链表是 insert undo链表 ，则该 Undo页面 链表的TRX_UNDO_STATE 属性会被设置为 TRX_UNDO_TO_FREE ，之后该 Undo页面 链表对应的段会被释放掉（也就意味着段中的页面可以被挪作他用），然后把该 undo slot 的值设置为FIL_NULL
	- 如果对应的 Undo页面 链表是update undo链表 ，则该Undo页面链表的TRX_UNDO_STATE 属性会被设置为 TRX_UNDO_TO_PRUGE ，则会将该 undo slot 的值设置为 FIL_NULL ，然后将本次事务写入的一组undo 日志放到所谓的 History链表 中

在系统表空间的第5号页面中存储了128个 Rollback Segment Header页面地址，每个Rollback Segment Header就相当于一个回滚段。在Rollback Segment Header页面中，又包含1024 个undo slot，每个undo slot都对应一个Undo页面链表，如图所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/bd9e0f223b7c427a9fe67a1ae15944a5.png)

**回滚段的分类：**
- 第 0 号、第 33～127 号回滚段属于一类。其中第 0 号回滚段必须在系统表空间中（就是说第 0 号回滚段对应的 Rollback Segment Header 页面必须在系统表空间中），第 33～127号回滚段既可以在系统表空间中，也可以在自己配置的 undo 表空间中，如果一个事务在执行过程中由于对普通表的记录做了改动需要分配 Undo页面 链表时，必须从这一类的段中分配相应的undo slot
- 第 1～32 号回滚段属于一类。这些回滚段必须在临时表空间（对应着数据目录中的ibtmp1 文件）中。如果一个事务在执行过程中由于对临时表的记录做了改动需要分配Undo页面链表时，必须从这一类的段中分配相应的undo slot

回滚段分类的原因：在修改针对普通表的回滚段中的Undo页面时，需要记录对应的redo日志，而修改针对临时表的回滚段中的Undo页面时，不需要记录对应的redo日志

**事务执行过程中分配Undo页面链表时的完整过程：**
- 事务在执行过程中对普通表的记录首次做改动之前，首先会到系统表空间的第 5 号页面中分配一个回滚段（其实就是获取一个 Rollback Segment Header 页面的地址）。一旦某个回滚段被分配给了这个事务，那么之后该事务中再对普通表的记录做改动时，就不会重复分配了。
使用传说中的 round-robin （循环使用）方式来分配回滚段。比如当前事务分配了第 0 号回滚段，那么下一个事务就要分配第 33 号回滚段，下下个事务就要分配第 34 号回滚段，简单一点的说就是这些回滚段被轮着分配给不同的事务（就是这么简单粗暴，没啥好说的）。
- 在分配到回滚段后，首先看一下这个回滚段的两个 cached链表 有没有已经缓存了的 undo slot ，比如如果事务做的是 INSERT 操作，就去回滚段对应的 insert undo cached链表 中看看有没有缓存的 undo slot ；如果事务做的是 DELETE 操作，就去回滚段对应的 update undo cached链表 中看看有没有缓存的 undo slot 。如果有缓存的 undo slot ，那么就把这个缓存的 undo slot 分配给该事务。
- 如果没有缓存的 undo slot 可供分配，那么就要到 Rollback Segment Header 页面中找一个可用的 undo slot 分配给当前事务。从 Rollback Segment Header 页面中分配可用的 undo slot 的方式我们上边也说过了，就是从第 0 个 undo slot 开始，如果该 undo slot 的值为 FIL_NULL ，意味着这个 undo slot 是空闲的，就把这个 undo slot
分配给当前事务，否则查看第 1 个 undo slot 是否满足条件，依次类推，直到最后一个 undo slot 。如果这1024个undo slot都没有值为 FIL_NULL 的情况，就直接报错喽（一般不会出现这种情况）～
- 找到可用的 undo slot 后，如果该 undo slot 是从 cached链表 中获取的，那么它对应的 Undo Log Segment 已经分配了，否则的话需要重新分配一个 Undo Log Segment ，然后从该 Undo Log Segment 中申请一个页面作为 Undo页面 链表的 first undo page 。
- 然后事务就可以把 undo日志 写入到上边申请的Undo页面链表了


# 关于事务的讨论

## 事务的问题
对于mysql中的事务并发执行过程中有以下几个问题：
- 脏写：一个事务修改了另一个未提交事务修改过的数据；
- 脏读：一个事务中读到未提交事务中修改过的数据；
- 不可重复读：一个事务<font color=red>**只能读取到另一个已经提交的事务修改过的数据**</font>，并且其他事务每对该数据修改一次并提交之后，该事务都能读取到最新的值；
- 幻读：一个事务按照某种条件查询出一些记录，另一个事务又向表中插入了一些符合这些条件的记录，原先的事务再次根据该条件查询时，能把另一个事务的插入记录也读取出来；

下面对这几种问题进行举例说明：

<center>脏写示意图</center>

|发生时间|sessionA|sessionB|
|--|--|--|
|1 |BEGIN | |
| 2| |BEGIN |
| 3| |UPDATE hero SET name="tom" where number=1 |
| 4| UPDATE hero SET name="bill" where number=1| |
| 5| COMMIT| |
| 6| |ROLLBACK ||

如果之后 Session B 中的事务进行了回滚，那么 Session A 中的更新也将不复存在，这种现象就称作`脏写`。
<center>脏读示意图</center>

|发生时间|sessionA|sessionB|
|--|--|--|
|1 |BEGIN | |
| 2| |BEGIN |
| 3| |UPDATE hero SET name="tom" where number=1 |
| 4| SELECT * FROM hero WHERE number=1<br>(此时读到的name是“tom”，则意味着出现了`脏读`)| |
| 5| COMMIT| |
| 6| |ROLLBACK ||

而Session B中的事务稍后进行了回滚，那么Session A中的事务相当于读到了一个不存在的数据，这种现象就称作`脏读`
<center>不可重复读示意图</center>

|发生时间|sessionA|sessionB|
|--|--|--|
| 1| BEGIN| |
| 2| SELECT * FROM hero WHERE number=1<br>(此时读到的name是“sam”)| |
| 3| | UPDATE hero SET name="tom" where number=1|
| 4| SELECT * FROM hero WHERE number=1<br>(此时读到的name是“tom”，则意味着出现了`不可重复读`)| |
| 5| |UPDATE hero SET name="bill" where number=1 |
| 6|SELECT * FROM hero WHERE number=1<br>(此时读到的name是“bill”，则意味着出现了`不可重复读`)| |


<center>幻读示意图</center>

|发生时间|sessionA|sessionB|
|--|--|--|
| 1| BEGIN| |
| 2| SELECT * FROM hero WHERE number>0<br>(此时读到的name是sam)| |
| 3| | UPDATE hero VALUE(2,'tom','china')|
| 4|SELECT * FROM hero WHERE number>0<br>(此时读到的name是sam和tom，则意味着出现了`幻读`) | |

` 幻读`强调的是一个事务按照某个相同条件多次读取记录时，后读取时读到了之前没有读到的记录，也就是说如果 Session B 中是删除了一些符合 number > 0 的记录而不是插入新记录，那Session A 中之后再根据 number > 0 的条件读取的记录变少了，这种现象不属于`幻读`。

## 事务问题解决方式
InnoDB提供了几种事务隔离级别，如下表所示：

|隔离级别| 脏读| 不可重复读| 幻读|
|--|--|--|--|
|READ UNCOMMITTED | ✅|✅ |✅ |
| READ COMMITTED| ❌| ✅|✅ |
|REPEATABLE READ |❌ |❌ |✅ |
| SERIALIZABLE| ❌| ❌| ❌|

## 事务隔离级别的设置方式
```sql
SET [GLOBAL|SESSION] TRANSACTION ISOLATION LEVEL level
//其中level选项如下
level: {
REPEATABLE READ
| READ COMMITTED
| READ UNCOMMITTED
| SERIALIZABLE
}
```
**GLOBAL参数：**
- 只对执行完该语句之后产生的会话起作用。
- 当前已经存在的会话无效

**SESSION参数：**
- 对当前会话的所有后续的事务有效
- 该语句可以在已经开启的事务中间执行，但不会影响当前正在执行的事务。
- 如果在事务之间执行，则对后续的事务有效

**上述两个关键字都不用：(只对执行语句后的下一个事务产生影响)**
- 只对当前会话中下一个即将开启的事务有效。
- 下一个事务执行完后，后续事务将恢复到之前的隔离级别。
- 该语句不能在已经开启的事务中间执行，会报错的

对于SESSION和连接的关系，这里简单讨论一下:
会话(Session) 是通信双方从开始通信到通信结束期间的一个上下文（Context）。这个上下文是一段位于服务器端的内存：记录了本次连接的客户端机器、通过哪个应用程序、哪个用户登录等信息

连接（Connection）：连接是从客户端到ORACLE实例的一条物理路径。连接可以在网络上建立，或者在本机通过IPC机制建立。通常会在客户端进程与一个专用服务器或一个调度器之间建立连接。

会话(Session) 是和连接(Connection)是同时建立的，两者是对同一件事情不同层次的描述。简单讲，连接(Connection)是物理上的客户端同服务器的通信链路，会话(Session)是逻辑上的用户同服务器的通信交互。

在一条连接而无相应的会话。另外，一个会话可以有连接也可以没有连接。使用高级Oracle Net特性（如连接池）时，客户可以删除一条物理连接，而会话依然保留（但是会话会空闲）。客户在这个会话上执行某个操作时，它会重新建立物理连接
```
有A/B两个城市，需要从A运送白菜到B城

   我们先建设一条公路
   然后运送白菜过去，包括准备白菜和运送白菜以及返回等一系列的动作。

   一条公路，可以运送0-n次的白菜
   当然从A到B的公路也可能不只一条。
   某一次运送白菜，可以在真正上路时才开通某一条道路
   一次运送不会影响别的运送的状态


   对应数据库 
   A代表客户端进程 
   B代表服务器端进程
   公路代表连接，
   运送一次白菜代表一个会话

   一个连接可以进行多次的会话
   一个会话可以不依赖于某个连接，甚至没有连接(当我准备好了，真正开始运送时再建立连接)
   一个会话不会影响别的会话
  ```
## 事务隔离级别的实现原理
InnoDB使用MVCC机制来实现事务的隔离级别，为了说明MVCC机制，我们先来介绍两个概念`版本链`和`ReadView`。
### 版本链
两个隐藏列：
- trx_id ：每次一个事务对某条聚簇索引记录进行改动时，都会把该事务的 事务id 赋值给trx_id 隐藏列。
- roll_pointer ：每次对某条聚簇索引记录进行改动时，都会把旧的版本写入到 undo日志中，然后这个隐藏列就相当于一个指针，可以通过它来找到该记录修改前的信息

==实际上insert undo只在事务回滚时起作用，当事务提交后，该类型的undo日志就没用了，它占用的Undo Log Segment也会被系统回收==

假设之后两个 事务id 分别为 100 、 200 的事务对这条记录进行 UPDATE 操作，操作流程如下：![在这里插入图片描述](https://img-blog.csdnimg.cn/cf4e8b4691de4720b4c86c482a51b5c7.png)
undo日志情况：
![在这里插入图片描述](https://img-blog.csdnimg.cn/f90e8a18f63e4180a35bcdba533cc6cf.png)

### ReadView
ReadView中四个重要内容：
- m_ids ：表示在生成ReadView时当前系统中活跃的读写事务的事务id列表。
- min_trx_id ：表示在生成ReadView时当前系统中活跃的读写事务中最小的事务id ，也就是m_ids中的最小值。
- max_trx_id ：表示生成ReadView时系统中应该分配给下一个事务的id值
- creator_trx_id ：表示生成该ReadView的事务的 事务id 。

判断规则：
- 如果被访问版本的trx_id属性值与ReadView中的creator_trx_id值相同，意味着当前事务在访问它自己修改过的记录，所以该版本可以被当前事务访问。
- 如果被访问版本的trx_id 属性值小于 ReadView 中的 min_trx_id 值，表明生成该版本的事务在当前事务生成 ReadView 前已经提交，所以该版本可以被当前事务访问。
- 如果被访问版本的 trx_id 属性值大于 ReadView 中的 max_trx_id 值，表明生成该版本的事务在当前事务生成 ReadView 后才开启，所以该版本不可以被当前事务访问。
- 如果被访问版本的 trx_id 属性值在 ReadView 的 min_trx_id 和 max_trx_id 之间，那就需要判断一下trx_id 属性值是不是在 m_ids 列表中，如果在，说明创建 ReadView 时生成该版本的事务还是活跃的，该版本不可以被访问；如果不在，说明创建 ReadView 时生成该版本的事务已经被提交，该版本可以被访问。

==READ COMMITTED 和 REPEATABLE READ 隔离级别的的一个非常大的区别就是它们生成ReadView的时机不同==
### 实际情况分析

#### READ COMMITTED —— 每次读取数据前都生成一个ReadView
```sql
# Transaction 100
BEGIN;
UPDATE hero SET name = '关羽' WHERE number = 1;
UPDATE hero SET name = '张飞' WHERE number = 1;
# Transaction 200
BEGIN;
# 更新了一些别的表的记录
...

# 使用READ COMMITTED隔离级别的事务
BEGIN;
# SELECT1：Transaction 100、200未提交
SELECT * FROM hero WHERE number = 1; # 得到的列name的值为'刘备'

# Transaction 100
BEGIN;
UPDATE hero SET name = '关羽' WHERE number = 1;
UPDATE hero SET name = '张飞' WHERE number = 1;
COMMIT;


# Transaction 200
BEGIN;
# 更新了一些别的表的记录
...
UPDATE hero SET name = '赵云' WHERE number = 1;
UPDATE hero SET name = '诸葛亮' WHERE number = 1;

# 使用READ COMMITTED隔离级别的事务
BEGIN;
# SELECT1：Transaction 100、200均未提交
SELECT * FROM hero WHERE number = 1; # 得到的列name的值为'刘备'
# SELECT2：Transaction 100提交，Transaction 200未提交
SELECT * FROM hero WHERE number = 1; # 得到的列name的值为'张飞'
```
事务2执行完之后（未提交），版本链如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/a7361b9d54994d4b9bbcdb665f7617ac.png)
**SELECT1分析：**
- 生成ReadView，max_trx_id=201，m_ids=[100,200]，min_trx_id=100，creator_trx_id=0
- “诸葛亮”、“赵云”、“张飞”、“关羽”记录的trx_id都在m_ids内，所以不可见；
- “刘备”记录的trx_id小于min_trx_id，可见；

**SELECT2分析：**
- 生成ReadView，max_trx_id=201，m_ids=[200]，min_trx_id=200，creator_trx_id=0
- “诸葛亮”、“赵云”记录的trx_id都在m_ids内，所以不可见；
- “张飞”记录的trx_id小于min_trx_id，可见；
#### REPEATABLE READ —— 在第一次读取数据时生成一个ReadView
分析过程与上述类似，只是在第一次读取数据的时候生成ReadView，后续直接使用，不再重新生成。

# 关于锁的讨论
有几种并发场景，读-读、读-写、写-写；
解决脏读、不可重复读、幻读的问题，有两种方案：
- 方案一：读操作使用MVCC，写操作进行加锁，采用MVCC时，读-写操作不冲突；
- 方案二：读、写操作都使用加锁的方式；

一般情况下，使用MVCC来解决读-写并发执行的问题，但是有些特殊的业务场景，要求必须使用加锁的方式执行，也没办法；

## 锁定读
有两类锁：
- 共享锁：S锁；
- 独占锁：X锁；

|兼容性|X|S|
|--|--|--|
|X|❌|❌|
|S|❌|✅|

```sql
// 对读取记录加S锁
SELECT ... LOCK IN SHARE MODE;

// 对读取记录加X锁
SELECT ... FOR UPDATE;
```

## 锁定写
- DELETE
**流程**：在B+树中定位记录位置➡️获取该记录X锁➡️执行delete mark
可以看成是一个`获取X锁的锁定读`
- UPDATE：
	- 修改前后存储空间不变
**流程**：在B+树中定位记录位置➡️获取该记录X锁➡️执行修改操作
可以看成是一个`获取X锁的锁定读`
 	- 修改前后至少有一列的存储空间发生变化；
**流程**：在B+树中定位记录位置➡️获取该记录X锁➡️将该记录彻底删除➡️查询一条新记录
可以看成是一个`获取x锁的锁定读` ，新插入的记录由INSERT 操作提供的`隐式锁`进行保护
	- 修改了该记录的键值
相当于在原记录上做一次DELETE和一次INSERT，详细过程可以参考上面的流程
- INSERT
一般情况下，新插入一条记录的操作并不加锁，通过`隐式锁`的方式来保护这条新插入的记录在本事务提交前不被别的事务访问

## 多粒度锁
- IS：意向共享锁；
- IX：意向独占锁；
IS、IX锁是表级锁，它们的提出仅仅为了在之后加表级别的S锁和X锁时可以快速判断表中的记录是否被上锁，以避免用遍历的方式来查看表中有没有上锁的记录，也就是说其实IS锁和IX锁是兼容的，IX锁和IX锁是兼容的。

|兼容性|X|IX|S|IS|
|--|--|--|--|--|
|X|❌|❌|❌|❌|
|IX|❌|✅|❌|✅|
|S|❌|❌|✅|✅|
|IS|❌|✅|✅|✅|


## MYSQL中的行锁和表锁
### 表锁
在表上加的锁。
- 表级别的S锁、X锁
DDL语句是在server层使用一种`元数据锁`方式来实现，一般情况下不会使用InnoDB自己提供的表级别的S锁和X锁；
实际情况中要避免使用表级别的S锁和X锁，对性能的影响很大；

- 表级别的IS锁、IX锁
IS锁和IX锁的使命只是为了后续在加表级别的 S锁 和 X锁 时判断表中是否有已经被加锁的记录，以避免用遍历的方式来查看表中有没有上锁的记录

- 表级别的AUTO-INC锁
用于系统实现自动给AUTO_COMMENT修饰的列递增赋值，原理有两个：
	- 采用 AUTO-INC 锁，也就是在执行插入语句时就在表级别加一个 AUTO-INC 锁，然后为每条待插入记录的AUTO_INCREMENT 修饰的列分配递增的值，在该语句执行结束后，再把 AUTO-INC 锁释放掉。这样一个事务在持有 AUTO-INC 锁的过程中，其他事务的插入语句都要被阻塞，可以保证一个语句中分配的递增值是连续的
	- 采用一个轻量级的锁，在为插入语句生成 AUTO_INCREMENT 修饰的列的值时获取一下这个轻量级锁，然后生成本次插入语句需要用到的 AUTO_INCREMENT 列的值之后，就把该轻量级锁释放掉，并不需要等到整个插入语句执行完才释放锁

配置方式：
当innodb_autoinc_lock_mode值为0时，一律采用AUTO-INC锁；当innodb_autoinc_lock_mode值为2时，一律采用轻量级锁；当innodb_autoinc_lock_mode值为1时，两种方式混着来（也就是在插入记录数量确定时采用轻量级锁，不确定时使用AUTO-INC锁）
### 行锁
在记录上加的锁。
|类型|官方命名|作用|备注|
|--|--|--|--|
|Record Locks|LOCK_REC_NOT_GAP||仅仅把一条记录锁上|
|Gap Locks|LOCK_GAP|为了防止插入幻影记录引入的|锁住记录`间隙`<br>使用Infimum记录(页面中最小记录)和Superemum(页面中最大记录)锁两端的间隙|
|Next-Key Locks|LOCK_ORDINARY|既想锁住某条记录，又想阻止其他事务在该记录前边的间隙插入新记录|
|Insert Intention Locks|LOCK_INSERT_INTENTION|事务在等待的时候也需要在内存中生成一个锁结构，表明有事务想在某个间隙中插入新记录，但是现在在等待|插入意向锁,并不会阻止别的事务继续获取该记录上任何类型的锁|

## 锁的结构
不同记录的锁，满足下列条件时，那么这些记录的锁就可以放到一个锁结构里：
- 在同一个事务中进行加锁操作
- 被加锁的记录在同一个页面中
- 加锁的类型是一样的
- 等待状态是一样的
![在这里插入图片描述](https://img-blog.csdnimg.cn/574f9b294c5f42b69575e4ae8376061e.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/f9b4b6509220448cb1488fda940ade8c.png)





