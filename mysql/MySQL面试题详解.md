# MySQL面试题详解

## 一、基础篇

### 1. MySQL存储引擎有哪些？InnoDB和MyISAM区别？

**常见存储引擎：**
- InnoDB（默认）
- MyISAM
- Memory
- Archive
- CSV

**InnoDB vs MyISAM：**

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| 事务支持 | 支持 | 不支持 |
| 锁粒度 | 行级锁 | 表级锁 |
| 外键 | 支持 | 不支持 |
| 全文索引 | MySQL 5.6+支持 | 支持 |
| 存储限制 | 64TB | 256TB |
| 崩溃恢复 | 支持（redo log） | 不支持 |
| 索引结构 | 聚簇索引 | 非聚簇索引 |
| 适用场景 | 高并发、事务处理 | 只读、统计分析 |

**选择建议：**
- 需要事务、外键、高并发写入：选择InnoDB
- 只读、统计分析、全文检索（老版本）：选择MyISAM

---

### 2. 索引类型有哪些？B+树结构特点？

**索引类型：**

1. **按数据结构分类：**
   - B+树索引（默认）
   - Hash索引（Memory引擎）
   - 全文索引
   - 空间索引（R-Tree）

2. **按物理存储分类：**
   - 聚簇索引：叶子节点存储完整数据行
   - 非聚簇索引（二级索引）：叶子节点存储主键值

3. **按逻辑分类：**
   - 主键索引
   - 唯一索引
   - 普通索引
   - 组合索引
   - 全文索引

**B+树特点：**

```
                    [根节点]
                   /    |    \
              [分支节点] [分支节点] [分支节点]
             /   |   \
        [叶子节点] [叶子节点] [叶子节点]
            ⇄      ⇄       ⇄
          (链表连接)
```

**核心特点：**
1. 非叶子节点只存储键值和指针，不存储数据
2. 叶子节点存储所有数据，用双向链表连接
3. 树高度低（通常3-4层），减少磁盘I/O
4. 范围查询效率高（叶子节点有序链表）

**为什么选择B+树？**
- 单节点存储更多键值，降低树高度
- 叶子节点链表支持范围查询
- 磁盘I/O次数少，适合数据库场景

---

### 3. 事务ACID特性是什么？

**A - Atomicity（原子性）：**
- 事务是不可分割的工作单位
- 要么全部成功，要么全部失败
- 实现：undo log（回滚日志）

**C - Consistency（一致性）：**
- 事务前后，数据库完整性约束不被破坏
- 数据从一个一致性状态到另一个一致性状态
- 实现：由原子性、隔离性、持久性共同保证

**I - Isolation（隔离性）：**
- 多个事务并发执行时互不干扰
- 实现：锁机制 + MVCC

**D - Durability（持久性）：**
- 事务提交后，数据永久保存
- 实现：redo log（重做日志）

---

## 二、进阶篇

### 4. 解释索引最左前缀原则

**定义：**
组合索引按照索引列定义顺序，从左到右依次匹配，必须连续使用前面的列才能使用后面的列。

**示例：**
```sql
-- 创建组合索引
CREATE INDEX idx_name_age_gender ON user(name, age, gender);

-- 能命中索引
WHERE name = '张三'
WHERE name = '张三' AND age = 18
WHERE name = '张三' AND age = 18 AND gender = '男'

-- 不能命中索引
WHERE age = 18                    -- 缺少最左列name
WHERE gender = '男'               -- 缺少name和age
WHERE name = '张三' AND gender = '男'  -- age列断层
```

**原理：**
- B+树索引按照定义顺序构建
- 类似字典查询，先按首字母，再按第二字母

**优化建议：**
- 最常用的列放在最左边
- 区分度高的列放在左边
- 范围查询的列放在最后

---

### 5. 什么情况下索引会失效？

**常见失效场景：**

1. **违反最左前缀原则**
```sql
-- 组合索引(a, b, c)
WHERE b = 1          -- 失效
WHERE c = 1          -- 失效
```

2. **在索引列上做运算**
```sql
WHERE YEAR(create_time) = 2023     -- 失效
WHERE create_time >= '2023-01-01' AND create_time < '2024-01-01'  -- 生效
```

