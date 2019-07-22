> 在计算机中，锁是协调多个进程或线程并发访问某一资源的一种机制。在数据库当中，数据也是一种供许多用户共享访问的资源。如何保证数据并发访问的一致性、有效性，是所有数据库必须解决的一个问题，锁的冲突也是影响数据库并发访问性能的一个重要因素。

### 贯穿始终的MDL锁
- 什么是`MDL`锁？
> `MDL`锁的全称是`Meta Data Lock`，即元数据锁。它是`MySQL`内置级别的锁，供`MySQL`预防共享资源冲突的场景。
- `MDL`锁的类型：

| 锁名称 | 简称 | 锁类型 | 说明 | 使用语句 | 
| --- | --- | --- | --- | --- |
| MDL_INTENTION_EXCLUSIVE |  | S锁 | 意向锁，锁住一个范围 | 任何语句都会获取MDL意向锁，然后再获取更强级别的MDL锁。 |
| MDL_SHARED | S | S锁 | 共享锁，表示只访问表结构 |  |
| MDL_SHARED_HIGH_PRIO | SH | S锁 | 共享锁，只访问表结构 | show create table 等<br>只访问INFORMATION_SCHEMA的语句 |
| MDL_SHARED_READ | SR | S锁 | 访问表结构并且读表数据 | select语句 <br>LOCK TABLE ...  READ |
| MDL_SHARED_WRITE | SW | S锁 |  | SELECT ... FOR UPDATE <br>  DML语句 |
| MDL_SHARED_UPGRADABLE | SU | S锁 | 可升级锁，访问表结构并且读写表数据 | Alter语句中间过程会使用 |
| MDL_SHARED_NO_WRITE | SNW | S锁 | 可升级锁，访问表结构并且读写表数据，并且禁止其它事务写。 | Alter语句中间过程会使用 |
| MDL_SHARED_NO_READ_WRITE | SNRW | S锁 | 可升级锁，访问表结构并且读写表数据，并且禁止其它事务读写。 | LOCK TABLES ... WRITE |
| MDL_EXCLUSIVE | X | X锁 | 禁止其它事务读写。 | CREATE/DROP/RENAME TABLE等DDL语句。 |
> `S`锁代表共享锁，`X`锁代表排他锁。
- `MDL`的兼容性矩阵(对象维度)

| Request type | S | SH | SR | SW | SU | SNW | SNRW | X |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| S | ✔️ | ✔️ | ✔️ | ✔️ | ✔️ | ✔️ | ✔️ | ✘ |
| SH | ✔️ | ✔️ | ✔️ | ✔️ | ✔️ | ✔️ | ✔️ | ✘ |
| SR | ✔️ | ✔️ | ✔️ | ✔️ | ✔️ | ✔️ | ✘ | ✘ |
| SW | ✔️ | ✔️ | ✔️ | ✔️ | ✔️ | ✘ | ✘ | ✘ |
| SU | ✔️ | ✔️ | ✔️ | ✔️ | ✘ | ✘ | ✘ | ✘ |
| SNW | ✔️ | ✔️ | ✔️ | ✘ | ✘ | ✘ | ✘ | ✘ |
| SNRW | ✔️ | ✔️ | ✘ | ✘ | ✘ | ✘ | ✘ | ✘ |
| X | ✘ | ✘ | ✘ | ✘ | ✘ | ✘ | ✘ | ✘ |
> 横向表示其它事务已经持有的锁，纵向表示事务想加的锁。

- 按对象/范围维度划分

| 属性 | 含义 | 范围/对象 |
| --- | --- | --- |
| GLOBAL | 全局锁 | 范围 |
| COMMIT | 提交保护锁 | 范围 |
| SCHEMA | 库锁 | 对象 |
| TABLE | 表锁 | 对象 |
| FUNCTION | 函数锁 | 对象 |
| PROCEDURE | 存储过程锁 | 对象 |
| TRIGGER | 触发器锁 | 对象 |
| EVENT | 事件锁 | 对象 |

