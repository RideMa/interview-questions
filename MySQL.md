## MySQL
### MySQL执行流程
* 连接器
* * 连接MySQL服务，通过TCP连接进行
* * 连接器会获取用户的权限等，可以使用show processlist看有多少个连接
* * 空闲的连接超过8小时就会自动断开，也可以用kill connection 来断开
* * 也有长连接和短连接之分，区别就是是否执行一个SQL就断开
* 缓存
* * 连接器发来的SQL会先在缓存中查找是否存在
* * 由于缓存不是很好用，MySQL8.0之后就删掉了
* 解析器
* * 词法分析语法分析，判断SQL是否正确
* * 这里不查询是否表或字段存在
* 预处理器
* * 查询表或者字符是否存在
* * 把*扩展成所有列
* 优化器
* * 负责确定执行方案
* * 优化器选择索引、选择是否使用索引、是否全表扫描、是否回表等
* 执行器
* * 执行语句，和存储引擎交互
* * 主键索引查询：调用read_first_record函数指针，把条件传给InnoDB，不断循环
* * 全表扫描：还是那个函数，读取记录，条件由执行器来判断
* * 索引下推：如果联合索引失效了，那么需要回表查询，然后由执行器判断条件，但是判断条件的列其实也是联合索引的一部分，就可以使用索引下推，就不需要执行器来判断，交给存储引擎来做（可以减少回表的次数）
* * * 索引下推的时候，存储引擎判断条件不符合就不回表了
* * * 判断条件的列使用了函数就没办法下推了
* * * 有子查询也没办法下推
### MySQL行存储结构
* MySQL在Linux里存在/var/lib/mysql底下
* * 一个库就有一个文件夹，一个table就有三个文件
* * 一个是db.opt是存储默认字符集和字符校验规则
* * 一个是.frm保存表的元数据
* * 一个是.ibd是表的数据
* 表空间文件的结构
* * 表->段->区->页->行
* * 记录按行存储
* * 页是I/O的最小单元，默认一个页大小是16kb，B+树的结点就是页
* * 为了让B+树的节点之间物理存储位置连续一下，就有区，一个区1mb，64个页
* * 段是多个区，分为数据段、索引段、回滚段
* 行格式
* * 分为Redundant、Compated、Dynamic和Compressed
* * Compact多用，其有记录额外信息和记录真实信息两个部分组成
* * 记录额外信息包括三个部分：变长字段长度列表、NULL值列表、记录头信息
* * * 变长字段长度列表是对varchar的长度进行记录，在行中会倒序存放，每个长度两个字节或一个字节（倒序是为了长度和靠前的数据在同一个CPU cache里）
* * * NULL长度列表是一个Bitmap，也是倒序的，每8个可能有空就多一个字节
* * * 记录头信息，记录是否被删除，下一条记录的指针，记录类型（叶子结点、最小记录、最大记录、非叶子结点）
* * 记录真实信息
* * * 三个隐藏字段，roll_id，trx_id，roll_ptr
* * * 一个是自增主键，如果建表的时候指定了就没这个了，6 bytes
* * * trx_id是修改这一行的事务的id，6 bytes
* * * roll_ptr是mvcc的上一个版本的指针，7 bytes
* * varchar(n) n最大是多少
* * * 一行记录加起来最大65535
* * * 如果单一字段，加上可变长度是2 bytes，加上NULL的bitmap 1 byte，可以存65532
* * * 多个字段要看可变长度，和NULL
* * 行溢出
* * * 一页16374b，比如说TEXT和BLOB都可能溢出，就需要存在另外的溢出页
* * * compacted是这一行保存一部分，其他用指针指过去，Dynamic和Compressed是直接指向溢出页地址
### MySQL索引
* InnoDB默认B+树索引，还支持full-text索引，MyISAM也是
* B+树索引
* * 节点里多个数据，数据按照主键顺序存放，B+树叶子结点是单向链表（MySQL是双向）
* * 查询的时候通过一个节点的主键值范围查找到对应的叶子结点
* * B+树的优势，层数少，磁盘I/O次数少，并且比B树更适合范围查找（叶子结点双向链表）
* * B+树一层通过设置超过1000个节点，所以在2000w行以内基本都是3-4层，比HASH也更是和范围查询
* * B+树二级索引是再建立一个B+树，其叶子结点存储的是索引的值和主键ID，查询如果不止这两个列的话，就要回表，即查到了主键ID再回聚簇索引里找
* * 不用回聚簇索引里找就叫索引覆盖
* * 主键默认是第一个UNIQUE NOT NULL
* * 没有就自增ID
* * 前缀索引：用字符类型字段的前几个字符建立的索引
* 联合索引
* * 即多个字段联合建立的索引
* * 联合索引的最左匹配原则
* * 因为联合索引建立的时候是按照顺序的，第一个全局有序，第二个在第一个一样的情况下有序
* * 所以如果不按顺序进行查询，就是无序的查询，没法利用联合索引，就会索引失效
* * 而如果第一个使用了范围查询，后面可能就没法用联合索引了
* * * 如果第一个使用了小于等于或大于等于，后面的在等于的情况其实是可以用的，不会从等于的第一个开始遍历，而是等于且满足第二个条件的那个节点开始遍历
* * * 使用Between and也是类似于有等于的情况
* * * like j%其实也是可以用的，就是会从满足第二个的首个节点开始遍历
* * * 如果第一个使用了大于或小于，以及其他范围查询，后面的就不能用了
* * * 使用左或者左右模糊匹配就啥索引都用不了like %xx%或者%xx
* 索引区分度
* * 建立联合索引的时候要把区分度大的放前面
* * $区分度=\frac{distinct(column)}{count(*)}$如果区分度小，比如说男女，比例达到30%以上，MySQL就会直接忽略索引，全表扫描
* 建立索引问题
* * 占用空间大
* * 维护代价高
* * 创建速度慢
* 适用与不适用建立索引的情况
* * 适用
* * 唯一性字段
* * 经常使用where比较的字段
* * 经常使用Group by和Order by的字段
* * 不适用
* * 区分度小
* * where group by order by用不着的字段
* * 表的数据少的时候
* * 经常更新的字段
* 索引优化
* * 前缀索引优化
* * * 减少空间，一个页更多索引，提高速度
* * * order by没法用，没法用覆盖索引
* * 覆盖索引优化
* * * 覆盖了就不用回表了
* * 主键最好是自增的
* * * 这样每次插入都是append，不需要移动数据
* * * 否则可能涉及页分裂等问题，可能造成内存碎片，影响查询效率
* * 索引最好NOT NULL
* * * NULL会使得优化器不好处理
* * * NULL还要占据额外的物理存储空间
* explain执行计划参数
* * possible_keys，可能用的索引
* * key，字段实际用的索引
* * key_len，索引长度
* * rows，扫描的数据行数
* * type，数据扫描类型
* * * 效率排行：ALL-Index-range-ref-eq_ref-const
* * Extra，是否用了额filesort，temporary，index
* 数据页的结构
* * 文件头（双向链表，链接其他的页）
* * 页头：页的状态信息
* * 最大和最小记录：假记录，最大和最小的记录
* * 用户记录
* * 空闲空间
* * 页目录，用户记录的相对位置
* * 文件尾：校验页是否完整
* * 页目录是多个槽组成，每个槽一些页，通过二分法来查找，槽对应的是最大记录，每个槽里是一个分组的记录，没几条
* * B+树的每个节点都是数据页，非叶节点的记录为下面的页里最小记录
* B+树的优势
* * 磁盘一个扇区是512B，操作系统一次读多个扇区
* * Linux一个页是4KB，一次8个扇区（不算预读）
* * MySQL一次32个扇区，16KB
* * 二叉树可能退化为O(n)的复杂度，平衡二叉树维护代价太高
* * 红黑树层数还是太高，对于数据库来说维护也复杂了
* * B树的单点查询是比B+树有一点点优势的
* * 但是B树的范围查询需要中序遍历，要花费更多的I/O次数
* * B+树的叶节点是连起来的，范围查询更快
* * B+树插入和删除的效率更高一些
* 2000w行效率下降的来源
* * 一个页是16K，在非叶节点中，主键加页号12 bytes的话，大概有1280行
* * 三层就能大概就是500w左右条记录
* * MySQL会把索引加载到内存，但是太多了加载不进来就会很慢，而且层数增加了就更慢了
* 索引失效
* * 使用左或者左右模糊匹配，就啥索引都用不了like %xx%或者%xx
* * 对索引使用函数就没用了比如说：where length(name)=6
* * 表达式计算也是一样的，也是函数
* * MySQL会自动把字符串转换成整形，如果你是一个字符串索引，这时候你输入了一个整形，那么MySQL会把字符串字段转换成整形，就会索引失效
* * 最左匹配失败
* * where or前后不都是索引列，只要一个不是索引列就没用了
* * 如果一个表所有的列都在二级索引里，那么有特殊情况
* * 查询可以直接在二级索引里进行，就算索引失效也会会直接覆盖索引
* * 这是因为聚簇索引里面有更多的信息，比如MVCC等，所以直接扫二级索引的树代价更低
* count性能
* * count（*）和count（0）是一样的，和count(1)也是一样的
* * count(主键字段)在没有二级索引的时候，走聚簇索引，有二级索引走二级索引
* * count(1)走的是聚簇索引，但是不看里面有啥，所以最快
* * count(普通字段)全表扫描
* * 使用遍历的方式count是因为由于MVCC，返回多少行不太确定，不能靠维护一个count来进行
* * 优化count可以使用近似值或者自己维护一个count
### MySQL事务
* ACID：原子性、一致性、隔离性、持久性
* * 原子性：一个事务中的操作要么全做要么不做
* * 一致性：事务前后数据满足完整性约束，数据库保持一致状态
* * 隔离性：多个事务执行中发生的数据不一致，事物之间不影响
* * 持久性：事务结束后对于数据的处理是持久的，断电也保存
* 并行事务带来的问题
* * 脏读：一个事务读取到另一个未提交事务修改过的数据，比如A读取到B改到一半的数据
* * 不可重复读：在一个事务中，前后两次读取到的数据不一样
* * 幻读：在一个事务中，前后两次查询到的记录数量不一样
* MySQL隔离级别
* * 读未提交：会有脏读、不可重复读、幻读
* * 读提交：解决脏读
* * 可重复读：解决不可重复读
* * 串行化：解决幻读
* * 默认是可重复读
* * MySQL在可重复读的级别能很大程度解决幻读
* * * 针对快照读，MySQL用MVCC在可重复读级别下解决了幻读
* * * 针对当前读，MySQL使用next-key lock解决了幻读
* * * 那是还是有幻读，比如先快照读，再当前读，中间如果有修改就会幻读
* * 串行化使用的是读写锁
* * 读提交是每个SQL都生成一个当前的readview
* * 可重复度是一个事务使用一个readview
* Read View
* * 四个字段：
* * creator_id：创建readview的事务id
* * m_ids：当前活跃且未提交的事务id
* * min_tx_id：创建readview的时候活跃且未提交的最小事务id
* * max_tx_id：创建readview的时候数据库应该给下一个事务的id
* * 如果记录的tx_id小于min_tx_id，就说明事务已经提交了，可以读
* * 如果记录的tx_id大于等于max_tx_id，则不可见，去看以前的版本
* * 如果记录的tx_id在m_ids里，则不可见，去看以前的版本
* * 否则可见，说明已经提交了
* * 由于行的隐藏列有一个指向以前版本的指针，所以是可以找到以前的版本的
* 当前读会加next-key lock，符合当前读范围的行都会加上锁，修改会被阻塞，
* 幻读的场景
* * 如果A快照读一个没有的行，然后B插入了这个行，然后A去修改这个行，那么接下来A就能找到这个行了
* * 如果A快照读了一下，然后B插入了，然后A当前读，就幻读了
### MySQL锁
* 全局锁
* * flush tables with read lock
* * 改表结构和数据增删改都阻塞
* * 全库逻辑备份的时候使用全局锁，防止不一致
* * 可重复度隔离级别下，先开启事务就可以逻辑备份不用阻塞（创建了readview）
* 表锁
* * lock tables name read （S型锁）
* * lock tables name write （X型锁）
* 元数据锁
* * 对表进行CRUD的时候会给元数据加读锁
* * 结构变更的时候会加写锁
* * MDL是在事务提交的时候才释放，任务执行期间是一直持有的，长事务会阻塞改变元数据的事务（写锁优先级高，后面的读锁都没法拿到锁了）
* 意向锁
* * 加行级锁的时候要加意向锁，意向共享锁、意向独占锁
* * 意向锁是为了加独占锁（表级锁）的时候更快发现是否有行级锁
* * 意向锁之间不会冲突
* AUTO-INC锁
* * 主键自增会用这个锁，插入一个数据的时候加锁，插完就解锁，不等事务结束
* * 轻量级自增的锁：给字段自增的值就释放了，不需要整个语句结束
* * 使用轻量级锁加binlog主从复制可能数据不一致
* * * 比如一个事务create table t2 like t1
* * * 然后另一个事务插入t2，这个插入语句在啥时候执行不一定
* * * 如果用binlog的话，也有可能不知道顺序
* * * 如果binlog用的是statement就不行，换成row就可以了
* Record lock
* * 记录锁，把一条记录锁上，S型和X型
* Gap Lock
* * 间隙锁，锁的是大的那个记录和小的记录之间的间隙，其实是记在大的那个记录上
* * 间隙锁之间是可以兼容的，虽然有S和X之分但是实际没区别
* Next-key Lock
* * Record和Gap lock结合，锁住一个左开右闭的区间
* * 这个S和X型锁是有互斥关系的
* * 这个是为了解决当前读的幻读问题
* 插入意向锁
* * 插入意向锁是插入有间隙锁的位置的时候加的锁
* * 插入意向锁设定为等待状态，等间隙锁释放转变为正常状态才算是加锁成功
* * 插入意向锁是特殊的间隙锁，锁住一个点，不同的位置的插入意向锁是不互斥的，但是在同一个位置的插入意向锁是互斥的
* select in share mode加S型锁
* select for update是X型锁
* 改和删都是X型锁
* 对于唯一值和非唯一值的等值查询和范围查询，最后加出来的锁是不一样的，记录是否存在也是有不同的
* * 判断的方式就是能否避免幻读
* * 给二级索引加锁也会给主键记录加记录锁（不会加间隙锁）
* * 看锁方式：select * from performance_schema.data_locks\G
* * 唯一值等值查询，如果记录存在记录锁，不存在间隙锁
* * 唯一值范围查询，如果记录存在，大于等于的那个会是记录锁，其他是next-key lock；小于退化成间隙锁，小于等于next-key lock；如果不存在都是间隙锁
* * 非唯一值等值查询，如果记录存在所有符合的加next-key lock，第一个不符合的间隙锁；如果不存在，间隙锁
* * 非唯一值间隙锁的两端有时可以插入成功有时不行
* * * 原因在那个地方的下一个记录有间隙锁才会阻塞
* * * 二级索引如果在右端点插入的比右端点主键值大，那就可以插
* * * 如果二级索引在左端点插入的比左端点主键大，但是下一个也有间隙锁，那就不行，比左端点主键小时不行的
* * 非唯一值范围查询，都加next-key lock
* * 没加索引全表扫描，每个都加next-key lock，很严重，会全表锁住！
* * 使用sql_safe_updates参数，只有where有索引，使用limit才能update成功
* MySQL死锁
* * 两个事务都持有间隙锁，并且都想插入，获取插入意向锁，就死锁了
* * Insert在正常执行的时候是不会加锁的，如果不可能冲突，InnoDB就跳过加锁阶段，使用隐式锁，可能发生冲突才加锁
* * * 具体方式是因为聚簇索引有trx_id这个字段，操作一个记录的是先看这个事务是否是活跃的，如果是活跃的，给这个事务转换为显式锁
* * * 检查是否有锁冲突，有的话创建锁，设置成waiting状态
* * 如果主键索引冲突，插入重复的主键，MySQL会报错，同时给主键索引加上一个S型记录锁，唯一二级索引冲突，会给它加上next-key lock
* * * 这样另一个事务的当前读就会阻塞
* * * 如果一个insert，先不加锁，另一个insert一样的，那么就会从隐式锁变成显式锁，第一个就会获得一个X型记录锁，和第二个的next-key lock或者记录锁冲突
* * MySQL死锁解决方案：打破四者之一：互斥、占有且等待、不可强占用、循环等待
* * 设置事务等待锁的超时时间innodb_lock_wait_timeout
* * 开启主动死锁检测，innodb_deadlock_detect，主动回滚死锁的某一个事务
### MySQL log
* undo log
* * 用于事务回滚和MVCC
* * 修改、删除、新增的undo log格式不同
* * MySQL每一行有隐藏字段trx_id和roll_pointer，这样就可以通过rool_pointer找到之前的版本（undo log）
* * undo log和readview加起来能实现MVCC，快照读的时候会根据readview找到undo log中对应readview的数据
* buffer poll
* * InnoDB请求的内存空间，有数据页、索引页、插入缓存页、自适应哈希索引、锁信息、undo页
* * 查询的时候会直接加载一页，undo log也会缓存在undo页里
* redo log
* * 用于持久化，掉电等故障恢复
* * WAL，write ahead log
* * 写操作不是立刻写到磁盘上，而是先写log，然后在合适的时候写到磁盘上
* * redo log是在XX表的XX页的XX偏移量位置做了什么操作
* * 掉电重启，可以通过redo log找到修改，重做以恢复
* * Undo log的持久化也是通过redo log实现的
* * redo log是顺序写，速度快，因为其是追加，在磁盘上是连续的
* * redo log也是有一个buffer的，不是立刻写入，大小为16M
* * redo log刷盘时机
* * * MySQL正常关闭
* * * redo log buffer的记录写入量大于内存空间的一半的时候
* * * 后台线程没1秒持久化
* * * 每次事务提交的时候
* * * innodb_flush_log_at_trx_commit 0不写入，1直接持久化，2写道pagecache里
* * 默认情况下，redo log文件有2个，是一个环状的，写第一个文件，满了写第二个，满了写第一个
* * 如果redo log文件满了，那MySQL就会阻塞，把Buffer poll中的脏页刷盘，然后就可以空出来没用的redo log
* binlog
* * 数据备份和主从复制
* * binlog是server层的，记录的更新的情况
* * 三种格式
* * * statement记录修改的SQL语句，但是可能因为并发或者动态函数导致数据不一致
* * * row 记录行数据最终被改成啥样，太大
* * * mixed 自动调整statement还是row
* 主从复制
* * 异步执行
* * 主库写入binlog，从库复制binlog，从库回放binlog
* * 从库受到binlog就会回复ack
* * 这样可以主库写、从库读
* * 如果从库太多，导致I/O太多，一般1主2从1备主
* * binlog首先写到binlog cache里，事务提交的时候再把binlog cache的内容写道binlog文件里
* * 每个线程都有binlog，但是都写道一个binlog文件里
* * sync_binlog 0只write不fsync，1每次都write直接fsync，N每次都write，N个事务fsync（100-1000）
* update执行流程
* * 执行器负责执行，调用存储引擎的接口，如果在buffer pool就直接，否则读磁盘
* * 执行器拿到以后看是否有变化，没有就不继续了
* * 开启事务，记录undo页（buffer pool中），要记录redo log
* * InnoDB更新记录，先记录redo log，后续WAL，合适机会刷盘
* * 执行完成以后，记录binlog，保存在binlog cache里，提交了再刷盘
* 两阶段提交
* * redolog和binlog要一致，先后都可能因断电而不一致，因此需要两阶段提交
* * MySQL开启一个XA事务，分两阶段执行
* * * 第一阶段，把XID写入redo log，将redo log事务状态变为prepare，然后将redolog持久化
* * * 第二阶段，把XID写入binlog，把binlog持久化到磁盘，然后调用引擎提交事务接口，把redo log状态设置为commit
* * * 只要binlog刷盘成功，这个commit有没有刷盘不重要，binlog有这个XID就算提交成功
* * MYSQL还是会以哦之redo log刷盘（1秒一次），不过没关系，如果事务中间崩溃了，MySQL会回滚，所以没有binlog也没事
* * 两阶段提交I/O次数比较高，而且锁竞争比较激烈，早期MySQL两阶段提交要一直持有锁
* * binlog组提交
* * * 多个事务提交的时候，会合并多个binlog
* * * commit阶段变为三个阶段
* * * flush阶段，多个事务按顺序写入binlog到cache（不刷盘）
* * * sync阶段，binlog文件做fsync操作，
* * * commit阶段，各个事务做InnoDB commit操作
* * * 每个阶段都有一个队列，每个阶段都有锁做保护，第一个进入队列的事务成为leader，领导队内的事务，完成后通知其他事务操作
* * redolog组提交
* * * 5.7版本之后引入
* * * 把redo log刷盘延迟到flush阶段
* * * 这样flush阶段刷redo log，写binlog
* * * sync阶段刷binlog，提交成功
* * * commit还是一样
* * * Binlog_group_commit_sync_delay binlog不是直接刷盘，要等一波一起，时间到了，或者一组满了就刷盘
* MySQL磁盘I/O很高的优化
* * 设置组提交参数，binlog_group_commit_sync_no_delay_count 和binlog_group_commit_sync_delay 延迟binlog刷盘
* * sync_binlog设置成大于1，100-1000，从而多个事务才刷binlog
* * innodb_flush_log_at_trx_commit设置为2，redolog写到pagecache里
### buffer poll
* 大小通过innodb_buffer_pool_size参数来控制，默认128M，可以扩展到60%到80%
* 数据页、索引页、插入缓存页、undo页、自适应哈希索引、锁信息
* 每个缓存也有控制块指向它
* 空闲页是利用一个链表管理，其指向每一个空闲的控制块
* 脏页是有一个链表，指向每一个脏的控制块
* 缓存替换LRU
* * 预读失效：提前加载的页没被访问，还被放在首部赶走了热点缓存
* * 解决：LRU链表分为young和old区域，预读放old
* * 缓存污染：扫描到大量数据，把buffer全给替换了
* * 解决：第一次和第二次扫描的间隔大于1s才从old提到young
* 脏页什么时候刷盘
* * redo log日志满了触发刷脏页
* * buffer poll空间不足，淘汰一部分数据页，如果是脏页就要刷盘
* * MySQL认为空闲的时候会刷盘
* * MySQL关闭的时候会刷盘