3. **使用函数**
```sql
WHERE UPPER(name) = 'TOM'          -- 失效
```

4. **隐式类型转换**
```sql
-- name是varchar类型
WHERE name = 123                    -- 失效（数字转字符串）
WHERE name = '123'                  -- 生效
```

5. **使用NOT、!=、<>**
```sql
WHERE status != 1                   -- 可能失效
```

6. **LIKE以通配符开头**
```sql
WHERE name LIKE '%张'              -- 失效
WHERE name LIKE '张%'              -- 生效
```

7. **OR连接非索引列**
```sql
WHERE a = 1 OR b = 2               -- 如果b无索引，全部失效
```

8. **IS NULL / IS NOT NULL**（部分情况）
```sql
WHERE name IS NULL                  -- 可能失效，取决于数据分布
```

9. **全表扫描更快时**
- 数据量小
- 数据区分度低
- 优化器自动选择全表扫描

---

### 6. MVCC原理是什么？解决了什么问题？

**MVCC（Multi-Version Concurrency Control）：多版本并发控制**

**解决的问题：**
- 读写冲突：读不阻塞写，写不阻塞读
- 提高并发性能

**实现机制：**

1. **隐藏字段：**
```
DB_TRX_ID    -- 最后修改事务ID
DB_ROLL_PTR  -- 回滚指针，指向undo log
DB_ROW_ID    -- 隐藏主键（无主键时）
```

2. **Undo Log（回滚日志）：**
- 存储数据的历史版本
- 通过DB_ROLL_PTR形成版本链

```
当前数据 → undo log v3 → undo log v2 → undo log v1
```

3. **Read View（读视图）：**
```
m_ids        -- 当前活跃事务ID列表
min_trx_id   -- 最小活跃事务ID
max_trx_id   -- 下一个将分配的事务ID
creator_trx_id -- 创建Read View的事务ID
```

**可见性判断规则：**
```
1. DB_TRX_ID < min_trx_id  → 可见（事务已提交）
2. DB_TRX_ID >= max_trx_id → 不可见（事务在Read View后开启）
3. DB_TRX_ID in m_ids      → 不可见（事务未提交）
4. DB_TRX_ID not in m_ids  → 可见（事务已提交）
5. DB_TRX_ID = creator_trx_id → 可见（自己修改的）
```

**RR级别如何避免幻读：**
- 第一次查询生成Read View，后续查询复用
- 通过版本链找到可见数据

---

### 7. 脏读、幻读、不可重复读是什么？各隔离级别如何解决？

**三个问题：**

| 问题 | 描述 | 示例 |
|------|------|------|
| 脏读 | 读到未提交事务的数据 | A修改未提交，B读到修改后的数据，A回滚 |
| 不可重复读 | 同一事务内，前后读取数据不一致 | A读数据，B修改提交，A再读数据不同 |
| 幻读 | 同一事务内，前后查询结果集行数不同 | A查询5条，B插入1条，A再查变6条 |

**四种隔离级别：**

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 实现方式 |
|---------|------|----------|------|---------|
| READ UNCOMMITTED | ✗ | ✗ | ✗ | 无 |
| READ COMMITTED | ✓ | ✗ | ✗ | MVCC（每次生成Read View） |
| REPEATABLE READ | ✓ | ✓ | ✗ | MVCC（复用Read View）+ Next-Key Lock |
| SERIALIZABLE | ✓ | ✓ | ✓ | 锁（读加共享锁） |

**注意：**
- InnoDB在RR级别通过MVCC + Next-Key Lock已经解决了幻读
- MVCC解决快照读幻读，Next-Key Lock解决当前读幻读