- 几种典型语句的加(释放)锁流程图
1. `select`语句操作`MDL`锁流程
![](https://user-gold-cdn.xitu.io/2019/7/18/16c030aa8717aa49?w=998&h=800&f=png&s=76683)
2. `DML`语句操作`MDL`锁流程
![](https://user-gold-cdn.xitu.io/2019/7/18/16c030af0840b686?w=998&h=798&f=png&s=76471)
3. `alter`操作`MDL`锁流程
![](https://user-gold-cdn.xitu.io/2019/7/18/16c030b4f11cd8d9?w=1000&h=1240&f=png&s=140203)
- 几种典型语句的阻塞分析
> 注：`DML`（`UPDATE`、`INSERT`、`DELETE`）；`DDL`（`CREATE`、`ALTER`、`DROP`）；`DQL`（`SELECT`）。
1. `select`与`alter`是否会相互阻塞
> 当执行`select`语句时，只要`select`语句在获取`MDL_SHARED_READ`锁之前，`alter`没有执行到rename阶段，那么`select`获取`MDL_SHARED_READ`锁成功，后续有`alter`执行到rename阶段，请求`MDL_EXCLUSIVE`锁时，就会被阻塞。
2. `DML`与`alter`是否会相互阻塞
> `alter`在opening阶段会将锁升级到`MDL_SHARED_NO_WRITE`，rename阶段再将升级为`MDL_EXCLUSIVE`，由于`MDL_SHARED_NO_WRITE`与`MDL_SHARED_WRITE`互斥，所以先执行`alter`或先执行`DML`语句，都会导致语句阻塞在opening tables阶段。
3. `select`与`DML`是否会相互阻塞
> 由于`MDL_SHARED_WRITE`与`MDL_SHARED_READ`兼容，所以它们不会因为`MDL`而导致等待的情况。
### 按锁的范围划分
#### 全局锁
- 全局锁（FTWRL）是对整个**数据库实例**加锁
    - 加锁使用命令：
    ```
    flush tables with read lock;
    ```
    - 释放锁：
    ```
    unlock tables
    ```
- 加锁流程：
    - 上全局读锁（lock_global_read_lock）：所有更新操作都会被堵塞
    - 清理表缓存
        1. 关闭所有未使用的表对象
        2. 更新全局字典的版本号
        3. 对于在使用的表对象，逐一检查，若表还在使用中，调用MDL_wait::timed_wait进行等待
        4. 将等待对象关联到table_cache对象中
        5. 继续遍历使用的表对象
        6. 直到所有表都不再使用，则关闭成功。
    - 上全局COMMIT锁（make_global_read_lock_block_commit）：会堵塞活跃事务提交
> 全局锁的典型使用场景是，做全库逻辑备份。把整库每个表都select出来存成文本。如果不加锁会不会出现问题呢？

| 表数据变更状态 | 备份状态 |
| --- | --- |
| account表a用户有200元余额account(a,200)<br>course表没有任何数据 |  |
|  | 备份account表 得到account(a,200) |
| 用户买了一门Java课，花了100元。 |  |
|account表a用户有100元余额<br>course表数据为course(a,java) |  |
| | 这个时候我备份跑到course表了，那么备份结果就是course(a,java) |
> 得到最终备份结果便是account(a,200) course(a,java)，这显然是不对的，因为不加锁的话中间有业务变更，所得数据是不一致。
- 按照上面说的，我们只需要备份开始，如果数据库的引擎是`InnoDB`，隔离级别在`RR`下，开启个事务，就能拿到一致性视图。
- `MySQL`官方提供的备份工具`mysqldump`
> 当 `mysqldump` 使用参数`–single-transaction` 的时候，导数据之前就会启动一个事务，来确保拿到一致性视图。
#### 表级锁
> `MySQL` 里面表级别的锁有两种：一种是表锁，一种是`MDL`锁（`TABLE`范围）。
- 表锁的语法是 `lock tables … read/write`。与 FTWRL 类似，可以用 `unlock tables` 主动释放锁，也可以在客户端断开的时候自动释放。
- `MDL`锁，当属性为TABLE，作用范围为表级别的时候，它也是一把表锁。正如我们上面几种典型语句的加(释放)锁分析的过程中那样。它不需要显示的使用，因为`MySQL`会根据你的执行语句来分析是否加锁和加何种锁。
#### 行锁(Record Lock)
- 行锁的功与过
    - 两阶段锁协议
    > **两阶段锁协议**：在`InnoDB`事务中，行锁是需要时候才加锁，但不会不需要了就释放掉，而是等待事务提交结束时才释放。
    - 根据行锁的两阶段锁协议特性优化代码
![](https://user-gold-cdn.xitu.io/2019/7/18/16c030bd39cca8f8?w=800&h=800&f=png&s=47030)

    > 如图多个用户都点击下单的时候，产生锁竞争的主要场所是影院账户余额新增票价，两阶段锁协议特性是事务结束才释放锁，那么将这步骤放在最后是锁暂用时间最短。
    - 死锁问题

| 事务A | 事务B |
| --- | --- |
| update t set k = k+1 where id = 1; |  |
|  | update t set k = k+3 where id = 2; |
| update t set k = k+1 where id = 2; |  |
|  | update t set k = k+5 where id = 1; |
> 上面会出现死锁，解决方式便是按顺序加锁来避免死锁。

| 事务A | 事务B |
| --- | --- |
| update t set k = k+1 where id = 1; |  |
|  | update t set k = k+3 where id = 1; |
| update t set k = k+1 where id = 2; |  |
|  | update t set k = k+5 where id = 2; |
- 行锁的类别
    - 读锁（S）
    - 写锁（X）
- 行锁的加锁策略
> 对于`insert`，`update`，`delete`操作，`InnoDB`会自动给涉及到的数据加排他锁，只有`select`需要我们手动设置加锁级别。
- 行锁的加锁语句
```
-- 读锁（S锁）
select * from t where id = 1 lock in share mode;
-- 写锁（X锁）
select * from t where id = 1 for update;
```
#### Record Lock、Gap Lock与Next-Key Lock加锁图析
- 回顾下`InnoDB`的主键索引和辅助索引
主键索引（聚簇）

![](https://user-gold-cdn.xitu.io/2019/2/20/169091204e3f0e23?w=1644&h=694&f=png&s=48492)
辅助索引

![](https://user-gold-cdn.xitu.io/2019/2/20/16909129ee643b35?w=1768&h=702&f=png&s=49345)

- `Record Lock`加锁策略

![](https://user-gold-cdn.xitu.io/2019/7/18/16c030c61371c4b2?w=1008&h=416&f=png&s=56605)
> 对于主键索引，会在主键索引标上锁标记。对于普通索引，不只在普通索引标上锁标记，而且也会在主键索引标上。
- `Gap Lock`加锁策略

![](https://user-gold-cdn.xitu.io/2019/7/18/16c030ca23695e09?w=954&h=162&f=png&s=17099)

> 间隙锁它锁的是索引与索引之间的间隙。
- `Next-Key Lock`加锁策略

![](https://user-gold-cdn.xitu.io/2019/7/18/16c030cd4e1ca41e?w=982&h=186&f=png&s=21139)

> 由图可以发现`Next-Key Lock`等于`Record Lock`加上`Gap Lock`。左开右闭。
#### `Next-Key Lock`的需要知道的几个小事
- 两原则、两优化  
**原则1**：加锁的基本单位是`Next-Key Lock`。
**原则2**：查找过程中访问到的对象才会加锁。
**优化1**：索引上的等值查询，给唯一索引加锁的时候，`Next-Key Lock` 退化为行锁。
**优化2**：索引上的等值查询，向右遍历时且最后一个值不满足等值条件的时候，`Next-Key Lock` 退化为间隙锁。
> 写到这，需要明确一个事儿，`Next-Key Lock`是`InnoDB RR`隔离级别下的锁，是**内置锁**，是MySQL帮我解决某种场景锁引入的锁，那么这个场景是什么？**其实它想解决的是某种情况下的幻读场景**。
#### 幻读场景分析
> 幻读仅专指“**新插入的行**”
- 当前读场景下的幻读
表t的结构与数据

| id | c | d(key) |
| --- | --- | --- |
| 0 | 0 | 0 |
| 5 | 5 | 5 |
| 10 | 10 | 10 |
| 15 | 15 | 15 |
| 20 | 20 | 20 |
| 25 | 25 | 25 |

再看下面这个场景：

|  | session A | sessionB |
| --- | --- | --- |
| T1 | begin;</br> update t set d=100 where d=5 |  |
| T2 |  | insert into t values(1,1,5); |
| T3 | commit; |  |
- 为什么这种情况会被称为幻读，幻读有什么问题？
1. 语义上问题：
```
select * from t where d = 5 for update
```
> 因为本想锁住d=5，这句话的语义被破坏了，“新插入”了一行新的d=5的数据（1,1,5）。
2. 数据一致性的问题：
上面的情况执行结果：

| id | c | d(key) |
| --- | --- | --- |
| 0 | 0 | 0 |
| 5 | 5 | 100 |
| 10 | 10 | 10 |
| 15 | 15 | 15 |
| 20 | 20 | 20 |
| 25 | 25 | 25 |
| 1 | 1 | 5 |
我们再分析下`bin log`中的记录内容
```
insert into t values(1,1,5);
update t set d=100 where d=5;
```
> 可以发现`bin log`发生与原执行不同的结果，出现了数据不一致。为了解决这个问题，`InnoDB RR`级别下，行锁并不会阻止当前读情况下的幻读问题，才引入了上面所提到的`Next-Key Lock`。

![](https://user-gold-cdn.xitu.io/2019/7/18/16c030d26cafcfb6?w=982&h=308&f=png&s=30570)

> 写到这，还需要明确的一件事`Next-Key Lock`只会阻止往当前范围的`insert`动作。`InnoDB RR`级别使用行锁时候默认加的。


  