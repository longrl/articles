### mysql笔记
**记录一些mysql相关的笔记和思考**

#### mysql体系结构
客户端连接
连接池组件  管理服务工具  sql接口组件  查询分析器组件  优化器组件  缓冲组件
插入式存储引擎
物理文件

常见的存储引擎：innodb、myisam、memory等等吧，但是最常见的还是innodb吧，innodb支持事务完整的实现了acid（原子性、一致性、隔离性、持久性），也支持行锁和外键，相比myisam就不支持事务，也只有表锁没有行锁；


#### 数据增删改查的实现

buffer pool
redo log
undo log
bin log

free list
flush list
lru list


#### 索引的实现


B+树
表空间 + 数据页


#### 并发下的事务管理

快照读
当前读

mvcc + 锁机制
read-view   undo-log链条



#### sql执行计划分析：exlpain

const  ref  range  index

#### mysql主从

bin log 
relay log