**查看/设置隔离级别：**
```sql
-- 查看
SELECT @@transaction_isolation;

-- 设置
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

---

## 三、高级篇

### 8. Explain执行计划各字段含义？如何优化SQL？

**Explain使用：**
```sql
EXPLAIN SELECT * FROM user WHERE name = '张三';
```

**核心字段：**

| 字段 | 含义 | 关注点 |
|------|------|--------|
| id | 查询标识符 | 相同则从上到下执行，不同则大的先执行 |
| select_type | 查询类型 | SIMPLE/PRIMARY/SUBQUERY/DERIVED |
| table | 访问的表 | |
| type | 访问类型 | 从优到差：system > const > eq_ref > ref > range > index > ALL |
| possible_keys | 可能使用的索引 | |
| key | 实际使用的索引 | NULL表示未使用索引 |
| key_len | 索引长度 | 越短越好 |
| ref | 索引比较的列 | |
| rows | 预估扫描行数 | 越少越好 |
| filtered | 条件过滤比例 | 越高越好 |
| Extra | 额外信息 | Using index/Using where/Using filesort/Using temporary |

**type详解（性能从好到坏）：**
```
system    → 单行系统表
const     → 主键/唯一索引常量查询
eq_ref    → 主键/唯一索引关联查询
ref       → 非唯一索引等值查询
range     → 索引范围查询
index     → 索引全扫描
ALL       → 全表扫描（最差）
```

**Extra重要信息：**
```
Using index         → 覆盖索引，性能好
Using where         → WHERE条件过滤
Using filesort      → 文件排序，需优化
Using temporary     → 临时表，需优化
Using index condition → 索引下推
```

**优化策略：**

1. **避免全表扫描**
   - 添加合适的索引
   - 避免索引失效场景

2. **优化索引使用**
   - 使用覆盖索引减少回表
   - 选择合适的索引列顺序
   - 使用索引下推

3. **优化ORDER BY**
   - 利用索引排序
   - 避免Using filesort

4. **优化分页**
```sql
-- 传统分页（慢）
SELECT * FROM user LIMIT 1000000, 10;

-- 优化方式1：子查询
SELECT * FROM user a
JOIN (SELECT id FROM user LIMIT 1000000, 10) b ON a.id = b.id;

-- 优化方式2：记录上次最大ID
SELECT * FROM user WHERE id > 1000000 LIMIT 10;
```

---

### 9. 慢查询如何定位和优化？

**定位慢查询：**

1. **开启慢查询日志：**
```sql
-- 查看配置
SHOW VARIABLES LIKE '%slow_query%';

-- 开启慢查询日志
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;  -- 超过1秒记录
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';

-- 记录未使用索引的查询
SET GLOBAL log_queries_not_using_indexes = ON;
```

2. **使用工具分析：**
```bash
# mysqldumpslow
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log

# pt-query-digest（推荐）
pt-query-digest /var/log/mysql/slow.log
```

3. **实时监控：**
```sql
-- 查看正在执行的查询
SHOW PROCESSLIST;

-- 查看InnoDB状态
SHOW ENGINE INNODB STATUS;
```

**优化步骤：**

1. **分析执行计划**
```sql
EXPLAIN SELECT ...
```

2. **定位问题**
   - 是否使用索引？
   - 扫描行数是否过多？
   - 是否有Extra警告？

3. **针对性优化**

   **索引问题：**
   ```sql
   -- 添加索引
   CREATE INDEX idx_name ON table(column);

   -- 组合索引
   CREATE INDEX idx_composite ON table(col1, col2, col3);
   ```

   **SQL改写：**
   - 避免SELECT *，只查需要的列
   - 避免在WHERE子句中对字段进行函数操作
   - 用EXISTS替代IN（大表查询时）
   - 用UNION ALL替代UNION（不需要去重时）

   **表结构优化：**
   - 选择合适的数据类型
   - 适当的反范式（减少关联查询）
   - 分库分表

---

### 10. 分库分表策略有哪些？什么场景使用？

**分库分表维度：**

```
垂直分库：按业务拆分，如用户库、订单库、商品库
垂直分表：按字段拆分，如用户基本信息表、用户扩展信息表
水平分库：按数据行拆分到多个库，如用户库1、用户库2
水平分表：按数据行拆分到多个表，如订单表_202301、订单表_202302
```

**分片策略：**

1. **Hash取模分片**
```sql
-- 根据用户ID取模
user_id % 分片数 = 分片号

优点：数据分布均匀
缺点：扩容困难，需要数据迁移
```

2. **范围分片**
```sql
-- 按时间范围
order_202301, order_202302, ...

