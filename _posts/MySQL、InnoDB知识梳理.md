# MySQL、InnoDB知识梳理

## 一、字符集和比较规则

1. 定义：字符集指的是某个字符范围的编码规则，比较规则是对某个字符集中的字符比较大小的一种规则。
2. MySQL有4个级别的字符集和比较规则：1、服务器级别；2、数据库级别；3、表级别；4、列级别。
3. 从发送请求到接收响应的过程中发生的字符集转换如下：
   1. 客户端采用操作系统当前使用的字符集对发送的请求字节序列进行编码；
   2. 服务器接收到请求字节序列后认为它是采用系统变量character_set_client值的字符集进行编码的；
   3. 服务器在处理请求时会把请求字节序列转换为以系统变量character_set_connection值的字符集编码的字节序列；
   4. 服务器在向客户端返回字节序列时，是采用系统变量character_set_results值的字符集进行编码的；
   5. 客户端在收到响应字节序列后，采用操作系统当前使用的字符集进行解码。

## 二、InnoDB存储结构

1. InnoDB行格式：记录在磁盘上的存放形式。
   1. COMPACT行格式
   ![COMPACT行格式示意图](https://cdn.jsdelivr.net/gh/n20u/PicBed/blogs/knowledge_combing/20220620163739.png)
      1. 记录的额外信息
         1. 变长字段长度列表：所有变长字段的真实数据占用的字节数，按照列的顺序逆序存放在记录的开头位置。
         2. NULL值列表：每个允许存储NULL的列对应一个二进制位，二进制位按照列的顺序逆序排列。该列的值为NULL时，对应的二进制位为1。
         3. 记录头信息
         ![记录头信息示意图](https://cdn.jsdelivr.net/gh/n20u/PicBed/blogs/knowledge_combing/20220620164810.png)

         |  名称  |大小（位）|          描述          |
         |:------:|:-------:|:----------------------|
         |预留位1|    1     |        没有使用         |
         |预留位2|    1     |        没有使用         |
         |deleted_flag|  1  |  标记该记录是否被删除    |
         |min_rec_flag|  1  |B+树的每层非叶子节点中最小的目录项记录都会添加该标记|
         |n_owned|4|一个页面中的记录会被分成若干个组，每个组中有一个记录是“带头大哥”，其余的记录都是“小弟”。“带头大哥”记录的n_owned值代表该组中所有的记录条数，“小弟”记录的n_owned值都为0|
         |heap_no|   13    |表示当前记录在页面堆中的相对位置|
         |record_type|  3  |表示当前记录的类型，0表示普通记录，1表示B+树非叶子节点的目录项记录，2表示Infimum记录，3表示Supremum记录|
         |next_record|  16  |表示下一条记录的相对位置|
      2. 记录的真实数据
        - 记录的真实数据除了用户定义的列的数据外，MySQL会为每个记录默认地添加一些列（也称为隐藏列），具体的列如表所示：

        |  列名  |是否必需|占用空间|    描述    |
        |:------:|:----:|:------:|:----------:|
        |DB_ROW_ID|  否  | 6字节 |行ID，唯一标识一条记录|
        |DB_TRX_ID|  是  | 6字节 |   事务ID   |
        |DB_ROLL_PTR| 是 | 7字节 |   回滚指针  |
        - InnoDB表的主键生成策略：优先使用用户自定义的主键作为主键；如果用户没有定义主键，则选取一个不允许存储NULL值的UNIQUE键作为主键；如果表中连不允许存储NULL值的UNIQUE键都没有定义，则InnoDB会为表默认添加一个名为DB_ROW_ID的隐藏列作为主键。
   2. REDUNDANT行格式
   ![REDUNDANT行格式示意图](https://cdn.jsdelivr.net/gh/n20u/PicBed/blogs/knowledge_combing/20220620170726.png)
      1. 记录的额外信息
         1. 字段长度偏移列表：所有列（包括隐藏列）的偏移量都按照逆序存储到字段长度偏移列表，偏移量指的是该列的值占用的空间在记录的真实数据处结束的位置。
         2. 记录头信息：REDUNDANT行格式的记录头信息与COMPACT行格式的记录头信息对比来看，多了n_field和1byte_offs_flag这两个属性，少了record_type这个属性。
      2. 记录的真实数据
      REDUNDANT行格式将列对应的偏移量值的第一个比特位作为是否为NULL的依据，该比特位也可以称之为NULL比特位。如果存储NULL值的字段是定长类型的，则NULL值也将占用记录的真实数据部分，并把该字段对应的数据使用0x00字节填充，否则不在记录的真实数据部分占用任何存储空间。
   3. DYNAMIC行格式和COMPRESSED行格式
   COMPRESSED行格式会采用压缩算法对页面进行压缩，以节省空间。
   4. 溢出列：在COMPACT和REDUNDANT行格式中，对于占用存储空间非常多的列，在记录的真实数据处只会存储该列前768字节的数据，而把剩余的数据分散存储在几个其他的页中，然后在记录的真实数据处用20字节存储指向这些页的地址（这20字节还包括分散在其他页面中的数据所占用的字节数）。DYNAMIC和COMPRESSED行格式把该列的所有真实数据都存储到溢出页中，只在记录的真实数据处存储20字节大小的指向溢出页的地址。
2. InnoDB索引页：存放记录的页。
   ![InnoDB数据页结构示意图](https://cdn.jsdelivr.net/gh/n20u/PicBed/blogs/knowledge_combing/20220620192151.png)
   1. File Header（文件头部）：表示页的一些通用信息。
   2. Page Header（页面头部）：表示数据页专有的一些信息。
   3. Infimum + Supremum：两个伪记录，分别表示页中的最小记录和最大记录。
   4. User Records：真正存储用户插入的记录。
   5. Free Space：页中尚未使用的部分。
   6. Page Directory（页目录）
      1. 组成：把页中所有正常的记录划分为若干个组，每个组的最后一条记录的头信息中的n_owned属性表示该组内共有几条记录。将每个组的最后一条记录在页面中的地址偏移量作为一个槽（Slot），按顺序存储到Page Directory（页目录）中，一个槽占用2字节。
      2. 在一个数据页中查找指定主键值的记录的过程：
         1. 通过二分法确定该记录所在分组对应的槽，然后找到该槽所在分组中主键值最小的那条记录；
         2. 通过记录的next_record属性遍历该槽所在的组中的各个记录。
   7. File Trailer（文件尾部）：前4字节代表页的校验和，与File Header中的校验和相对应；后4字节代表页面被最后修改时对应的LSN的后4字节，与File Header部分的FIL_PAGE_LSN的后4字节相对应。如果这两部分与File Header中的对应部分不相同，则意味着刷新期间发生了错误。用于检验页是否完整。
3. Buffer Pool
   1. 定义：为了缓存磁盘中的页，InnoDB存储引擎向操作系统申请的一片连续的内存。
   2. 组成：缓冲页和对应的包含控制信息的控制页，这些控制信息包括该页所属的表空间编号、页号、缓冲页在Buffer Pool中的地址、链表节点信息等。
   3. 管理链表：
      1. free链表：每一个节点都代表一个空闲的缓冲页对应的控制块，在将磁盘中的页加载到Buffer Pool中时，会从free链表中寻找空闲的缓冲页。
      2. flush链表：每一个节点都代表一个被修改过的缓冲页对应的控制块。
      3. LRU链表：分为young区域和old区域。首次从磁盘加载到Buffer Pool中的页会放到old区域的头部，在一定的间隔时间内访问该页时，不会把它移动到young区域头部。在Buffer Pool中没有可用的空闲缓冲页时，会首先淘汰掉old区域中的一些页。
   4. 为了快速定位某个页是否被加载到Buffer Pool中，可使用表空间号 + 页号作为key，缓冲页控制块的地址作为value的形式来建立哈希表。

## 三、索引

InnoDB存储引擎支持B+树索引，哈希索引和全文索引。

1. B+树索引
  InnoDB存储引擎中的B+树索引可以分为聚集索引和辅助索引。
   1. 聚簇索引
      - 使用记录主键值的大小进行记录和页的排序：
        - 页内的记录按照主键的大小顺序排成一个单向链表；
        - 各个存放用户记录的页和同一层级中存放目录项记录的页也是根据页中记录的主键大小顺序排成一个双向链表。
      - B+树的叶子节点存储的是完整的用户记录，内部节点存储的是主键值 + 子节点的编号的目录项记录。
      ![聚簇索引](https://cdn.jsdelivr.net/gh/n20u/PicBed/blogs/knowledge_combing/20220621112053.png)
   2. 辅助索引
      - 使用记录索引列的大小进行记录和页的排序：
        - 页内的记录按照索引列的大小顺序排成一个单向链表；
        - 各个存放用户记录的页和同一层级中存放目录项记录的页也是根据页中记录的索引列的大小顺序排成一个双向链表。
      - B+树的叶子节点存储的是索引列 + 主键值的用户记录，内部节点存储的是索引列 + 主键值 + 子节点的编号的目录项记录。
      - 使用辅助索引查找完整的用户记录时，先在辅助索引的B+树的叶子节点中查找到该记录的主键值，然后根据该主键值到聚簇索引中查找到完整的用户记录。
   3. 联合索引：同时以多个列的大小作为排序规则，也就是同时为多个列建立索引。
      最左前缀匹配：从最左边索引列开始连续匹配，遇到范围查询（<、>、between、like）会停止匹配。
   4. 覆盖索引：索引中已经包含所有需要读取的列的查询方式。
2. 哈希索引
InnoDB存储引擎支持的哈希索引是自适应的，会监控对表上各索引页的查询，如果观察到建立哈希索引可以带来速度提升，则建立哈希索引，因此称之为自适应哈希索引。不能人为干预是否在一张表中生成哈希索引。
3. 全文索引
全文索引通常使用倒排索引来实现。倒排索引同B+树索引一样，也是一种索引结构。它在辅助表中存储了单词与单词在文档中的位置之间的映射。辅助表是存放于磁盘上的持久表，还有一个全文检索索引缓存（FTS Index Cache），它是一个红黑树结构，其根据单词与单词位置的映射排序，用来提高全文检索的性能。
4. B+树索引和哈希索引的优劣势对比
   1. B+树索引支持范围查询、排序和分组，哈希索引不支持。
   2. B+树索引支持模糊查询和联合索引的最左前缀匹配，哈希索引不支持。
   3. 哈希索引查询效率优于B+树索引。
   4. 由于哈希表中存在哈希冲突，所以哈希索引的性能不稳定，而B+树索引的性能相对稳定。
5. 索引的代价：
   1. 空间上的代价：每建立一个索引，都要为它建立一棵B+树。
   2. 时间上的代价：每当对表中的数据进行增删改操作时，都需要修改各个B+树索引。如果建了太多索引，还可能会导致执行查询语句前的成本分析过程耗时太多，从而影响查询语句的执行性能。

## 四、表访问

1. 单表访问方法：const、ref、ref_or_null、range、index、all、index_merge。
   索引合并：Intersection索引合并、Union索引合并、Sort-Union索引合并。
   优化器生成执行计划的步骤：
   1. 根据搜索条件，找出所有可能使用的索引；
   2. 计算全表扫描的代价；
   3. 计算使用不用索引执行查询的代价；
   4. 对比各种执行方案的代价，找出成本最低的那个方案。

   在优化器生成执行计划的过程中，需要依赖一些数据：
   - index dive：通过直接访问索引对应的B+树来获取数据。
   - 索引统计数据：直接依赖对表或者索引的统计数据。

   InnoDB以表为单位来收集统计数据。这些统计数据可以是基于磁盘的永久性统计数据，也可以是基于内存的非永久性统计数据。可以自动重新计算统计数据，也可以手动调用ANALYZE TABLE语句来更新统计信息。
2. 多表连接原理：嵌套循环连接算法。
   1. 选取驱动表，使用与驱动表相关的过滤条件，选取代价最低的单表访问方法来执行对驱动表的单表查询；
   2. 对步骤1中查询驱动表得到的结果集中的每一条记录，都分别到被驱动表中查找匹配的记录。
   由于被驱动表可能会访问多次，因此可以为被驱动表建立合适的索引以加快查询速度。如果被驱动表非常大，多次访问被驱动表可能导致很多次的磁盘I/O，此时可以使用基于块的嵌套循环连接算法来缓解由此造成的性能损耗。

   对于内连接来说，为了生成成本最低的执行计划，需要考虑两方面的事情：
   - 选择最优的表连接顺序；
   - 为驱动表和被驱动表选择成本最低的访问方法。
3. 查询优化
   1. 条件简化：移除不必要的括号、常量传递、移除没用的条件、表达式计算、HAVING子句和WHERE子句的合并、常量表检测。
   2. 外连接消除：在外连接查询中，指定的WHERE子句中包含被驱动表中的列不为NULL值的条件称为空值拒绝（reject-NULL）。在被驱动表的WHERE子句符合空值拒绝的条件时，外连接和内连接可以相互转换。
   3. 子查询优化
      1. 子查询分类：
         - 按照子查询返回的结果集分类：标量子查询、行子查询、列子查询、表子查询
         - 按照与外层查询的关系来分类：不相关子查询、相关子查询
      2. 优化IN子查询
         1. 如果IN子查询符合转换为半连接的条件，查询优化器会优先把该子查询转换为半连接，然后从下面5种执行半连接查询的策略中选择成本最低的来执行子查询：
            - Table pullout（子查询中的表上拉）：当子查询的查询列表处只有主键或者唯一索引列时，可以直接把子查询中的表上拉到外层查询的FROM子句中，并把子查询中的搜索条件合并到外层查询的搜索条件中。
            - Duplicate Weedout（重复值消除）：建立一个临时表，每当某条记录要加入结果集时，都先尝试把这条记录的id值加入到这个临时表中。
            - LooseScan（松散扫描）：扫描索引时，只取键值相同的第一条记录去执行匹配操作。
            - Semi-join Materialization（半连接物化）：先把外层查询的IN子句中的不相关子查询进行物化，然后再将外层查询的表与物化表进行连接。
            - FirstMatch（首次匹配）：先取一条外层查询中的记录，然后到子查询的表中寻找一条符合匹配条件的记录。

            半连接的适用条件：
            - 该子查询必须是与IN操作符组成的布尔表达式，并且在外层查询的WHERE或者ON子句中出现；
            - 外层查询的其他的搜索条件必须使用AND操作符与IN子查询的搜索条件连接起来；
            - 该子查询必须是一个单一的查询，不能是由UNION连接起来的若干个查询；
            - 该子查询不能包含GROUP BY、HAVING语句或者聚集函数。
         2. 如果IN子查询不符合转换为半连接的条件，查询优化器会从下面的两种策略中找出一种成本更低的方式执行子查询：
            - 先将子查询物化，再执行查询；
            - 执行IN到EXISTS的转换。
         3. MySQL在处理带有派生表的语句时，优先尝试把派生表和外层查询进行合并；如果不行，再把派生表物化掉，然后执行查询。
   4. 通过EXPLAIN语句可以查看某个语句的执行计划，紧接着还可以使用SHOW WARNINGS语句查看与这个查询的执行计划有关的扩展信息。optimizer trace功能可以查看优化器生成执行计划的整个过程。

## 五、事务

### 1. 特性

InnoDB存储引擎中的事务完全符合ACID的特性：

- 原子性（atomicity）：事务是不可分割的工作单位。
- 一致性（consistency）：事务开始前和提交后，数据库都处于一致的状态。
- 隔离性（isolation）：事务提交前对其他事务都不可见。
- 持久性（durability）：事务提交后，其结果就是永久性的。

### 2. 实现

事务的隔离性由锁来实现，原子性、一致性、持久性通过InnoDB存储引擎的redo log和undo log来实现。redo log称为重做日志，用来保证事务的持久性，undo log称为撤销日志，用来保证事务的原子性。

1. redo log
   1. 定义：记录事务执行过程中的修改内容的日志。事务提交时只将执行过程中产生的redo日志刷新到磁盘，而不是将所有修改过的页面都刷新到磁盘。这样做有两个好处：redo日志占用的空间非常小；redo日志是顺序写入磁盘的。
   2. 结构
      ![redo日志通用结构](https://cdn.jsdelivr.net/gh/n20u/PicBed/blogs/knowledge_combing/20220624090717.png)
      - type：这条redo日志的类型。
      - space ID：表空间ID。
      - page number：页号。
      - data：这条redo日志的具体内容。
   3. Mini-Transaction：MySQL对底层页面进行一次原子访问的过程为一个Mini-Transaction（MTR）。一个MTR产生的redo日志为一组，在进行崩溃恢复时，需要把这一组redo日志作为一个不可分割的整体来处理。
   4. 存储
      1. redo log block：为了更好地管理redo日志，InnoDB把redo日志都放在大小为512字节的页中，也即redo log block。
         ![redo log block的示意图](https://cdn.jsdelivr.net/gh/n20u/PicBed/blogs/knowledge_combing/20220624092159.png)
      2. redo日志缓冲区：因为磁盘速度过慢，MySQL启动时向操作系统申请了一大片称为redo log buffer（redo日志缓冲区）的连续内存空间，也可以将其简称为log buffer，这片内存空间被划分成若干个连续的redo log block。
         每个MTR运行过程中产生的日志先暂时存到一个地方，当该MTR结束时，再将过程中产生的一组redo日志全部复制到log buffer中。·
      3. redo日志文件
         1. redo日志刷盘时机：log buffer空间不足时、事务提交时、后台线程大约以每秒一次的频率将log buffer中的redo日志刷新到磁盘、正常关闭服务器时、做checkpoint时。
         2. redo日志文件组：磁盘上的redo日志文件不止一个，而是以一个日志文件组的形式出现，采用循环的方式将redo日志写入日志文件组。
         3. redo日志文件格式：redo日志文件由若干个redo log block组成，前4个block用来存储一些管理信息。
            ![redo日志文件log file header的结构](https://cdn.jsdelivr.net/gh/n20u/PicBed/blogs/knowledge_combing/20220624093852.png)
            ![redo日志文件checkpoint的结构](https://cdn.jsdelivr.net/gh/n20u/PicBed/blogs/knowledge_combing/20220624093901.png)
      4. log sequence number（lsn）
         1. 定义：lsn表示写入内存的redo日志量，flushed_to_disk_lsn表示刷新到磁盘中的redo日志量。
         2. lsn值和redo日志文件组中的偏移量的对应关系
            ![lsn值和redo日志文件组中的偏移量的对应关系](https://cdn.jsdelivr.net/gh/n20u/PicBed/blogs/knowledge_combing/20220624094500.png)
         3. flush链表中的lsn
            第一次修改某个Buffer Pool中的页面时，会把这个页面对应的控制块插入到flush链表的头部，之后再修改该页面时，不会再次插入。在这个过程中，会在缓冲页对应的控制块中记录两个关于页面何时被修改的属性：
            - oldest_modification：第一次修改该页面的MTR开始时对应的lsn值。
            - newest_modification：每次修改该页面的MTR结束时对应的lsn值。
      5. checkpoint
         执行一次checkpoint可以分为两个步骤：
         1. 计算当前系统中可以被覆盖的redo日志对应的lsn值最大是多少。redo日志可以被覆盖，意味着它对应的脏页被刷新到了磁盘中。flush链表的尾节点就是当前系统中最早修改的脏页，它的oldest_modification就是当前系统中可以被覆盖的redo日志对应的lsn值的最大值。把该脏页的oldest_modification赋值给checkpoint_lsn。
         2. 将checkpoint_lsn与对应的redo日志文件组偏移量以及此次checkpoint的编号写到日志文件组中第一个日志文件的管理信息中。
      6. 崩溃恢复
         1. 确定恢复的起点：选取checkpoint_no更大的checkpoint信息，从中得到对应的checkpoint_lsn值以及它在redo日志文件组中的偏移量checkpoint_offset。
         2. 确定恢复的终点：普通block的log block header部分有一个名为LOG_BLOCK_HDR_DATA_LEN的属性，记录了当前block中使用了多少字节的空间。因崩溃而恢复系统时，只需要从checkpoint_lsn在日志文件组中对应的偏移量开始，一直扫描redo日志文件的block，知道某个block的LOG_BLOCK_HDR_DATA_LEN值不等于512为止。
         3. 在恢复过程中，使用哈希表可以加快恢复过程，并且会跳过已经刷新到磁盘中的页面。
2. undo log
   1. 定义：记录回滚一个操作所需的必要内容的日志。trx_id隐藏列是对这个聚簇索引记录进行改动的语句所在的事务对应的事务id，roll_pointer隐藏列是一个指向记录对应的undo日志的指针。
   2. 格式：前两个字节next记录本undo log的结束位置，尾部的两个字节start记录本undo log的开始位置，还有undo log的类型，表ID，主键各列信息，以及回滚不同操作所需的必要内容。
      - INSERT操作对应的undo日志：类型为TRX_UNDO_INSERT_REC，包含插入记录的主键的每个列占用的存储空间大小和真实值。
      - DELETE操作对应的undo日志
         使用DELETE语句删除记录的过程需要经历两个阶段：
         1. delete mark阶段：仅仅将记录的deleted_flag标识位设置为1，其他的不做修改。
            ![delete mark过程示意图](https://cdn.jsdelivr.net/gh/n20u/PicBed/blogs/knowledge_combing/20220624122625.png)
         2. purge阶段：当该删除语句所在的事务提交之后，会有专门的线程来真正地把记录删除掉，也就是把该记录从正常记录链表中移除，并且加入到垃圾链表中。
            ![purge过程示意图](https://cdn.jsdelivr.net/gh/n20u/PicBed/blogs/knowledge_combing/20220624122707.png)
         在删除语句所在的事务提交之前，只会经历delete mark阶段。而一旦事务提交，也就不需要再回滚这个事务了。所以在设计undo日志时，只需要考虑对删除操作在delete mark阶段所做的影响进行回滚就好了。DELETE操作对应的undo日志类型为TRX_UNDO_DEL_MARK_REC，包括旧记录的trx_id值和roll_pointer值，主键各列信息和索引列各列信息。可以通过undo日志的roll_pointer属性找到上一次对该记录进行改动时产生的undo日志，形成一个版本链。
      - UPDATE操作对应的undo日志
         1. 不更新主键
            - 就地更新（in-place update）：在更新记录时，对于被更新的每个列来说，如果更新后的列与更新前的列占用的存储空间一样大，那么可以进行就地更新。
            - 先删除旧记录，再插入新记录：在不更新主键的情况下，如果有任何一个被更新的列在更新前和更新后占用的存储空间大小不一致，那么就需要先把这条旧记录从聚簇索引页面中真正地删除掉，然后再根据更新后列的值创建一条新的记录并插入到页面中。
            针对UPDATE操作不更新主键的情况，undo日志的类型为TRX_UNDO_UPD_EXIST_REC，如果更新的列包含索引列，那么undo日志会添加“索引列各列信息”这个部分，否则不会添加这个部分。
         2. 更新主键
            针对UPDATE语句中更新了记录主键值的这种情况，InnoDB在聚簇索引中分了两步进行处理：
            1. 将旧记录进行delete mark操作，记录一条类型为TRX_UNDO_DEL_MARK_REC的undo日志；
            2. 根据更新后各列的值创建一条新记录，并将其插入到聚簇索引中，记录一条类型为TRX_UNDO_INSERT_REC的undo日志。
   3. Undo页面：类型为FIL_PAGE_UNDO_LOG的页面专门用来存储undo日志。
      ![FIL_PAGE_UNDO_LOG类型的页面的通用结构](https://cdn.jsdelivr.net/gh/n20u/PicBed/blogs/knowledge_combing/20220624131523.png)
      1. 在InnoDB存储引擎中，undo log分为两大类，不同大类的undo日志不能混着存储：
         - TRX_UNDO_INSERT：包括TRX_UNDO_INSERT_REC类型的undo日志。简称为insert undo日志。因为insert操作的记录只对本事务可见，所以该undo log可以在事务提交后直接删除，不需要进行purge操作。
         - TRX_UNDO_UPDATE：包括TRX_UNDO_DEL_MARK_REC、TRX_UNDO_UPD_EXIST_REC类型的日志。该undo log可能需要提供MVCC机制，因此不能在事务提交后就删除，需要放入undo log链表中，等待purge线程进行删除。
      2. 一个事务产生的多条undo日志分为两个大类，每个大类的undo日志可能占用多个Undo页面，同一个大类的多个Undo页面形成一个链表。为了尽可能提高undo日志的写入效率，不同事务执行过程中产生的undo日志需要写入不同的Undo页面链表中。
      ![Undo页面链表示意图](https://cdn.jsdelivr.net/gh/n20u/PicBed/blogs/knowledge_combing/20220624132326.png)
      3. 重用Undo页面。一个Undo页面链表如果可以被重用，那么它需要符合如下条件：该链表中只包含一个Undo页面且该Undo页面已经使用的空间小于整个页面空间的3/4。
   4. 回滚段
      1. 定义：为了更好地管理Undo页面链表，InnoDB的名为Rollback Segment Header的页面中，存放了各个Undo页面链表的first undo page的页号，这些页号称为undo slot。一个Rollback Segment Header页面中有1024个undo slot。每一个Rollback Segment Header页面都对应着一个段，这个段就称为回滚段（Rollback Segment）。在系统表空间的第5号页面中存储了128个Rollback Segment Header页面地址。
      2. 从回滚段中申请undo slot
         - 当一个事务提交时，它所占用的undo slot有两种“命运”。
           - 如果该undo slot指向的Undo页面链表符合被重用的条件，该undo slot就处于被缓存的状态。insert undo页面链表的undo slot会被加入insert undo cached链表中，update undo页面链表的undo slot会被加入update undo cached链表中。一个回滚段对应着上述两个cached链表。
           - 如果该undo slot指向的Undo页面链表不符合被重用的条件，insert undo页面链表对应的段会被释放掉，update undo页面链表中本次事务写入的一组undo日志会被放到History链表中，然后把该undo slot的值设置为FIL_NULL。
         - 当一个新事务要分配undo slot，都先从对应的cached链表中找，如果没有被缓存的undo slot，再到回滚段的Rollback Segment Header页面中寻找第一个值为FIL_NULL的undo slot。
      3. 为事务分配Undo页面链表的详细过程：
         1. 事务在执行过程中对普通表的记录进行首次改动之前，首先会到系统表空间的第5号页面中使用round-robin（循环使用）分配一个回滚段；
         2. 在分配到回滚段后，从回滚段中申请undo slot；
         3. 找到可用的undo slot后，如果该undo slot是从cached链表中获取的，那么它对应的Undo Log Segment就已经分配了；否则需要重新分配一个Undo Log Segment，然后从该Undo Log Segment中申请一个页面作为Undo页面链表的first undo page，并把该页的页号填入获取的undo slot中；
         4. 然后事务就可以把undo日志写入到上面申请的Undo页面链表中了。
   5. undo日志在崩溃恢复时的作用：找到那些值不为FIL_NULL的undo slot，每一个undo slot对应着一个Undo页面链表。然后从Undo页面链表第一个页面的Undo Segment Header中找到TRX_UNDO_STATE属性，如果该属性的值为TRX_UNDO_ACTIVE，则意味着有一个活跃的事务正在向这个Undo页面链表中写入undo日志。然后再在Undo Segment Header中找到TRX_UNDO_LAST_LOG属性，通过该属性可以找到本Undo页面链表最后一个Undo Log Header的位置。从该Undo Log Header中可以找到对应事务的事务id以及一些其他信息，则该事务id对应的事务就是未提交的事务。通过undo日志中记录的信息将该事务对页面所做的更改全部回滚掉，这样就保证了事务的原子性。

## 六、多版本并发控制（Multi Version Concurrency Control，MVCC）

1. 一致性问题
   - 脏读：一个事务读到了另一个未提交事务修改过的数据。
   - 不可重复读：一个事务修改了另一个未提交事务读取的数据。
   - 幻读：一个事务先根据某些搜索条件查询出一些记录，在该事务未提交时，另一个事务写入了一些符合那些搜索条件的记录。

   SQL标准中规定：针对不同的隔离级别，并发事务执行过程中可以发生不同的现象：

   |    隔离级别    | 脏读 |不可重复读| 幻读 |
   |:-------------:|:----:|:-------:|:----:|
   |READ UNCOMMITTED（未提交读）|可能|可能|可能|
   |READ COMMITTED（已提交读）|不可能|可能|可能|
   |REPEATABLE READ（可重复读）|不可能|不可能|可能|
   |SERIALIZABLE（可串行化）|不可能|不可能|不可能|
   MySQL在REPEATABLE READ隔离级别下，可以很大程度上禁止幻读现象的发生。
2. MVCC原理
   1. 版本链：聚簇索引记录中包含两个必要的隐藏列：trx_id表示上一次修改该记录的事务id，roll_pointer表示上一次修改该记录时产生的undo日志的指针。每次更新该记录后，都会将旧值放到一条undo日志中。每条undo日志也都有一个roll_pointer属性，通过这个属性可以将这些undo日志串成一个链表，这个链表称为版本链。另外，每个版本中还包含生成该版本时对应的事务id。
   2. ReadView（一致性视图）：用于判断版本链中的哪个版本是当前事务可见的。
      ReadView中主要包含4个比较重要的内容：
         - m_ids：在生成ReadView时，当前系统中活跃的读写事务的事务id列表。
         - min_trx_id：在生成ReadView时，当前系统中活跃的读写事务中最小的事务id。
         - max_trx_id：在生成ReadView时，系统应该分配给下一个事务的事务id值。
         - creator_trx_id：生成该ReadView的事务的事务id。

      有了这个ReadView后，在访问某条记录时，只需要按照下面的步骤来判断记录的某个版本是否可见：
      1. 如果被访问版本的trx_id属性值与ReadView中的creator_trx_id值相同，意味着当前事务在访问它自己修改过的记录，所以该版本可以被当前事务访问；
      2. 如果被访问版本的trx_id属性值小于ReadView中的min_trx_id值，表明生成该版本的事务在当前事务生成ReadView前已经提交，所以该版本可以被当前事务访问；
      3. 如果被访问版本的trx_id属性值大于或等于ReadView中的max_trx_id值，表明生成该版本的事务在当前事务生成ReadView后才开启，所以该版本不可以被当前事务访问；
      4. 如果被访问版本的trx_id属性值在ReadView的min_trx_id和max_trx_id之间，则需要判断trx_id属性值是否在m_ids列表中。如果在，说明创建ReadView时生成该版本的事务还是活跃的，该版本不可以被访问；如果不在，说明创建ReadView时生成该版本的事务已经被提交，该版本可以被访问。

      在MySQL中，READ COMMITTED与REPEATABLE READ隔离级别之间一个非常大的区别就是它们生成ReadView的时机不同：READ COMMITTED在每次读取数据前都生成一个ReadView，REPEATABLE READ在第一次读取数据时生成一个ReadView。
   3. 二级索引与MVCC
      只有聚簇索引记录中才有trx_id和roll_pointer隐藏列，如果某个查询语句是使用辅助索引来执行查询，则按照下面两步来判断可见性：
      1. 辅助索引页面的Page Header部分的PAGE_MAX_TRX_ID属性值记录了修改该辅助索引页面的最大事务id。当查询某个辅助索引记录时，首先会看一下对应的ReadView的min_trx_id是否大于该页面的PAGE_MAX_TRX_ID。如果是，说明该页面中的所有记录都对该ReadView可见；否则就得回表之后再判断可见性；
      2. 利用辅助索引记录中的主键值进行回表操作，得到对应的聚簇索引记录后再按照前面讲过的方式找到对该ReadView可见的第一个版本，然后判断该版本中相应的辅助索引列的值是否与利用该辅助索引查询时的值相同。如果是，就把这条记录发送给客户端，否则就跳过该记录。
3. purge
   1. 定义：为了节约存储空间，应该在合适的时候把update undo日志以及仅仅被标记为删除的记录彻底删除掉，这个删除操作就称为purge。
   2. 操作：
      1. 在一个事务提交时，会为这个事务生成一个名为事务no的值，该值用来表示事务提交的顺序；
      2. 在生成一个ReadView时，会包含一个事务no的属性，表示比当前系统中最大的事务no值还大1的值；
      3. InnoDB把当前系统中所有的ReadView按照创建时间连成了一个链表。当执行purge操作时，就把系统中最早生成的ReadView取出来。然后从各个回滚段的History链表中取出事务no值小于当前系统最早生成的ReadView的事务no属性值的各组undo日志，将它们从History链表中移除，并且释放掉它们占用的存储空间。如果该组undo日志中包含因delete mark操作而产生的undo日志，那么也需要将对应的标记为删除的记录给彻底删除。

## 七、锁

1. 锁问题
   InnoDB存储引擎中有两种读取数据的方式：一致性无锁读和锁定读，也称快照读和当前读。对于READ COMMITTED和REPEATABLE READ事务隔离级别下的一致性无锁读，使用多版本并发控制（Multi Version Concurrency Control，MVCC）解决一致性问题。对于其他读取数据的方式，使用锁解决一致性问题。
   - 共享锁（Shared Lock）：简称S锁。在事务要读取一条记录时，需要先获取该记录的S锁。
   - 独占锁（Exclusive Lock）：简称X锁。在事务要改动一条记录时，需要先获取该记录的X锁。
2. InnoDB的行锁和表锁
   1. InnoDB中的表级锁
      - 表级别的S锁、X锁：只会在一些特殊情况下（比如在系统崩溃恢复时）用到。
      - 表级别的IS锁、IX锁：为了在之后加表级别的S锁和X锁时，可以快速判断表中的记录是否被上锁，以避免用遍历的方式来查看表中有没有上锁的记录。
      - 表级别的AUTO-INC锁：用于系统自动给AUTO_INCREMENT修饰的列进行递增赋值。
   2. InnoDB中的行级锁
      - Record Lock：只对记录本身加锁。
      - Gap Lock：锁住记录前的间隙，防止别的事务向该间隙插入新记录。
      - Next-Key Lock：Record Lock和Gap Lock的结合体，既保护记录本事，也防止别的事务向该间隙插入新记录。
      - Insert Intention Lock：事务在插入一条记录时，如果插入位置已被别的事务加了gap锁，就需要等待并在内存中生成一个锁结构。
      - 隐式锁：依靠记录的trx_id属性来保护不被别的事务改动该记录。
3. InnoDB锁的内存结构
   InnoDB在对不同记录加锁时，如果符合下面这些条件，这些记录的锁就可以放到一个锁结构中：
   - 在同一个事务中进行加锁操作；
   - 被加锁的记录在同一个页面中；
   - 加锁的类型是一样的；
   - 等待状态是一样的。

   ![InnoDB存储引擎事务锁结构](https://cdn.jsdelivr.net/gh/n20u/PicBed/blogs/knowledge_combing/20220624195241.png)

## 八、主从复制（replication）

主从复制的工作原理分为以下3个步骤：

1. 主服务器（master）把数据更改记录到二进制日志（binlog）中；
2. 从服务器（slave）的I/O线程负责把主服务器的二进制日志复制到自己的中继日志（relay log）中；
3. 从服务器的SQL线程负责重做中继日志，把更改应用到自己的数据库上，以达到数据的最终一致性。

## 九、二进制日志

二进制日志有3种格式：

1. 语句（STATEMENT）格式：二进制日志记录的是对数据库执行更改的逻辑SQL语句。基于此格式的二进制日志在主从复制时，有可能会出现主从数据库上的数据不一致，比如在主服务器运行RAND()、UUID()等不确定函数时，或者在主服务器的InnoDB存储引擎使用READ COMMITTED的事务隔离级别时，会出现丢失更新的现象；
2. 行（ROW）格式：二进制日志记录的是表的行更改情况。基于此格式的二进制日志可以为数据库的恢复和复制带来更好的可靠性，在主从复制时，也可以将主服务器的InnoDB存储引擎的事务隔离级别设为READ COMMITTED，以获得更好的并发性。但有些语句下的行格式需要更多的容量，这使得二进制日志体积更大，因此主从复制的网络开销也有所增加；
3. 混合（MIXED）模式：当存储引擎对二进制日志格式只支持语句格式和行格式中的一种，则采用其支持的那种格式。当两种格式都支持时，MySQL默认采用语句格式，但是在一些情况下会采用行格式，比如主服务器使用了不确定函数，使用了用户定义函数（UDF）或临时表。从而同时获得了语句格式的空间优势和行格式的可靠性优势。

因为备份及恢复的需要，需要保证MySQL数据库上层的二进制日志的写入顺序和InnoDB存储引擎层的事务提交顺序一致，否则会导致事务数据的丢失。为了保证这两种日志之间的一致性，同时还要利用group commit提高数据库的整体性能，MySQL5.6采用了Binary Log Group Commit（BLGC），其实现方式为：
将MySQL数据库上层提交的事务按顺序放入一个队列中，队列中的第一个事务（leader）控制着其他事务（follower）的行为。BLGC分为以下三个阶段：

1. Flush阶段：将每个事务的二进制日志写入内存中；
2. Sync阶段：将内存中的二进制日志刷新到磁盘，仅一次fsync操作就完成了队列中所有事务的二进制日志的写入，这就是BLGC；
3. Commit阶段：leader根据顺序调用存储引擎层事务的提交，此处能够利用InnoDB存储引擎的group commit特性。

当一组事务在进行Commit阶段时，其他新事务可以进行Flush阶段，从而使group commit不断生效。

## 十、常见问题

1. 查询语句的执行步骤