-- 按ID范围
user_0（1-100万）, user_1（100万-200万）

优点：扩容方便
缺点：可能存在热点数据
```

3. **一致性Hash**
```
优点：扩容时数据迁移量小
缺点：实现复杂
```

4. **地理位置分片**
```
按城市、区域分片
适用：区域性业务
```

**适用场景：**

| 场景 | 方案 |
|------|------|
| 单表数据量超过500万-1000万 | 水平分表 |
| 单库并发连接数过高 | 水平分库 |
| 业务模块耦合严重 | 垂直分库 |
| 大字段、冷热数据分离 | 垂直分表 |
| 跨地域访问延迟高 | 地理位置分库 |

**分库分表带来的问题：**

1. **跨库JOIN问题**
   - 解决：应用层关联、数据冗余、全局表

2. **分布式事务问题**
   - 解决：两阶段提交、TCC、最终一致性

3. **全局唯一ID问题**
   - 解决：UUID、雪花算法、号段模式

4. **跨库排序分页**
   - 解决：先查各分片再合并、冗余排序字段

5. **聚合函数问题**
   - 解决：各分片聚合后合并

**中间件选择：**
- ShardingSphere（推荐）
- MyCat
- Vitess

---

### 11. 主从复制原理？如何解决主从延迟？

**主从复制原理：**

```
Master                    Slave
  |                         |
  | 1.写入binlog            |
  |------------------------>|
  |                         | 2.写入relay log
  |                         | 3.重放relay log
  |                         | 4.数据同步完成
```

**三个线程：**

1. **Binlog Dump Thread（主库）**
   - 读取binlog，发送给从库

2. **I/O Thread（从库）**
   - 连接主库，接收binlog
   - 写入relay log

3. **SQL Thread（从库）**
   - 读取relay log
   - 执行SQL事件

**复制模式：**

| 模式 | 说明 | 特点 |
|------|------|------|
| 异步复制 | 主库不等待从库确认 | 性能高，可能丢数据 |
| 半同步复制 | 至少一个从库确认 | 性能折中，数据较安全 |
| 全同步复制 | 所有从库确认 | 性能低，数据安全 |

**主从延迟原因：**

1. 从库硬件配置低
2. 网络延迟
3. 大事务执行时间长
4. 从库执行其他慢查询
5. 单线程复制（MySQL 5.6之前）

**解决方案：**

1. **硬件优化**
   - 提升从库配置
   - 使用SSD
   - 优化网络

2. **参数优化**
```sql
-- 开启并行复制（MySQL 5.7+）
slave_parallel_workers = 4
slave_parallel_type = LOGICAL_CLOCK

-- 开启半同步复制
rpl_semi_sync_master_enabled = 1
rpl_semi_sync_slave_enabled = 1
```

3. **架构优化**
   - 读写分离：关键业务读主库
   - 缓存：减少数据库压力
   - 分库分表：降低单库压力

4. **业务优化**
   - 避免大事务
   - 批量操作分批执行
   - 非关键业务允许读延迟

**强制走主库场景：**
```java
// 伪代码示例
if (isWriteOperation || requireRealTimeData) {
    // 走主库
} else {
    // 走从库
}
```

---

### 12. 死锁产生条件？如何避免？

**死锁四个必要条件：**

1. **互斥条件**：资源不能共享
2. **请求与保持**：持有资源同时请求新资源
3. **不可剥夺**：资源不能强制抢占
4. **循环等待**：形成循环等待关系

**MySQL死锁示例：**

```sql
-- 事务A
BEGIN;
UPDATE account SET balance = balance - 100 WHERE id = 1;  -- 锁定id=1
UPDATE account SET balance = balance + 100 WHERE id = 2;  -- 等待id=2锁

-- 事务B（同时执行）
BEGIN;
UPDATE account SET balance = balance - 100 WHERE id = 2;  -- 锁定id=2
UPDATE account SET balance = balance + 100 WHERE id = 1;  -- 等待id=1锁

-- 死锁产生！
```

**如何检测死锁：**

```sql
-- 查看死锁日志
SHOW ENGINE INNODB STATUS;

-- 查看锁信息
SELECT * FROM information_schema.INNODB_LOCKS;
SELECT * FROM information_schema.INNODB_LOCK_WAITS;
```

**避免死锁策略：**

1. **按固定顺序访问资源**
```sql
-- 所有事务都按id升序操作
BEGIN;
UPDATE account SET balance = balance - 100 WHERE id = 1;
UPDATE account SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

2. **大事务拆小事务**
```sql
-- 不好的做法
BEGIN;
-- 大量操作
COMMIT;

-- 好的做法：拆分成多个小事务
BEGIN;
UPDATE account SET balance = balance - 100 WHERE id = 1;
COMMIT;

BEGIN;
UPDATE account SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

3. **降低隔离级别**（不推荐）
   - RC级别不会产生间隙锁，减少死锁概率

4. **设置锁等待超时**
```sql
-- 设置锁等待超时时间（秒）
SET innodb_lock_wait_timeout = 5;
```

5. **合理使用索引**
   - 无索引会锁住更多行
   - 创建合适索引减少锁范围

6. **使用乐观锁替代悲观锁**
```sql
-- 悲观锁（可能死锁）
SELECT * FROM account WHERE id = 1 FOR UPDATE;

-- 乐观锁（版本号）
UPDATE account 
SET balance = balance - 100, version = version + 1 
WHERE id = 1 AND version = old_version;
```

**死锁处理：**
- MySQL自动检测并回滚代价最小的事务
- 应用层捕获死锁异常并重试

---

### 13. 为什么InnoDB使用B+树而不是B树或红黑树？

**对比分析：**

| 数据结构 | 树高度 | 磁盘I/O | 范围查询 | 适用场景 |
|---------|--------|---------|---------|---------|
| B+树 | 低（3-4层） | 少 | 快（叶子链表） | 数据库索引 |
| B树 | 中等 | 较多 | 慢 | 文件系统 |
| 红黑树 | 高（log₂N） | 多 | 慢 | 内存结构 |
| Hash | O(1) | - | 不支持 | 等值查询 |

**选择B+树的原因：**

**1. 磁盘I/O更少**
```
B+树非叶子节点只存键值和指针：
- 16KB页可存储更多键值
- 树高度更低（通常3-4层）
- 查询只需3-4次磁盘I/O

B树非叶子节点存储数据：
- 16KB页存储键值少
- 树高度更高
- 需要更多磁盘I/O
```

**示例计算：**
```
假设：
- 一行数据1KB
- 主键bigint占8字节
- 指针6字节
- 一页16KB

B+树：
- 非叶子节点可存：16KB / (8+6) ≈ 1170个键值
- 叶子节点可存：16KB / 1KB = 16条数据
- 3层B+树可存：1170 × 1170 × 16 ≈ 2190万条数据

B树：
- 非叶子节点存储数据，存储量大幅减少
- 需要更多层才能存储相同数据量
```

**2. 范围查询效率高**
```
B+树：
- 叶子节点用双向链表连接
- 找到起点后顺序遍历
- SELECT * FROM user WHERE id BETWEEN 1 AND 100

B树：
- 需要中序遍历
- 在不同层级间跳转
- 效率低
```

**3. 查询性能稳定**
```
B+树：
- 所有数据都在叶子节点
- 查询时间稳定

B树：
- 数据可能在不同层级
- 查询时间波动
```

**为什么不选红黑树？**
```
红黑树特点：
- 平衡二叉树，高度log₂N
- 1000万数据：log₂(10000000) ≈ 24层
- 每层一次磁盘I/O
- 需要24次磁盘I/O

B+树：
- 1000万数据：3-4层
- 只需3-4次磁盘I/O
- 性能差距巨大
```

**为什么不选Hash？**
```
Hash索引优点：
- 等值查询O(1)

Hash索引缺点：
- 不支持范围查询
- 不支持排序
- 不支持最左前缀匹配
- 不支持模糊查询
```

**总结：**
B+树是磁盘存储的最佳平衡：
- 单节点存储大量键值，降低树高度
- 叶子链表支持高效范围查询
- 适合磁盘I/O密集场景

---

## 四、实战篇

### 14. 大表数据删除/更新如何处理？

**问题：**
- 大事务导致锁表时间长
- binlog日志暴增
- 主从延迟增大

**删除大量数据方案：**

**方案1：分批删除**
```sql
-- 每次删除1000条
DELETE FROM log WHERE create_time < '2022-01-01' LIMIT 1000;

-- 循环执行直到影响行数为0
-- 应用层实现循环或存储过程
```

**方案2：使用临时表**
```sql
-- 创建新表保留需要的数据
CREATE TABLE log_new LIKE log;
INSERT INTO log_new SELECT * FROM log WHERE create_time >= '2022-01-01';

-- 重命名表（原子操作）
RENAME TABLE log TO log_old, log_new TO log;

-- 删除旧表
DROP TABLE log_old;
```

**方案3：分区表**
```sql
-- 创建按时间分区的表
CREATE TABLE log (
    id BIGINT,
    create_time DATETIME
) PARTITION BY RANGE (YEAR(create_time)) (
    PARTITION p2021 VALUES LESS THAN (2022),
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024)
);

-- 删除旧分区（瞬间完成）
ALTER TABLE log DROP PARTITION p2021;
```

**大批量更新方案：**

```sql
-- 不好的做法
UPDATE user SET status = 1 WHERE age > 18;

-- 分批更新
UPDATE user SET status = 1 WHERE age > 18 LIMIT 1000;

-- 使用临时表
CREATE TABLE user_update AS
SELECT id FROM user WHERE age > 18;

-- 分批关联更新
UPDATE user u 
JOIN user_update tmp ON u.id = tmp.id 
SET u.status = 1 
LIMIT 1000;
```

**注意事项：**
1. 避开业务高峰期
2. 监控主从延迟
3. 预估执行时间
4. 做好回滚预案
5. 删除后执行`OPTIMIZE TABLE`回收空间

---

### 15. 如何设计一个高并发订单系统？

**核心挑战：**
- 高并发下单
- 库存扣减一致性
- 订单数据量增长

**架构设计：**

```
用户 → CDN → 负载均衡 → [应用服务器集群]
                              ↓
         [缓存层：Redis集群] ← [消息队列]
                              ↓
         [数据库：MySQL主从/分库分表]
```

**数据库设计：**

```sql
-- 订单主表（垂直拆分核心字段）
CREATE TABLE `order` (
    `id` BIGINT PRIMARY KEY,
    `order_no` VARCHAR(64) NOT NULL COMMENT '订单号',
    `user_id` BIGINT NOT NULL COMMENT '用户ID',
    `total_amount` DECIMAL(10,2) NOT NULL COMMENT '总金额',
    `status` TINYINT NOT NULL COMMENT '订单状态',
    `create_time` DATETIME NOT NULL,
    `update_time` DATETIME NOT NULL,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_order_no` (`order_no`),
    KEY `idx_user_id` (`user_id`),
    KEY `idx_create_time` (`create_time`)
) ENGINE=InnoDB;

-- 订单明细表（垂直拆分）
CREATE TABLE `order_item` (
    `id` BIGINT PRIMARY KEY,
    `order_id` BIGINT NOT NULL,
    `product_id` BIGINT NOT NULL,
    `quantity` INT NOT NULL,
    `price` DECIMAL(10,2) NOT NULL,
    KEY `idx_order_id` (`order_id`)
) ENGINE=InnoDB;

-- 商品库存表
CREATE TABLE `product_stock` (
    `product_id` BIGINT PRIMARY KEY,
    `stock` INT NOT NULL,
    `version` INT NOT NULL DEFAULT 0 COMMENT '乐观锁版本号'
) ENGINE=InnoDB;
```

**核心方案：**

**1. 库存扣减（防超卖）**

```sql
-- 乐观锁方案
UPDATE product_stock 
SET stock = stock - 1, version = version + 1 
WHERE product_id = 1 AND stock > 0 AND version = old_version;

-- 影响行数为0则扣减失败

-- 悲观锁方案（高并发不推荐）
SELECT stock FROM product_stock WHERE product_id = 1 FOR UPDATE;
UPDATE product_stock SET stock = stock - 1 WHERE product_id = 1;
COMMIT;

-- Redis预扣库存（推荐）
-- 1. 库存预热到Redis
SET product:stock:1 100

-- 2. Redis原子扣减
DECR product:stock:1

-- 3. 异步同步到MySQL
```

**2. 唯一订单号生成**

```java
// 雪花算法
public class SnowflakeIdGenerator {
    private final long twepoch = 1288834974657L;
    private final long workerIdBits = 5L;
    private final long datacenterIdBits = 5L;
    private final long maxWorkerId = -1L ^ (-1L << workerIdBits);
    private final long maxDatacenterId = -1L ^ (-1L << datacenterIdBits);
    private final long sequenceBits = 12L;
    
    private long workerId;
    private long datacenterId;
    private long sequence = 0L;
    private long lastTimestamp = -1L;
    
    public synchronized long nextId() {
        long timestamp = timeGen();
        
        if (timestamp < lastTimestamp) {
            throw new RuntimeException("时钟回拨");
        }
        
        if (lastTimestamp == timestamp) {
            sequence = (sequence + 1) & ((1 << sequenceBits) - 1);
            if (sequence == 0) {
                timestamp = tilNextMillis(lastTimestamp);
            }
        } else {
            sequence = 0L;
        }
        
        lastTimestamp = timestamp;
        
        return ((timestamp - twepoch) << (sequenceBits + workerIdBits + datacenterIdBits))
                | (datacenterId << (sequenceBits + workerIdBits))
                | (workerId << sequenceBits)
                | sequence;
    }
}
```

**3. 分库分表策略**

```
分库键：user_id（用户维度查询）
分表键：create_time（时间范围查询）

或使用基因法：
user_id后几位作为分库分表基因
保证同一用户的订单在同一库表
```

**4. 缓存设计**

```
热点数据缓存：
- 商品信息：Redis缓存，过期时间设置
- 库存信息：Redis缓存，原子操作
- 用户订单列表：Redis缓存分页数据

缓存一致性：
- 延迟双删
- 订阅binlog更新缓存
```

**5. 读写分离**

```java
// 强制读主库的场景
if (orderJustCreated) {
    // 创建订单后立即查询，走主库
    routingDataSource.setMaster();
}

// 其他场景走从库
routingDataSource.setSlave();
```

**6. 异步处理**

```
下单流程：
用户下单 → 扣库存(Redis) → 创建订单(MySQL) → 发送消息(MQ)
                                          ↓
                     [消费者] → 扣减MySQL库存、发送通知、积分等
```

**7. 限流降级**

```java
// 接口限流
@RateLimiter(qps = 1000)
public Result createOrder(OrderRequest request) {
    // 下单逻辑
}

// 服务降级
@SentinelResource(value = "createOrder", fallback = "createOrderFallback")
public Result createOrder(OrderRequest request) {
    // 下单逻辑
}

public Result createOrderFallback(OrderRequest request) {
    return Result.fail("系统繁忙，请稍后重试");
}
```

**8. 分页查询优化**

```sql
-- 传统分页（慢）
SELECT * FROM `order` ORDER BY create_time DESC LIMIT 1000000, 10;

-- 优化：记录上次最大ID
SELECT * FROM `order` WHERE id < #{lastId} ORDER BY id DESC LIMIT 10;

-- 优化：覆盖索引+关联
SELECT o.* FROM `order` o
INNER JOIN (SELECT id FROM `order` ORDER BY create_time DESC LIMIT 1000000, 10) t
ON o.id = t.id;
```

---

### 16. 遇到过哪些MySQL性能问题？如何解决的？

**问题1：慢查询**

**场景：**
某查询接口响应时间5秒以上

**排查：**
```sql
-- 查看慢查询日志
mysqldumpslow -s t -t 10 slow.log

-- 分析执行计划
EXPLAIN SELECT * FROM order WHERE user_id = 123;
```

**原因：**
- user_id字段无索引
- 全表扫描

**解决：**
```sql
CREATE INDEX idx_user_id ON order(user_id);
```

**效果：**
查询时间从5秒降到50毫秒

---

**问题2：死锁**

**场景：**
高并发扣库存时频繁死锁

**排查：**
```sql
SHOW ENGINE INNODB STATUS;
```

**原因：**
多个事务以不同顺序更新库存记录

**解决：**
```java
// 所有事务按库存ID升序操作
List<Long> sortedIds = productIds.stream()
    .sorted()
    .collect(Collectors.toList());

for (Long id : sortedIds) {
    // 扣减库存
}
```

---

**问题3：主从延迟**

**场景：**
用户创建订单后立即查询，查不到数据

**排查：**
```sql
-- 查看从库延迟
SHOW SLAVE STATUS\G
-- Seconds_Behind_Master: 5
```

**原因：**
- 从库配置低
- 大事务执行时间长

**解决：**
```java
// 方案1：关键业务读主库
if (isJustCreated) {
    masterDataSource.query();
} else {
    slaveDataSource.query();
}

// 方案2：缓存
String order = redis.get("order:" + orderId);
if (order == null) {
    order = db.query();
    redis.set("order:" + orderId, order, 60);
}
```

---

**问题4：索引失效**

**场景：**
有索引但查询慢

**排查：**
```sql
EXPLAIN SELECT * FROM user WHERE YEAR(create_time) = 2023;
-- type: ALL，全表扫描
```

**原因：**
在索引列上使用函数

**解决：**
```sql
-- 改写查询条件
SELECT * FROM user 
WHERE create_time >= '2023-01-01' AND create_time < '2024-01-01';
```

---

**问题5：连接数耗尽**

**场景：**
应用报错"Too many connections"

**排查：**
```sql
SHOW VARIABLES LIKE 'max_connections';
-- 151

SHOW STATUS LIKE 'Threads_connected';
-- 150
```

**原因：**
- 连接池配置不合理
- 长连接未释放

**解决：**
```sql
-- 临时解决
SET GLOBAL max_connections = 500;

-- 永久解决：my.cnf
max_connections = 500
wait_timeout = 3600
```

```java
// 应用层：配置连接池
HikariConfig config = new HikariConfig();
config.setMaximumPoolSize(50);
config.setMinimumIdle(10);
config.setIdleTimeout(600000);
config.setMaxLifetime(1800000);
```

---

**问题6：大表DDL**

**场景：**
给千万级表添加字段，锁表30分钟

**解决：**
```bash
# 使用pt-online-schema-change
pt-online-schema-change \
  --alter "ADD COLUMN age INT" \
  --execute D=db,t=user

# 或使用gh-ost
gh-ost --max-load=Threads_running=25 \
  --critical-load=Threads_running=1000 \
  --chunk-size=1000 \
  --throttle-control-replicas="..." \
  --database=db \
  --table=user \
  --alter="ADD COLUMN age INT" \
  --allow-master-master \
  --execute
```

---

**问题7：CPU飙升**

**场景：**
MySQL CPU使用率100%

**排查：**
```sql
-- 查看正在执行的SQL
SHOW FULL PROCESSLIST;

-- 找到耗时SQL
SELECT * FROM information_schema.processlist 
WHERE time > 5 AND command != 'Sleep';

-- 查看哪些SQL执行频率高
SELECT * FROM sys.statements_with_runtimes_in_95th_percentile;
```

**原因：**
- 缺少索引导致全表扫描
- 复杂查询
- 锁等待

**解决：**
```sql
-- 临时终止问题SQL
KILL 12345;

-- 添加索引
CREATE INDEX idx_create_time ON log(create_time);

-- 优化SQL
-- 拆分复杂查询
-- 使用覆盖索引
-- 减少返回字段
```

---

## 总结

MySQL面试考察核心点：

**基础能力：**
- 存储引擎、索引原理
- 事务ACID、隔离级别

**进阶能力：**
- 索引优化、SQL调优
- MVCC原理、锁机制

**架构能力：**
- 分库分表设计
- 主从复制、高可用

**实战经验：**
- 问题定位（慢查询、死锁）
- 性能优化
- 高并发方案设计

**学习方法：**
1. 理解原理（B+树、MVCC、锁）
2. 动手实践（搭建环境、复现问题）
3. 阅读源码（InnoDB源码）
4. 关注新技术（MySQL 8.0新特性）