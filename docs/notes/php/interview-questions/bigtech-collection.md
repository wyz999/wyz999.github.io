---
title: 大厂面试合集（一）
createTime: 2025/08/20 15:00:00
permalink: /php/面试题/bigtech-collection/
---
---

---

> 使用说明：本页收录常见于一线/大厂的高频与高难面试题，已按主题分组并预留“答案”区，便于集中练习与复盘。

## Mysql

### 1. 有哪些事务隔离级别？MySQL 的事务隔离级别是怎么实现的？（每家都问）

<details>
<summary>答案与解析</summary> 

#### 一、SQL 标准定义的四大事务隔离级别（按一致性从低到高）
事务隔离级别的核心是平衡“并发性能”与“数据一致性”，解决脏读、不可重复读、幻读三大问题：
1. **读未提交（Read Uncommitted, RU）**  
   - 核心特性：允许事务读取其他事务**未提交**的修改数据；  
   - 问题：存在**脏读**（读取未提交的临时数据，后续可能回滚）、不可重复读、幻读；  
   - 适用场景：仅用于临时统计（如实时访问量）等对一致性无要求的场景，生产环境极少使用。

2. **读已提交（Read Committed, RC）**  
   - 核心特性：仅允许事务读取其他事务**已提交**的修改数据；  
   - 问题：解决脏读，但仍存在**不可重复读**（同一事务内多次读同一数据，结果因其他事务提交而变化）、**幻读**（同一查询条件下，前后查询行数不同）；  
   - 适用场景：Oracle、PostgreSQL 等数据库默认级别，适合对一致性要求较低的业务（如商品浏览、非核心数据查询）。

3. **可重复读（Repeatable Read, RR）**  
   - 核心特性：同一事务内多次读取同一数据，结果始终一致（不受其他事务提交影响）；  
   - 问题：解决脏读、不可重复读，**InnoDB 额外解决幻读**（非 SQL 标准增强，区别于其他数据库）；  
   - 适用场景：MySQL InnoDB 默认级别，平衡一致性与性能（如订单创建、用户信息修改等核心业务）。

4. **串行化（Serializable）**  
   - 核心特性：事务串行执行（读写互斥，写操作阻塞读，读操作阻塞写）；  
   - 问题：无并发问题，但性能极低（锁粒度为表级）；  
   - 适用场景：仅用于金融对账、库存清算等一致性要求极高的场景。


#### 二、MySQL InnoDB 隔离级别的实现原理（核心：MVCC + 锁机制）
InnoDB 通过 **MVCC（多版本并发控制）** 实现“非锁定读”（读不阻塞写、写不阻塞读），通过 **锁机制** 实现“锁定读”（解决幻读等问题），不同级别通过调整“Read View 生成时机”和“锁粒度”实现：

1. **读未提交（RU）**  
   - 无 Read View 生成：直接读取数据的**最新版本**（即使数据未提交）；  
   - 无锁参与：无需加任何锁，因此存在脏读，性能虽高但一致性极差。

2. **读已提交（RC）**  
   - Read View 生成时机：**每次执行 SELECT 时重新生成**，仅可见“已提交事务的版本”（解决脏读）；  
   - 锁机制：仅对写入数据加**行锁**（Record Lock），无间隙锁；  
   - 不可重复读原因：因 Read View 每次更新，同一事务内多次读可能看到不同版本（如事务 A 第一次读数据为“张三”，事务 B 提交修改为“李四”后，事务 A 再次读变为“李四”）。

3. **可重复读（RR）**  
   - Read View 生成时机：**仅在事务第一次执行 SELECT 时生成**，后续读复用该视图（确保同一事务内结果一致，解决不可重复读）；  
   - 幻读解决：通过 **Next-Key Lock（行锁+间隙锁）** 锁定“查询条件覆盖的行+间隙”，防止其他事务插入/删除符合条件的行；  
     - 示例：查询 `age > 25` 时，InnoDB 会锁定所有 `age > 25` 的行（行锁），以及 `age=25` 到最大存在值、最大存在值到 ∞ 的间隙（间隙锁），避免插入新行导致幻读。

4. **串行化（Serializable）**  
   - 放弃 MVCC：直接对读取的行加 **共享锁（S 锁）**，写入时加 **排他锁（X 锁）**；  
   - 事务串行执行：完全隔离，但并发能力骤降（同一时间仅一个事务可操作数据）。


#### 三、关键补充
- MVCC 依赖 **undo 日志**（存储数据历史版本，用于回滚或追溯）和 **事务 ID（trx_id）**（标记数据版本归属）；  
- RR 级别是 InnoDB 对 SQL 标准的核心增强——通过间隙锁实现“幻读防护”，这是与 Oracle（RC 级别默认）的关键区别；  
- 查看/设置隔离级别：通过 `SELECT @@transaction_isolation;` 查看，`SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;` 设置会话级别。
</details>

### 2. 索引原理（每家都问）

<details>
<summary>答案与解析</summary> 

#### 一、索引的核心价值
索引是数据库优化查询的“加速器”，通过“有序数据结构”减少磁盘 IO 次数，将全表扫描（时间复杂度 O(n)）转为索引查找（O(log n)），本质是“用空间换时间”——索引会占用额外磁盘空间，但能大幅提升查询效率。


#### 二、MySQL 主流索引结构与原理
MySQL 不同引擎支持的索引结构不同，InnoDB/MyISAM 核心依赖 B+ 树，Memory 支持哈希索引，以下是高频考点结构：

##### 1. B+ 树索引（InnoDB/MyISAM 核心，面试重点）
B+ 树是“平衡多叉树”，专为磁盘 IO 优化设计，结构特点：
- **非叶子节点**：仅存储索引键（不存数据），节省空间，提升单次 IO 加载的键数量（降低树高）；  
- **叶子节点**：按顺序链表连接（支持范围查询，如 `age > 25`），存储数据或数据地址；  
- **树高低**：百万级数据仅 3-4 层，每次查询仅需 3-4 次磁盘 IO（磁盘 IO 是数据库性能瓶颈）。

InnoDB 中 B+ 树索引分为两类，这是与 MyISAM 的核心区别：
- **聚簇索引（主键索引）**：  
  - 叶子节点直接存储**完整数据行**，表数据按主键顺序存储；  
  - 优势：主键查询极快（直接定位数据，无需二次查找）；  
  - 限制：一张表仅 1 个聚簇索引（默认主键；无主键则选唯一索引；无唯一索引则隐式生成 6 字节 RowID 作为聚簇索引）。

- **辅助索引（非主键索引）**：  
  - 叶子节点存储“主键值”，而非完整数据；  
  - 查询流程：通过辅助索引找到主键 → 再通过聚簇索引查找数据（称为“回表”）；  
  - 例外：**覆盖索引**（查询字段均在辅助索引中）可避免回表，如 `SELECT id, name FROM user WHERE name = '张三'`（若 `name` 索引包含 `id`，则无需回表）。

##### 2. 哈希索引（Memory 引擎/InnoDB 自适应）
- **结构**：基于哈希表，通过哈希函数将索引键映射为哈希值，快速定位数据（时间复杂度 O(1)）；  
- **优势**：等值查询极快（如 `id = 1001`）；  
- **劣势**：不支持范围查询（如 `age > 25`）、排序（哈希值无序）、模糊查询（如 `LIKE '张%'`）；  
- **适用场景**：纯等值查询场景（如缓存 key 映射），InnoDB 仅在“高频等值查询”时自动生成自适应哈希索引。

##### 3. 全文索引（InnoDB 5.6+/MyISAM）
- **结构**：基于“倒排索引”，将长文本拆分为词项（Token，如“PHP 索引”拆为“PHP”“索引”），映射到包含该词项的行；  
- **优势**：支持关键词匹配（如 `MATCH(content) AGAINST('PHP 索引')`）；  
- **劣势**：不支持精确查询（性能低于 B+ 树），仅支持 `TEXT/VARCHAR` 字段；  
- **适用场景**：文章搜索、日志检索等文本匹配场景。


#### 三、索引失效的核心场景（高频考点）
即使创建索引，若查询条件不符合规则，索引仍会失效（退化为全表扫描），常见场景：
1. 索引列加函数/运算（如 `YEAR(birthday) = 1990`、`id + 1 = 1002`）；  
2. 字符串条件无引号（如 `email = 123`，触发“字符串转数字”类型转换，索引失效）；  
3. 联合索引不满足“最左前缀原则”（如 `(a,b,c)` 索引，查询 `b=2` 或 `c=3` 时失效）；  
4. `LIKE '%xxx'` 前缀模糊查询（如 `name LIKE '%三'`，无法利用索引有序性）；  
5. `OR` 连接非索引列（如 `age=25 OR gender='男'`，若 `gender` 无索引，整体索引失效）；  
6. 使用 `NOT IN`/`!=`/`<>`（除 `NOT IN (NULL)` 外，多数情况索引失效）。


#### 四、索引设计原则（实战重点）
1. **高区分度列优先**：选择区分度高的列（如 `user_id` 区分度 100%，`gender` 区分度 2%），避免为低区分度列建索引；  
2. **联合索引按“最左前缀+高频查询列”排序**：如高频查询 `a=1 AND b=2`，建 `(a,b)` 而非 `(b,a)`；  
3. **避免冗余索引**：若已建 `(a,b)` 索引，无需再建 `(a)` 索引（`(a,b)` 已覆盖 `(a)` 的查询需求）；  
4. **大字段用前缀索引**：如 `email(20)`（仅索引前 20 个字符），减少索引体积；  
5. **控制索引数量**：每张表索引不超过 5-8 个（索引会降低写入性能，每次写入需同步更新索引）。
</details>

### 3. 分库分表的策略；若以分表字段以外的字段作为查询条件怎么办（每家都问）

<details>
<summary>答案与解析</summary> 

#### 一、分库分表核心策略（按拆分维度分两类）
分库分表的核心目标是解决“单库/单表数据量过大”（通常单表超 1000 万行、单库超 10 亿行）导致的查询慢、写入慢问题，分为垂直拆分和水平拆分：

##### 1. 垂直拆分（按“字段/业务”拆分，解决“表过大/库过大”）
垂直拆分不改变数据行数，仅按“字段关联性”或“业务模块”拆分，分为两类：
- **垂直分库**：  
  - 逻辑：按业务模块拆分数据库（如 `user_db` 存储用户数据，`order_db` 存储订单数据，`product_db` 存储商品数据）；  
  - 优势：降低单库压力，业务边界清晰（不同模块团队独立维护）；  
  - 适用场景：单库包含多个独立业务模块（如电商系统的用户、订单、商品模块），且各模块数据量均较大。

- **垂直分表**：  
  - 逻辑：按字段访问频率拆分表（如 `user_base` 存高频字段 `id/name/email/phone`，`user_extend` 存低频大字段 `avatar/intro/address`）；  
  - 优势：减少单表数据量，提升查询速度（避免大字段拖累 IO，如 `avatar` 是 Base64 字符串，占用空间大）；  
  - 适用场景：表字段过多（50+ 字段）、含 `TEXT/BLOB` 大字段，且高频查询仅涉及少数字段。

##### 2. 水平拆分（按“数据行”拆分，解决“行数过多”）
水平拆分将单表数据按规则分散到多个表/库，数据行数减少，是分库分表的核心场景，按拆分规则分为三类：
- **范围拆分**：  
  - 逻辑：按时间（如 `order_202401`、`order_202402`）或 ID 范围（`user_1_100w`、`user_100w_200w`）拆分；  
  - 优势：扩容简单（新增 `order_202403` 即可）、范围查询高效（如查 2024 年 1 月订单，仅需访问 `order_202401`）；  
  - 劣势：热点数据倾斜（如最新月份订单表 `order_202405` 访问量远高于历史表）；  
  - 适用场景：时间序列数据（订单、日志）、ID 自增且范围查询频繁的场景。

- **哈希拆分**：  
  - 逻辑：按分表字段哈希取模（如 `user_id % 8` 分 8 张表，`order_no % 16` 分 16 张表）；  
  - 优势：数据分布均匀，无热点倾斜（每个表访问量相近）；  
  - 劣势：范围查询需跨表聚合（如查 `user_id 1-100` 需遍历 8 张表，用 `UNION ALL` 合并结果）；  
  - 适用场景：无明显热点、范围查询少的场景（如用户数据、支付记录）。

- **一致性哈希拆分**：  
  - 逻辑：将分表节点映射到“哈希环”（0-2^32-1），按分表字段哈希值定位节点；  
  - 优势：扩容时仅迁移部分数据（解决普通哈希“扩容全量迁移”问题）；  
  - 优化：通过“虚拟节点”（每个物理节点对应 100-200 个虚拟节点）解决节点少导致的数据倾斜；  
  - 适用场景：分布式缓存、分库分表扩容频繁的场景。


#### 二、跨分表字段查询的解决方案（核心痛点）
分表字段（如 `user_id`）是路由依据（通过分表字段确定数据在哪个表），若查询条件不含分表字段（如查 `order` 表的 `status=1`、`order_no='20240501001'`），需按场景选择方案：

##### 1. 方案 1：二级索引表（路由表，推荐）
- **适用场景**：高频非分表字段查询（如 `order_no` 查订单、`phone` 查用户）；  
- **实现逻辑**：  
  1. 创建二级索引表（如 `order_no_route`），存储“非分表字段（如 `order_no`）”与“分表标识（如 `table_index`）”的映射；  
  2. 数据写入时：同步写入业务表（如 `order_0`）和索引表（如 `order_no='20240501001'` 对应 `table_index=0`），通过事务或消息队列保证一致性；  
  3. 数据查询时：先查索引表获 `table_index`，再路由到目标业务表（如 `order_0`）查询；  
- **优势**：精准路由，无跨表查询，性能高；  
- **示例**：  
  ```sql
  -- 二级索引表
  CREATE TABLE order_no_route (
    order_no VARCHAR(64) PRIMARY KEY,
    table_index INT NOT NULL COMMENT '分表索引（0-7）',
    create_time DATETIME NOT NULL
  );
  -- 查询时先查索引表
  SELECT table_index FROM order_no_route WHERE order_no='20240501001';
  -- 再路由到 order_${table_index} 查详情
  SELECT * FROM order_0 WHERE order_no='20240501001';
  ```

##### 2. 方案 2：全局表/广播表
- **适用场景**：查询条件是“全局唯一且数据量小”的字段（如 `merchant_id` 查商户下的订单、`category_id` 查分类下的商品）；  
- **实现逻辑**：  
  1. 创建全局表（如 `merchant_user_map`），存储“非分表字段（如 `merchant_id`）”与“分表字段范围（如 `user_id_min`、`user_id_max`）”的映射；  
  2. 查询时：先查全局表获 `user_id` 范围，再路由到对应分表（如 `user_id 1-100w` 对应 `user_0`）；  
- **优势**：无跨表查询，适合小数据量维度（如商户数仅 1 万）；  
- **限制**：非分表字段需全局唯一且数据量小，否则全局表会成为瓶颈。

##### 3. 方案 3：跨表聚合查询（万不得已）
- **适用场景**：低频非分表字段查询（如后台统计 `status=1` 的订单总数、导出某类商品所有数据）；  
- **实现逻辑**：通过中间件（Sharding-JDBC/MyCat）自动遍历所有分表，用 `UNION ALL` 聚合结果；  
- **优化手段**：  
  1. 限制查询频率（如每分钟 1 次，避免频繁跨表）；  
  2. 结果缓存（将统计结果存入 Redis，过期时间 5 分钟）；  
  3. 分页查询时限制页数（如最多支持前 100 页，避免全量遍历）；  
- **劣势**：性能差，仅适合非核心、低频场景。

##### 4. 方案 4：冗余分表字段（从源头规避）
- **适用场景**：设计阶段可预见的跨字段查询；  
- **实现逻辑**：将分表字段冗余到查询条件字段的索引中，通过分表字段路由后再过滤；  
  - 示例：`order` 表按 `user_id` 分表，查 `status=1` 时，建联合索引 `idx_status_user(status, user_id)`；  
  - 查询时：先按 `user_id` 路由到分表，再通过 `status=1` 过滤（利用联合索引）；  
- **优势**：无需额外表，性能最优；  
- **限制**：依赖前期设计，后期改造成本高。


#### 三、关键实战建议
1. **优先用中间件**：通过 Sharding-JDBC（客户端模式）或 MyCat（代理模式）管理分库分表，避免代码层手动路由；  
2. **分表字段选择**：优先选高频查询、唯一标识的字段（如 `user_id`、`order_no`），避免选低频或非唯一字段；  
3. **平衡一致性与性能**：核心业务用“二级索引表”，非核心用“跨表聚合+缓存”，避免过度设计。
</details>

### 4. MVCC 和间隙锁原理（滴滴/字节/百度）

<details>
<summary>答案与解析</summary> 

#### 一、MVCC（多版本并发控制）原理
MVCC 是 InnoDB 实现“非锁定读”的核心机制，允许“读不阻塞写、写不阻塞读”，解决高并发场景下的性能与一致性矛盾，本质是“通过数据多版本实现并发隔离”。

##### 1. MVCC 核心组成
MVCC 依赖三大组件实现数据版本管理：
- **事务 ID（trx_id）**：每个事务开始时，InnoDB 分配唯一递增的 64 位事务 ID，标记数据版本归属；  
  - 读写事务：写事务（INSERT/UPDATE/DELETE）会生成新 `trx_id`，读事务仅使用已有的 `trx_id`；  
- **undo 日志**：事务修改数据时，先将旧版本数据写入 undo 日志（用于回滚或追溯历史版本）；  
  - 结构：undo 日志按事务 ID 排序，形成“版本链”（最新版本指向旧版本）；  
- **Read View（读视图）**：事务读取数据时生成的“版本过滤规则”，决定当前事务能看到哪些版本的数据。

##### 2. Read View 核心规则
Read View 包含 4 个关键参数，用于判断数据版本的可见性：
- `m_ids`：当前活跃事务的 ID 列表（未提交的事务）；  
- `min_trx_id`：活跃事务中的最小 ID；  
- `max_trx_id`：下一个待分配的事务 ID（即当前最大事务 ID + 1）；  
- `creator_trx_id`：当前事务的 ID。  

数据版本可见性判断逻辑（按顺序执行）：
1. 若数据的 `trx_id == creator_trx_id`：当前事务修改的数据，可见；  
2. 若数据的 `trx_id < min_trx_id`：数据在当前事务开始前已提交，可见；  
3. 若数据的 `trx_id > max_trx_id`：数据在当前事务开始后创建，不可见；  
4. 若 `min_trx_id ≤ trx_id ≤ max_trx_id`：  
   - 若 `trx_id` 在 `m_ids` 中（事务未提交）：不可见；  
   - 若 `trx_id` 不在 `m_ids` 中（事务已提交）：可见。

##### 3. MVCC 工作流程（RR 级别示例）
以“用户修改姓名”为例，展示 MVCC 如何实现可重复读：
1. **初始状态**：`user` 表中 `id=1` 的数据版本为 `trx_id=5`（已提交事务），`name=张三`；  
2. **事务 A（trx_id=10）启动**：执行 `SELECT * FROM user WHERE id=1`，生成 Read View（`m_ids=[10], min=10, max=11`）；  
   - 判断：数据 `trx_id=5 < min=10`，可见，返回 `name=张三`；  
3. **事务 B（trx_id=11）启动并提交**：执行 `UPDATE user SET name='李四' WHERE id=1`；  
   - 操作：将旧版本（`trx_id=5`，`name=张三`）写入 undo 日志，新数据 `trx_id=11`，`name=李四`；  
   - 提交：事务 B 提交，`trx_id=11` 从活跃列表移除；  
4. **事务 A 再次查询**：复用第一次生成的 Read View（`m_ids=[10], min=10, max=11`）；  
   - 判断：新数据 `trx_id=11` 满足 `min≤trx_id≤max`，且 `trx_id=11` 不在 `m_ids` 中，但 Read View 未更新，仍判定为不可见；  
   - 追溯：通过 undo 日志找到旧版本（`trx_id=5`），返回 `name=张三`，实现“可重复读”。


#### 二、间隙锁（Next-Key Lock）原理
间隙锁是 InnoDB 在 RR 隔离级别下为解决“幻读”引入的锁机制，本质是“锁定数据行之间的间隙，防止其他事务插入新行”，确保同一事务内多次查询的行数一致。

##### 1. 幻读的产生场景
幻读是“同一事务内，同一查询条件下，前后查询行数不同”的问题，示例：
1. 事务 A（RR 级别）执行 `SELECT * FROM user WHERE age > 25`，返回 2 行（`age=26`、`age=28`）；  
2. 事务 B 执行 `INSERT INTO user (age) VALUES (30)` 并提交；  
3. 事务 A 再次执行相同查询，返回 3 行（新增 `age=30`），产生幻读。

##### 2. 间隙锁的锁定范围
InnoDB 的间隙锁并非单独存在，而是与“行锁”组合形成 **Next-Key Lock**，锁定范围包括：
- **记录锁（Record Lock）**：锁定当前存在的行（如 `age=26`、`age=28`）；  
- **间隙锁（Gap Lock）**：锁定不存在的行之间的间隙（如 `age 25-26`、`age 28-∞`）。  

以上述幻读场景为例，事务 A 执行 `SELECT * FROM user WHERE age > 25 FOR UPDATE` 时，InnoDB 会锁定：
1. 所有 `age > 25` 的行（记录锁：`age=26`、`age=28`）；  
2. 间隙：`age=25` 到 `age=26`、`age=28` 到 `∞`（间隙锁）；  
其他事务无法在这些间隙插入 `age=27`、`age=30` 等行，从而防止幻读。

##### 3. 间隙锁的触发条件
间隙锁仅在特定条件下触发，并非所有查询都会加间隙锁：
1. **隔离级别**：仅在 RR 隔离级别下触发（RC 级别无间隙锁，无法防幻读）；  
2. **查询条件**：  
   - 范围查询（如 `>`, `<`, `BETWEEN`, `IN`，如 `age BETWEEN 20 AND 30`）；  
   - 等值查询但数据不存在（如 `age=27` 无数据，会锁定 `age=26-28` 间隙）；  
3. **索引依赖**：必须基于“索引列”查询（无索引则退化为表锁，锁定整个表的间隙）。

##### 4. 间隙锁的问题与优化
间隙锁虽能防幻读，但可能导致锁冲突和死锁，需针对性优化：
- **问题 1：死锁**：两个事务交叉锁定不同间隙，导致互相等待；  
  - 示例：事务 A 锁 `age 20-30` 间隙，事务 B 锁 `age 25-35` 间隙，A 需 `25-30` 间隙，B 需 `25-30` 间隙，形成死锁；  
  - 优化：按“固定顺序”加锁（如按 `age` 升序查询），避免交叉锁定。

- **问题 2：锁粒度过大**：范围查询可能锁定大量间隙，影响并发；  
  - 优化：  
    1. 等值查询优先：能用 `age=25` 就不用 `age>25`（等值查询若数据存在，仅加记录锁，不加间隙锁）；  
    2. 用 RC 隔离级别：若业务可接受幻读，改用 RC 级别（无间隙锁，并发更高）；  
    3. 自增主键：表设计用自增主键，减少范围查询的间隙锁范围（如 `id` 自增，范围查询 `id>100` 锁定的间隙更小）。


#### 三、总结
- MVCC 是 InnoDB 高性能的核心：通过多版本和 Read View 实现读写并发，避免锁竞争；  
- 间隙锁是 RR 级别防幻读的关键：代价是可能增加锁冲突，需根据业务权衡“一致性”与“并发性能”——核心业务用 RR+间隙锁，非核心用 RC 提升并发。
</details>

### 5. EXPLAIN 的 type 字段有哪些（知乎）

<details>
<summary>答案与解析</summary> 

#### 一、type 字段的核心意义
`EXPLAIN` 是 MySQL 性能分析的核心工具，`type` 字段表示“MySQL 访问数据的方式”（即“访问类型”），直接反映索引是否有效、查询性能高低，是优化 SQL 的关键依据。

`type` 字段的值按性能从优到差排序，生产环境需至少优化到 `range` 级别，避免 `ALL`（全表扫描）。


#### 二、type 字段的具体类型与场景
##### 1. const（常量查询，性能最优）
- **核心场景**：通过**主键**或**唯一索引**等值查询，最多返回 1 行数据；  
- **原理**：索引键唯一，直接定位到 1 行数据，无需扫描其他行；  
- **示例**：  
  ```sql
  EXPLAIN SELECT * FROM user WHERE id=1001; -- id 是主键
  EXPLAIN SELECT * FROM user WHERE phone='13800138000'; -- phone 是唯一索引
  ```
- **性能**：最优，仅需 1 次磁盘 IO，直接定位数据。

##### 2. eq_ref（唯一索引等值关联，性能次优）
- **核心场景**：多表 JOIN 时，关联字段是**唯一索引**（主键/唯一键），且每次关联仅返回 1 行；  
- **原理**：JOIN 时通过唯一索引精准匹配，无多余行扫描；  
- **示例**：  
  ```sql
  EXPLAIN SELECT o.id FROM orders o 
  JOIN user u ON o.user_id = u.id; -- u.id 是主键（唯一索引）
  ```
- **性能**：次优，比 `const` 多一次关联查询，但仍高效（适合多表 JOIN 核心场景）。

##### 3. ref（非唯一索引等值查询，性能良好）
- **核心场景**：通过**非唯一索引**等值查询，可能返回多行（但远少于全表）；  
- **原理**：索引键不唯一，定位到匹配的行范围，扫描范围远小于全表；  
- **示例**：  
  ```sql
  EXPLAIN SELECT * FROM user WHERE name='张三'; -- name 是普通索引（非唯一）
  ```
- **性能**：良好，适合高频等值查询（如按用户名、邮箱查询）。

##### 4. fulltext（全文索引查询）
- **核心场景**：使用**全文索引**进行关键词匹配（仅支持 `TEXT/VARCHAR` 字段）；  
- **原理**：基于倒排索引，匹配包含关键词的行；  
- **示例**：  
  ```sql
  EXPLAIN SELECT * FROM article WHERE MATCH(content) AGAINST('PHP 索引');
  ```
- **性能**：仅适用于全文检索，效率低于 B+ 树索引（不支持范围查询、排序）。

##### 5. ref_or_null（非唯一索引等值+NULL 查询）
- **核心场景**：非唯一索引查询，且条件包含 `IS NULL`（同时匹配非 NULL 和 NULL 值）；  
- **原理**：在 ref 基础上，额外扫描 NULL 值的索引条目；  
- **示例**：  
  ```sql
  EXPLAIN SELECT * FROM user WHERE name='张三' OR name IS NULL;
  ```
- **性能**：与 `ref` 接近，需额外处理 NULL 值，无明显性能损耗。

##### 6. index_merge（索引合并，性能中等）
- **核心场景**：查询条件使用**多个独立索引**，MySQL 自动合并多个索引的结果（如 `OR` 连接两个索引列）；  
- **原理**：分别扫描每个索引，合并结果（去重），避免全表扫描；  
- **示例**：  
  ```sql
  EXPLAIN SELECT * FROM user WHERE name='张三' OR age=25; -- name 和 age 均为独立索引
  ```
- **性能**：优于全表扫描，但合并结果有额外开销；  
- **优化建议**：优先将多个独立索引优化为**联合索引**（如 `(name, age)`），避免索引合并。

##### 7. unique_subquery（子查询中的唯一索引）
- **核心场景**：子查询中使用**唯一索引**等值查询（如 `IN (SELECT ...)`）；  
- **原理**：子查询返回唯一值，避免返回大量数据导致的性能问题；  
- **示例**：  
  ```sql
  EXPLAIN SELECT * FROM user WHERE id IN (
    SELECT user_id FROM orders WHERE status=1 -- user_id 是唯一索引
  );
  ```
- **性能**：优于 `range`，适合子查询结果唯一的场景。

##### 8. index_subquery（子查询中的非唯一索引）
- **核心场景**：子查询中使用**非唯一索引**查询（子查询可能返回多行）；  
- **示例**：  
  ```sql
  EXPLAIN SELECT * FROM user WHERE name IN (
    SELECT author FROM article WHERE category=1 -- author 是普通索引
  );
  ```
- **性能**：低于 `unique_subquery`，子查询返回行数越多，性能越差（建议用 JOIN 替代子查询）。

##### 9. range（索引范围查询，性能中等）
- **核心场景**：通过索引进行**范围匹配**（如 `>`, `<`, `BETWEEN`, `IN`, `LIKE 'xxx%'`）；  
- **原理**：扫描索引的某个范围（而非全表），范围越小性能越好；  
- **示例**：  
  ```sql
  EXPLAIN SELECT * FROM user WHERE age BETWEEN 20 AND 30; -- age 是索引
  EXPLAIN SELECT * FROM user WHERE name LIKE '张%'; -- 前缀模糊查询（索引有效）
  ```
- **性能**：中等，生产环境最低可接受级别（需避免范围过大）；  
- **优化建议**：缩小查询范围（如加时间条件），避免 `BETWEEN 1 AND 1000000` 这类大范围查询。

##### 10. index（全索引扫描，性能较差）
- **核心场景**：扫描**整个索引树**（而非数据行），通常因查询字段均在索引中（覆盖索引）；  
- **原理**：索引树体积远小于数据行（仅存索引键），扫描速度比全表快，但仍需遍历所有索引条目；  
- **示例**：  
  ```sql
  EXPLAIN SELECT id, name FROM user; -- id 是主键，name 是辅助索引（覆盖索引）
  ```
- **性能**：低于 `range`，高于全表扫描；  
- **注意**：虽比全表好，但仍需优化（如加 WHERE 条件缩小范围）。

##### 11. ALL（全表扫描，性能最差）
- **核心场景**：未使用任何索引，扫描**整个表数据**；  
- **原因**：无索引、索引失效（如函数操作、非最左前缀）、低区分度列（如 `gender='男'`）；  
- **示例**：  
  ```sql
  EXPLAIN SELECT * FROM user WHERE gender='男'; -- gender 无索引
  ```
- **性能**：最差，数据量越大越慢（百万级表可能耗时几秒到几分钟）；  
- **优化建议**：加索引、调整查询条件（避免索引失效）、拆分表（分库分表）。


#### 三、实战分析建议
1. **核心优化目标**：生产环境中，`type` 应至少优化到 `range` 及以上，避免 `ALL` 和 `index`；  
2. **结合其他字段判断**：  
   - `key` 字段：确认实际使用的索引（`key` 为 `NULL` 表示无索引）；  
   - `rows` 字段：预估扫描行数（`rows` 越小性能越好，需与实际数据量对比）；  
   - `Extra` 字段：关注 `Using index`（覆盖索引，无回表）、`Using filesort`（文件排序，需优化）、`Using temporary`（临时表，需优化）；  
3. **常见优化手段**：  
   - 加索引（针对查询条件）；  
   - 调整查询条件（避免函数操作、非最左前缀）；  
   - 用 JOIN 替代子查询（减少 `index_subquery`）；  
   - 分库分表（解决 `ALL` 场景下的数据量过大问题）。
</details>

### 6. UPDATE 语句执行流程，binlog 的作用与几种格式（滴滴）

<details>
<summary>答案与解析</summary> 
MySQL UPDATE 语句执行流程（InnoDB 引擎）
UPDATE 语句从客户端发起请求到数据落地，涉及“连接器、分析器、优化器、执行器、存储引擎”五大模块，共 8 个核心步骤，需重点关注 WAL（Write-Ahead Logging）机制和锁操作：

##### 1. 连接器：建立连接与权限验证
- 客户端与 MySQL 建立 TCP 连接，连接器验证账号密码（如用户名、密码、IP 权限）；  
- 验证通过后，分配独立的连接线程，维持连接状态（直到超时或断开）。

##### 2. 分析器：SQL 解析与语法校验
- **词法分析**：识别 SQL 关键字（如 `UPDATE`）、表名（如 `user`）、字段名（如 `name`）、条件（如 `id=1001`）；  
- **语法分析**：检查 SQL 格式是否正确（如是否缺少 `WHERE`、字段名是否存在），若语法错误则直接返回错误。

##### 3. 优化器：选择最优执行计划
- 优化器根据表结构（如索引）选择最优执行路径，核心是“是否使用索引”；  
- 示例：`UPDATE user SET name='李四' WHERE id=1001` 中，优化器会选择主键索引（`id`），而非全表扫描；  
- 若存在多个索引（如 `id` 和 `phone`），优化器会计算“成本”（如扫描行数、IO 次数），选择成本最低的方案。

##### 4. 执行器：调用存储引擎接口执行更新
执行器按优化器的计划，调用 InnoDB 接口执行更新，核心步骤：
- **步骤 4.1：加排他锁（X 锁）**：InnoDB 通过索引找到 `id=1001` 的行，加排他锁（防止其他事务同时修改或删除该行）；  
- **步骤 4.2：写入 undo 日志**：将该行的旧版本数据（如 `name=张三`）写入 undo 日志，用于事务回滚或 MVCC 版本追溯；  
- **步骤 4.3：修改内存数据**：在 InnoDB 缓冲池（Buffer Pool）中修改数据（如 `name` 改为“李四”），标记该数据页为“脏页”（内存与磁盘数据不一致）。

##### 5. 写入 redo 日志（WAL 机制核心）
- 按 WAL 原则（“先写日志，再写磁盘”），将“数据修改后的新值”写入 redo 日志（存储引擎层日志）；  
- redo 日志记录“物理修改”（如“修改 `user` 表 `id=1001` 行的 `name` 字段为‘李四’”），确保 MySQL 崩溃后可恢复未刷盘的脏页数据。

##### 6. 写入 binlog（二进制日志）
- 事务提交时，MySQL 服务层将 UPDATE 操作写入 binlog（二进制日志）；  
- binlog 记录“逻辑修改”（如 `UPDATE user SET name='李四' WHERE id=1001`），用于主从同步和数据恢复；  
- **写入时机**：仅在事务提交时写入（`sync_binlog=1` 时同步刷盘，确保 binlog 不丢失）。

##### 7. 事务提交与释放锁
- 执行 `COMMIT` 命令，InnoDB 标记事务为“已提交”，释放之前加的排他锁（X 锁）；  
- 若执行 `ROLLBACK`，则通过 undo 日志恢复数据，释放锁。

##### 8. 异步刷盘（脏页写入磁盘）
- 后台线程（如 Page Cleaner 线程）异步将缓冲池中的脏页写入磁盘（避免阻塞事务提交）；  
- 刷盘时机：缓冲池满、redo 日志满、定时刷盘（如每 10 秒）等，确保数据最终持久化。


#### 二、binlog（二进制日志）的核心作用
binlog 是 MySQL 服务层生成的日志（所有引擎均支持），记录所有“数据变更操作”（INSERT/UPDATE/DELETE/DDL），不记录查询操作（SELECT），核心作用有三：

##### 1. 主从同步
- 主库将 binlog 发送给从库，从库通过 IO 线程接收 binlog 并写入中继日志（relay log）；  
- 从库 SQL 线程读取中继日志，执行 binlog 中的操作，实现主从数据一致（如主库执行 UPDATE 后，从库同步执行相同 UPDATE）。

##### 2. 数据恢复
- 通过 binlog 恢复指定时间点或指定操作的数据；  
- 示例：误删表后，先通过全量备份恢复到某个时间点，再通过 `mysqlbinlog` 工具解析 binlog，重放备份后到误删前的操作，恢复数据。

##### 3. 审计日志
- 记录所有数据变更操作，包含操作时间、账号、SQL 语句，便于追溯问题（如定位“谁修改了某条订单数据”）。


#### 三、binlog 的三种格式及区别
binlog 格式通过 `binlog_format` 配置（可在 `my.cnf` 或会话级别设置），三种格式各有优缺点，需根据场景选择：

##### 1. STATEMENT（语句模式）
- **记录内容**：直接记录执行的 SQL 语句（如 `UPDATE user SET name='李四' WHERE id=1001`）；  
- **优势**：  
  - 日志体积小（仅存 SQL 语句，无数据行细节）；  
  - 写入性能高（无需记录每行数据的变更）；  
- **劣势**：  
  - 不支持非确定性函数（如 `NOW()`、`RAND()`、`UUID()`），主从执行结果可能不一致；  
    - 示例：主库执行 `UPDATE user SET update_time=NOW() WHERE id=1001` 时，`NOW()` 为 10:00，从库执行时可能为 10:01，导致数据不一致；  
  - 不支持存储过程、自定义函数的精确同步；  
- **适用场景**：无非确定性函数、存储过程的简单业务，对日志体积敏感的场景。

##### 2. ROW（行模式，InnoDB 推荐）
- **记录内容**：不记录 SQL 语句，仅记录“数据行的变更前后状态”（如 `id=1001` 的 `name` 从“张三”改为“李四”）；  
- **优势**：  
  - 主从数据绝对一致（不依赖 SQL 执行逻辑，直接记录数据变更）；  
  - 支持所有函数、存储过程、自定义函数；  
  - 可精准恢复单条数据（如误更新某行，可通过 binlog 恢复该行旧值）；  
- **劣势**：  
  - 日志体积大（如更新 1000 行数据，需记录 1000 行的变更细节）；  
- **优化手段**：通过 `binlog_row_image=MINIMAL` 仅记录“变更的字段”（而非整行数据），减少日志体积；  
- **适用场景**：主从同步、数据恢复要求高的场景（如金融、电商核心业务），是 InnoDB 默认格式。

##### 3. MIXED（混合模式）
- **记录逻辑**：自动切换 STATEMENT 和 ROW 模式——  
  - 简单 SQL（如无函数的等值更新）用 STATEMENT；  
  - 复杂 SQL（如含 `NOW()`、批量更新、存储过程）用 ROW；  
- **优势**：平衡日志体积和一致性；  
- **劣势**：行为不可预测（需通过 `binlog_format` 确认当前模式），排查问题时复杂度高；  
- **适用场景**：过渡场景（如从 STATEMENT 迁移到 ROW 的中间阶段），目前逐渐被 ROW 模式替代。


#### 四、关键补充
- binlog 与 redo 日志的区别：redo 是存储引擎层日志（仅 InnoDB 支持），记录物理修改，用于崩溃恢复；binlog 是服务层日志（所有引擎支持），记录逻辑修改，用于主从同步和数据恢复；  
- 生产环境配置建议：`binlog_format=ROW` + `sync_binlog=1`（同步刷盘，确保 binlog 不丢失） + `binlog_row_image=MINIMAL`（减少日志体积）。
</details>

### 7. 主从同步的原理与常见问题（字节/滴滴/陌陌）

<details>
<summary>答案与解析</summary> 

#### 一、MySQL 主从同步核心原理（基于 binlog 的异步复制）
MySQL 主从同步通过“三个线程+binlog”实现，主库（Master）负责生成 binlog，从库（Slave）负责接收和执行 binlog，核心是“异步复制”（主库不等待从库执行完成），流程如下：

##### 1. 主库：Binlog Dump 线程
- **作用**：监听主库 binlog 变化，向从库发送 binlog 数据；  
- **工作流程**：  
  1. 主库开启 binlog（`log_bin=ON`），数据变更（INSERT/UPDATE/DELETE）写入 binlog；  
  2. 从库连接主库时，主库创建 Binlog Dump 线程（每个从库对应一个线程）；  
  3. Binlog Dump 线程读取主库 binlog 的最新内容，实时发送给从库的 IO 线程。

##### 2. 从库：IO 线程
- **作用**：接收主库发送的 binlog，写入从库本地的“中继日志（relay log）”；  
- **工作流程**：  
  1. 从库执行 `CHANGE MASTER TO` 命令，配置主库信息（主库 IP、端口、复制账号、binlog 文件名和位置）；  
  2. 从库启动 IO 线程，连接主库的 Binlog Dump 线程；  
  3. IO 线程接收 binlog 数据，按顺序写入中继日志（relay log 格式与 binlog 一致，避免直接修改 binlog）。

##### 3. 从库：SQL 线程
- **作用**：读取中继日志，执行其中的 SQL 操作，将从库数据更新为与主库一致；  
- **工作流程**：  
  1. 从库启动 SQL 线程，按顺序读取中继日志中的 SQL 语句；  
  2. 执行 SQL 操作（如同步 UPDATE、INSERT），更新从库数据；  
  3. 执行完成后，删除已处理的中继日志（通过 `relay_log_purge=ON` 自动清理，避免磁盘占用）。

##### 4. 核心流程总结
主库数据变更 → 写入 binlog → Binlog Dump 线程发送 binlog → 从库 IO 线程接收并写入中继日志 → 从库 SQL 线程执行中继日志 → 从库数据与主库一致。


#### 二、主从同步的常见问题与解决方案
主从同步的核心痛点是“延迟”“数据不一致”“线程停止”“主库宕机切换”，需针对性解决：

##### 1. 问题 1：主从延迟（最高频问题）
- **定义**：从库数据比主库晚更新的时间（通过 `SHOW SLAVE STATUS` 的 `Seconds_Behind_Master` 查看，单位秒）；  
- **核心原因**：  
  1. **主库端**：  
     - 大事务（如批量更新 10 万行数据）：主库写入 binlog 慢，从库执行更慢；  
     - 高并发写入：主库 binlog 生成速度超过从库接收/执行速度；  
  2. **从库端**：  
     - 配置低：CPU/内存不足，SQL 线程执行慢；  
     - 复杂 SQL：从库执行 JOIN、子查询等复杂 SQL 耗时；  
     - 单线程执行：MySQL 5.6 及以前，SQL 线程仅 1 个，无法并行处理 binlog；  
- **解决方案**：  
  - **主库优化**：  
    1. 拆分大事务：将批量更新拆为小批量（如 1000 行/批），避免长时间占用 binlog；  
    2. 避免长事务：事务仅包含必要 SQL，不包含外部接口调用、睡眠等操作；  
  - **从库优化**：  
    1. 提升配置：给从库分配更高 CPU/内存（如主库 8C16G，从库至少 4C8G）；  
    2. 开启并行复制（MySQL 5.7+）：配置 `slave_parallel_workers=4`（4 个 SQL 线程并行执行中继日志），按“事务组”或“Schema”并行；  
    3. 从库只读优化：关闭从库 binlog（`log_bin=OFF`，从库无需生成 binlog）、禁用从库查询缓存（`query_cache_type=OFF`）；  
  - **架构优化**：  
    1. 延迟敏感业务读主库：如支付结果查询、实时库存查询，直接读主库；  
    2. 非敏感业务读从库：如商品列表、历史订单，读从库；  
    3. 级联复制：主库 → 中间从库 → 从库，减少主库的 Binlog Dump 线程压力（适合从库数量多的场景）；  
  - **监控告警**：`Seconds_Behind_Master` 超过阈值（如 30 秒）时告警，及时排查。

##### 2. 问题 2：主从数据不一致
- **定义**：主从库数据存在差异（如主库某行 `name=李四`，从库为 `name=张三`）；  
- **核心原因**：  
  1. 主库 binlog 格式为 STATEMENT：含非确定性函数（如 `NOW()`、`RAND()`），主从执行结果不一致；  
  2. 从库中途停止同步：如从库宕机，重启后部分 binlog 未执行；  
  3. 手动修改从库数据：如直接在从库执行 UPDATE/DELETE，破坏一致性；  
  4. 主从库配置不一致：如 `sql_mode` 不同（主库严格模式，从库非严格模式），导致从库执行 SQL 不报错但数据不一致；  
- **解决方案**：  
  - **预防措施**：  
    1. 统一 binlog 格式为 ROW：避免函数导致的不一致；  
    2. 从库设为只读：`read_only=1`（仅允许超级管理员操作，禁止普通用户修改）；  
    3. 主从配置一致：`sql_mode`、`character_set`、`collation` 等配置保持相同；  
  - **修复手段**：  
    1. 轻量不一致（少量数据）：用 `pt-table-checksum`（Percona 工具）检测不一致，`pt-table-sync` 同步数据；  
      - 示例：`pt-table-checksum h=主库IP,u=root,p=密码,D=test,t=user` 检测 `test.user` 表一致性；  
    2. 重度不一致（大量数据）：重新初始化从库；  
      - 步骤：主库导出全量数据（`mysqldump`）→ 从库导入数据 → 从库重新配置主从同步（`CHANGE MASTER TO`）；  

##### 3. 问题 3：从库 IO 线程/SQL 线程停止
- **现象**：`SHOW SLAVE STATUS` 中 `Slave_IO_Running` 或 `Slave_SQL_Running` 为 `No`；  
- **核心原因**：  
  1. **IO 线程停止**：  
     - 主库复制账号权限不足：无 `REPLICATION SLAVE` 权限；  
     - 主库 binlog 文件损坏或丢失：从库无法找到指定的 binlog 文件（如主库删除了已发送但从库未执行的 binlog）；  
     - 网络问题：主从库网络中断、防火墙拦截 3306 端口；  
  2. **SQL 线程停止**：  
     - 从库执行 SQL 报错：如主库有某表，从库无该表（DDL 未同步）、主键冲突（从库已有相同主键数据）；  
     - 中继日志损坏：从库无法解析中继日志；  
- **解决方案**：  
  - **IO 线程修复**：  
    1. 检查权限：主库执行 `GRANT REPLICATION SLAVE ON *.* TO 'repl'@'从库IP' IDENTIFIED BY '密码';`；  
    2. 检查 binlog：主库执行 `SHOW BINARY LOGS` 确认 binlog 存在，从库重新配置 `MASTER_LOG_FILE` 和 `MASTER_LOG_POS`；  
    3. 检查网络：`ping` 主库 IP、`telnet 主库IP 3306` 确认网络通畅；  
  - **SQL 线程修复**：  
    1. 查看错误日志：从库 `error.log` 定位报错原因（如“Table 'test.user' doesn't exist”）；  
    2. 修复报错：如从库缺少表则创建表，主键冲突则删除冲突数据；  
    3. 重启同步：`STOP SLAVE; RESET SLAVE; START SLAVE;`（`RESET SLAVE` 清空中继日志，重新从主库获取 binlog）；  

##### 4. 问题 4：主库宕机后从库切换
- **痛点**：主库宕机后，需将从库提升为主库，但可能存在“主库未同步的 binlog”（如主库宕机前未将 binlog 发送给从库），导致数据丢失；  
- **解决方案**：  
  1. 开启半同步复制（MySQL 5.5+）：  
     - 主库配置：`plugin-load-add=rpl_semi_sync_master.so` + `rpl_semi_sync_master_enabled=1` + `rpl_semi_sync_master_timeout=1000`（超时 1 秒退化为异步）；  
     - 从库配置：`plugin-load-add=rpl_semi_sync_slave.so` + `rpl_semi_sync_slave_enabled=1`；  
     - 原理：主库事务提交前，需等待至少一个从库的 IO 线程确认接收 binlog，再返回事务成功，减少数据丢失风险；  
  2. 自动切换工具：  
     - 用 MGR（MySQL Group Replication）：多主模式，主库宕机后自动选举新主库，无需手动干预；  
     - 用 Keepalived+VIP：主从库绑定同一 VIP（虚拟 IP），主库宕机后 Keepalived 自动将 VIP 切换到从库，应用无感知；  
  3. 手动切换步骤（无工具时）：  
     1. 从库执行 `STOP SLAVE IO_THREAD;`（停止接收 binlog，避免主库恢复后冲突）；  
     2. 确认从库数据：`SHOW SLAVE STATUS` 查看 `Seconds_Behind_Master` 为 0（数据已同步）；  
     3. 提升从库为主库：`STOP SLAVE; RESET MASTER;`（清空从库 binlog，作为新主库）；  
     4. 应用切换：将应用的数据库地址改为新主库 IP 或 VIP；  


#### 三、主从同步架构优化（进阶）
1. **一主多从**：1 个主库写入，多个从库分担读压力（如 1 主 3 从），通过读写分离中间件（如 MyCat、Sharding-JDBC）分发读请求到从库；  
2. **GTID 复制（MySQL 5.6+）**：  
   - 用全局事务 ID（GTID）替代“binlog 文件名+位置”，每个事务对应唯一 GTID；  
   - 优势：主从切换时无需手动指定 `MASTER_LOG_FILE` 和 `MASTER_LOG_POS`，简化操作；  
3. **延迟从库**：配置 1 个延迟从库（如延迟 1 小时），用于恢复误操作（如误删表后，从延迟从库恢复数据）。


#### 四、总结
主从同步是 MySQL 高可用的基础，核心是“三个线程+binlog”，常见问题需从“主库写入”“从库执行”“架构设计”三方面优化——主库拆分大事务，从库开启并行复制，架构层面用半同步+自动切换工具，确保同步稳定、数据一致。
</details>

### 8. 死锁产生原因以及如何解决（滴滴/顺丰）
 
 <details>
 <summary>答案与解析</summary> 
 
 #### 一、死锁的本质与常见触发原因
 死锁的本质是“循环等待”：事务 A 持有锁 1 并等待锁 2，事务 B 持有锁 2 并等待锁 1，彼此相互等待，MySQL InnoDB 会检测到死锁并主动回滚代价最小的事务（报错 1213）。常见触发：
 1. 加锁顺序不一致：不同事务按不同顺序更新相同集合的行/索引。
 2. 范围查询 + 间隙锁：RR 隔离级别下 range 条件触发 next-key lock，多个范围交叉导致互等。
 3. 外键/级联更新：父子表更新时涉及隐藏索引与额外加锁，顺序不一致易死锁。
 4. 无合适索引导致“锁范围扩大”：走全表或错误索引，形成大范围行锁/间隙锁，冲突概率上升。
 5. 大事务：持锁时间长，竞争窗口扩大。
 
 #### 二、如何定位死锁
 - SHOW ENGINE INNODB STATUS;（或 performance_schema.events_transactions_current/history）查看最近死锁的锁资源、SQL、锁类型（Record/GAP/Next-Key）。
 - 打开 innodb_print_all_deadlocks（8.0 用 performance_schema）。
 
 #### 三、系统性解决方案
 1. 统一加锁顺序（最重要）：
    - 业务上约定固定顺序，如总是先按 id 小到大更新：SELECT ... FOR UPDATE 时 ORDER BY id ASC。
 2. 减少范围锁：
    - 尽量使用等值命中唯一/主键索引，避免大范围 range 条件；必要时切换到 RC（可接受幻读）。
 3. 索引正确性：
    - WHERE 条件的列必须有合适索引，避免退化为全表锁定；JOIN/外键列务必建索引。
 4. 将大事务拆小：
    - 每次更新少量行，缩短持锁时间，失败后易于重试。
 5. 使用“尝试-重试”机制：
    - 捕获 ER_LOCK_DEADLOCK(1213)/ER_LOCK_WAIT_TIMEOUT(1205) 并指数退避重试。
 6. 避免“先查后改”的竞态：
    - 直接使用 UPDATE ... WHERE ... LIMIT ... + 索引，或 SELECT ... FOR UPDATE 锁定后再改。
 
 #### 四、示例：两条更新顺序不一致导致死锁
 ```sql
 -- 表与索引
 CREATE TABLE product (
   id BIGINT PRIMARY KEY,
   stock INT NOT NULL,
   price INT NOT NULL,
   KEY idx_price (price)
 ) ENGINE=InnoDB;
 
 -- 事务 A（会话1）
 START TRANSACTION;
 -- 命中 idx_price 先锁 price=100 的行范围
 SELECT id FROM product WHERE price=100 FOR UPDATE; 
 -- 后续再锁 id=2 的聚簇记录
 UPDATE product SET stock=stock-1 WHERE id=2;
 
 -- 事务 B（会话2）
 START TRANSACTION;
 -- 先锁 id=2 的聚簇记录
 UPDATE product SET stock=stock-1 WHERE id=2;
 -- 再去锁 price=100 范围
 SELECT id FROM product WHERE price=100 FOR UPDATE; 
 -- 两边锁集合交叉，极易死锁
 ```
 
 #### 五、代码层面：PHP（PDO）死锁重试模板
 ```php
 <?php
 function withRetry(callable $fn, int $maxRetry = 3) {
     $attempt = 0;
     while (true) {
         try {
             return $fn();
         } catch (PDOException $e) {
             $code = isset($e->errorInfo[1]) ? (int)$e->errorInfo[1] : 0; // MySQL 错误码
             if (($code === 1213 || $code === 1205) && $attempt < $maxRetry) {
                 usleep((int) (100000 * pow(2, $attempt))); // 指数退避: 100ms, 200ms, 400ms
                 $attempt++;
                 continue; // 重试
             }
             throw $e; // 非死锁/超时 或 重试耗尽
         }
     }
 }
 
 // 使用示例：按固定顺序（id 升序）锁定后扣减库存，降低死锁概率
 function decrementStocks(PDO $pdo, array $ids): void {
     sort($ids, SORT_NUMERIC); // 固定顺序
     withRetry(function () use ($pdo, $ids) {
         $pdo->beginTransaction();
         // 显式选择合适隔离级别（可选）：RR 或 RC
         // $pdo->exec("SET TRANSACTION ISOLATION LEVEL REPEATABLE READ");
 
         // 先锁定将要操作的行集合
         $in = implode(',', array_fill(0, count($ids), '?'));
         $stmt = $pdo->prepare("SELECT id FROM product WHERE id IN ($in) FOR UPDATE");
         $stmt->execute($ids);
 
         // 扣减库存（确保有索引，使用等值命中）
         $u = $pdo->prepare("UPDATE product SET stock = stock - 1 WHERE id = ? AND stock > 0");
         foreach ($ids as $id) {
             $u->execute([$id]);
         }
 
         $pdo->commit();
         return true;
     });
 }
 ?>
 ```
 
 #### 六、补充：参数与监控建议
 - innodb_lock_wait_timeout：锁等待超时时间，过短会增加重试次数，过长影响吞吐（常用 5-20s）。
 - 监控“死锁次数/锁等待时间/回滚数”，与热点行比例、事务耗时关联分析。
 </details>

### 9. 如何优化大 OFFSET（陌陌）

<details>
<summary>答案与解析</summary> 

#### 一、为什么大 OFFSET 很慢
- OFFSET 的实现是“跳过 N 行再取 M 行”，即使有索引，也要扫描并丢弃前 N 行；N 越大，扫描越多，CPU/IO 越高；
- 常见表现：`LIMIT 100000, 20` 明显变慢，`EXPLAIN` 出现 `Using filesort`、扫描行数巨大。

#### 二、核心思路：用“定位”替代“跳过”
1) Keyset 分页（基于游标/锚点，强烈推荐）
- 思路：记录上一页最后一条记录的排序键，下一页用 `WHERE (sort_key, id) < (?, ?)` 继续；
- 要求：稳定排序且有匹配的联合索引（如 `(status, create_time DESC, id DESC)`）。

2) 延迟关联（先取主键，后回表）
- 思路：先用覆盖索引拿到一页主键，再回表拿完整列，避免大行回表参与排序；
- 适用：单行较“胖”、选择列较多的场景。

3) 限制深分页 & 改“页码”为“下一页”
- 业务限制最大页数（如 ≤ 100 页），鼓励按时间/ID跳转；
- 前端用“下一页游标”而非“第 N 页”。

4) 缩小候选集
- 将过滤条件放入索引前缀（如 `(status, create_time DESC, id DESC)`），减少扫描范围。

5) 近似总数
- 大表总数用异步统计/缓存/估算，避免阻塞式 `COUNT(*)`。

#### 三、SQL 示例
```sql
-- 表结构与索引
CREATE TABLE post (
  id BIGINT PRIMARY KEY,
  title VARCHAR(200),
  author_id BIGINT,
  status TINYINT,
  create_time DATETIME,
  KEY idx_status_ct_id (status, create_time DESC, id DESC)
) ENGINE=InnoDB;

-- 反例：深 OFFSET（慢）
EXPLAIN SELECT * FROM post WHERE status=1
ORDER BY create_time DESC, id DESC
LIMIT 100000, 20;

-- 方案1：Keyset 分页
-- 假设上一页最后一条 (create_time, id) = ('2025-01-01 10:00:00', 888888)
SELECT id, title, author_id, status, create_time FROM post
WHERE status=1
  AND (create_time < '2025-01-01 10:00:00'
       OR (create_time = '2025-01-01 10:00:00' AND id < 888888))
ORDER BY create_time DESC, id DESC
LIMIT 20; -- 命中 idx_status_ct_id

-- 方案2：延迟关联
SELECT p.* FROM post p
JOIN (
  SELECT id FROM post
  WHERE status=1
  ORDER BY create_time DESC, id DESC
  LIMIT 20
) t ON p.id = t.id
ORDER BY p.create_time DESC, p.id DESC;
```

#### 四、PHP 游标分页示例
```php
<?php
function encodeCursor(DateTime $ts, int $id): string {
    return rtrim(strtr(base64_encode($ts->format('Y-m-d H:i:s') . '|' . $id), '+/', '-_'), '=');
}
function decodeCursor(string $cursor): array {
    $raw = base64_decode(strtr($cursor, '-_', '+/'));
    [$ts, $id] = explode('|', $raw, 2);
    return [new DateTime($ts), (int)$id];
}

function listPostsFirst(PDO $pdo, int $limit = 20): array {
    $sql = "SELECT id, title, author_id, status, create_time
            FROM post WHERE status=1
            ORDER BY create_time DESC, id DESC
            LIMIT :lim";
    $stmt = $pdo->prepare($sql);
    $stmt->bindValue(':lim', $limit, PDO::PARAM_INT);
    $stmt->execute();
    $rows = $stmt->fetchAll(PDO::FETCH_ASSOC);
    if (!$rows) return ['items' => [], 'next' => null];
    $last = end($rows);
    $next = encodeCursor(new DateTime($last['create_time']), (int)$last['id']);
    return ['items' => $rows, 'next' => $next];
}

function listPostsNext(PDO $pdo, string $cursor, int $limit = 20): array {
    [$ts, $id] = decodeCursor($cursor);
    $sql = "SELECT id, title, author_id, status, create_time
            FROM post WHERE status=1
              AND (create_time < :ts OR (create_time = :ts AND id < :id))
            ORDER BY create_time DESC, id DESC
            LIMIT :lim";
    $stmt = $pdo->prepare($sql);
    $stmt->bindValue(':ts', $ts->format('Y-m-d H:i:s'));
    $stmt->bindValue(':id', $id, PDO::PARAM_INT);
    $stmt->bindValue(':lim', $limit, PDO::PARAM_INT);
    $stmt->execute();
    $rows = $stmt->fetchAll(PDO::FETCH_ASSOC);
    if (!$rows) return ['items' => [], 'next' => null];
    $last = end($rows);
    $next = encodeCursor(new DateTime($last['create_time']), (int)$last['id']);
    return ['items' => $rows, 'next' => $next];
}
?>
```

#### 五、注意事项
- 索引必须与排序一致（如 `(status, create_time DESC, id DESC)`），否则会 `Using filesort`；
- 锚点字段需“单调+唯一性兜底”（时间+主键）；
- 需要上一页时可维护游标栈，或升序查询后反转结果。
</details>

---

## Redis

### 1. 缓存如何保证一致性（每家都问）

<details>
<summary>答案与解析</summary> 

#### 一、问题与目标
- 读取：尽量命中缓存，缓存未命中则读库并回填；
- 更新：以数据库为准，避免脏缓存与穿透/击穿；
- 目标：最终一致或在可接受窗口内接近强一致。

#### 二、常见策略
1. Cache-Aside（旁路缓存，最常用）
   - 读：先查缓存，miss 则查库，写入缓存并设置 TTL；
   - 写：先写库，再删缓存（而不是更新缓存）。若高并发下仍有脏读，可采用“双删”策略（提交后删除一次，延迟 200~500ms 再删一次，配合异步重试）。
2. Read-Through / Write-Through / Write-Back
   - 依赖代理/中间件将读写透传到存储；PHP 直连 MySQL 时较少使用。
3. 订阅 binlog 异步修复
   - 通过 Canal/Debezium 订阅主库 binlog，异步删除或更新相关 key，保证最终一致；
   - 适合读多写多、跨语言服务的“中心化修复”。
4. 逻辑过期 + 后台刷新（stale-while-revalidate）
   - 值过期后先返回旧值，同时异步刷新，降低抖动。
5. 热点 Key 保护
   - 加互斥锁生成缓存，避免“缓存击穿”；对不存在数据可缓存空值+短 TTL，避免穿透。

#### 三、PHP 实战模板
```php
<?php
function redis(): Redis {
    static $r = null;
    if ($r) return $r;
    $r = new Redis();
    $r->connect('127.0.0.1', 6379, 1.0);
    $r->setOption(Redis::OPT_SERIALIZER, Redis::SERIALIZER_JSON);
    return $r;
}

function getUser(PDO $pdo, int $id): ?array {
    $key = "user:$id";
    $r = redis();
    $val = $r->get($key);
    if ($val !== false) return $val; // 命中

    // 互斥保护避免击穿
    $lockKey = "lock:$key";
    $token = bin2hex(random_bytes(8));
    $locked = $r->set($lockKey, $token, ['NX', 'PX' => 3000]);
    if (!$locked) {
        usleep(50_000); // 50ms 退避
        return getUser($pdo, $id);
    }

    try {
        // 双查缓存，减少重建风暴
        $val = $r->get($key);
        if ($val !== false) return $val;

        $stmt = $pdo->prepare('SELECT id, name, email FROM user WHERE id = ?');
        $stmt->execute([$id]);
        $row = $stmt->fetch(PDO::FETCH_ASSOC);
        if (!$row) {
            // 缓存空值短 TTL，防穿透
            $r->setex($key, 60, null);
            return null;
        }
        $r->setex($key, 3600, $row); // TTL 视业务设定
        return $row;
    } finally {
        // 安全释放锁
        $lua = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
        $r->eval($lua, [$lockKey, $token], 1);
    }
}

function updateUser(PDO $pdo, int $id, array $data): void {
    $pdo->beginTransaction();
    try {
        // 写库
        $stmt = $pdo->prepare('UPDATE user SET name = ?, email = ? WHERE id = ?');
        $stmt->execute([$data['name'], $data['email'], $id]);
        $pdo->commit();
    } catch (Throwable $e) {
        $pdo->rollBack();
        throw $e;
    }
    // 提交后删缓存（而非更新），保证下一次读取走库回填
    $key = "user:$id";
    $r = redis();
    $r->del($key);
    // 双删兜底（可选）
    usleep(200_000); // 200ms
    $r->del($key);
}
?>
```

#### 四、要点
- 删除优于更新：写库后删缓存，避免并发时序导致脏数据；
- 设置合理 TTL 与热点保护；
- 关键路径失败要有重试/补偿（消息队列或 binlog 订阅修复）。
</details>

### 2. 用过 Redis 哪些数据结构，使用场景是什么（每家都问）

<details>
<summary>答案与解析</summary> 

#### 一、核心数据结构与典型场景
1) String（字符串/二进制安全）
- 场景：配置、计数器（INCR）、分布式锁值、Token/JWT、简单 KV；
- 扩展：位图（SETBIT/GETBIT）用于活跃用户打点；Bitfield 用于压缩计数；
- 复杂度：GET/SET/INCR O(1)。

2) Hash（哈希/对象）
- 场景：用户画像、对象字段（user:1001 => name/email/age）；
- 优点：按字段读写，节省内存；
- 注意：大哈希慎用 HGETALL（可能很大），推荐精确字段或分页。

3) List（列表）
- 场景：消息队列（LPUSH/BRPOP）、任务队列、时间线简单实现；
- 注意：避免超长列表导致慢操作，必要时分桶或使用 Stream。

4) Set（集合）
- 场景：去重、标签、共同好友（SINTER）、抽奖池（SRANDMEMBER）；
- 复杂度：SADD/SMEMBERS/SISMEMBER；交并差适合中小集合。

5) Sorted Set（有序集合）
- 场景：排行榜（score=分数/权重）、延时队列（score=时间戳）、近实时推荐/Feed；
- 特点：按 score 排序，支持范围/排名查询；底层跳表+哈希表。

6) HyperLogLog（基数估算）
- 场景：UV 统计、去重估算；
- 特点：内存固定 ~12KB，误差约 0.81%，只支持估算基数，不存明细。

7) Bitmap / Bitfield
- 场景：每日活跃、签到（位图），多个小整数打包存储（Bitfield）。

8) Geo（地理位置）
- 场景：附近的人/店，范围搜索（GEOSEARCH/GEOSEARCHSTORE）。

9) Stream（日志流，消费者组）
- 场景：可靠消息、事件日志、多人消费（XADD/XREADGROUP/ACK）。

#### 二、PHP 常用操作示例（phpredis 扩展）
```php
<?php
$r = new Redis();
$r->connect('127.0.0.1', 6379);

// 1) String 计数器
$key = 'pv:2025-09-11';
$r->incrBy($key, 1);          // 原子自增
$r->expire($key, 86400);

// 2) Hash 用户画像
$r->hMSet('user:1001', ['name' => 'Alice', 'email' => 'a@x.com']);
$name = $r->hGet('user:1001', 'name');

// 3) List 简单队列（生产/消费）
$r->lPush('queue:task', json_encode(['id' => 1]));
$job = $r->brPop(['queue:task'], 5); // [key, value]

// 4) Set 去重 + 交集
$r->sAdd('tags:1', 'php', 'mysql');
$r->sAdd('tags:2', 'php', 'redis');
$common = $r->sInter(['tags:1', 'tags:2']); // ['php']

// 5) ZSet 排行榜
$r->zAdd('rank:score', 98.5, 'u1001');
$r->zAdd('rank:score', 99.0, 'u1002');
$top = $r->zRevRange('rank:score', 0, 9, true); // 前10名含分数

// 6) HyperLogLog UV 统计
$r->pfAdd('uv:2025-09-11', ['u1','u2','u3']);
$uv = $r->pfCount('uv:2025-09-11');

// 7) Bitmap 签到
$r->setBit('checkin:2025-09', 10, 1); // 10号签到
$day10 = $r->getBit('checkin:2025-09', 10);

// 8) GEO 附近搜索
$r->geoAdd('shop:geo', 116.397, 39.908, 'tiananmen');
$near = $r->rawCommand('GEOSEARCH', 'shop:geo', 'FROMLONLAT', 116.40, 39.90, 'BYRADIUS', 5, 'km', 'WITHDIST', 'COUNT', 10);

// 9) Stream 消费者组
$r->xAdd('order:stream', '*', ['orderId' => 'O123']);
$r->xGroup('CREATE', 'order:stream', 'g1', '$');
$msgs = $r->xReadGroup('g1', 'c1', ['order:stream' => '>'], 10, 2000);
?>
```

#### 三、选择建议
- 数据量大且需范围排序 => ZSet；
- 仅去重集合运算 => Set；
- 简单队列、低可靠 => List；多消费者、可确认 => Stream；
- UV/去重估算 => HyperLogLog；
- 活跃/签到位级存储 => Bitmap；
- 地理范围检索 => Geo。
</details>

### 3. redis 的 connect 和 pconnect 的区别，pconnect 有什么问题（滴滴/陌陌）

<details>
<summary>答案与解析</summary> 

#### 一、区别
- connect：短连接，每次请求建立/关闭 TCP 连接；
- pconnect：持久连接，连接在 PHP-FPM 进程生命周期内复用，不随请求结束而断开；通过 `persistent_id` 区分不同池。

#### 二、优缺点
- 优点（pconnect）：
  - 省去 TCP 三次握手与认证开销，高 QPS 下更稳定；
- 风险（pconnect）：
  - 连接泄漏/占满：FPM 进程数 × 每进程连接数 ≈ Redis `maxclients`，易耗尽资源；
  - 状态残留：`SELECT db`、`WATCH/MULTI`、`SUBSCRIBE` 等状态在进程间复用，若未清理，影响后续请求；
  - 参数继承：超时、读写选项可能沿用上一次设置；
  - 网络抖动后“半开连接”，需要 keepalive/timeout 策略。

#### 三、最佳实践
```php
<?php
function redis_persistent(): Redis {
    static $r = null;
    if ($r) return $r;
    $r = new Redis();
    // 使用 persistent_id 隔离不同业务/集群
    $r->pconnect('127.0.0.1', 6379, 1.0, 'bizA');
    $r->setOption(Redis::OPT_READ_TIMEOUT, 1.0);
    $r->setOption(Redis::OPT_SERIALIZER, Redis::SERIALIZER_JSON);
    $r->auth('password');
    $r->select(0); // 显式选择 DB，避免残留
    return $r;
}
```
- 配置方面：
  - 控制 PHP-FPM `pm.max_children` 与 Redis `maxclients` 匹配，预留监控/后台连接；
  - 开启 TCP keepalive，设置合理 `read_timeout`，定期健康检查；
  - 不使用订阅/事务的连接混用普通请求（必要时为订阅单独 persistent_id）。

#### 四、何时用 connect
- 任务脚本/CLI、低 QPS 或连接数受限场景，用 connect 更安全；
- Web 高 QPS 且连接预算充足、严格治理残留状态时，用 pconnect。
</details>

### 4. 如何实现分布式锁，有什么问题（陌陌）

<details>
<summary>答案与解析</summary> 

#### 一、基础实现（SET NX PX + 值比对释放）
- 加锁：`SET lock:key unique_token NX PX 30000`（不存在才设置，含过期时间）；
- 解锁：Lua 脚本原子校验值并删除，避免误删他人锁。

```php
<?php
function acquireLock(Redis $r, string $key, int $ttlMs = 30000, int $retry = 5): ?string {
    $token = bin2hex(random_bytes(16));
    for ($i = 0; $i < $retry; $i++) {
        $ok = $r->set($key, $token, ['NX', 'PX' => $ttlMs]);
        if ($ok) return $token;
        usleep((int)(50_000 * pow(2, $i))); // 指数退避
    }
    return null;
}

function releaseLock(Redis $r, string $key, string $token): void {
    $lua = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
    $r->eval($lua, [$key, $token], 1);
}
```

#### 二、进阶：续约与看门狗
- 长任务可后台续约（周期性校验 token 一致后 `PEXPIRE`），避免锁意外过期；
- 续约失败要立刻中止后续写操作（否则可能发生并发写）。

#### 三、常见问题与对策
1. 锁过期引发“并发入侵”
   - 任务被 GC/Stop-the-world 卡住超时，锁到期被他人获取；对策：缩短临界区、定期续约、在资源侧做“幂等/版本校验”。
2. 跨主实例的可用性（RedLock 争议）
   - 多实例获取多数派可提高可用性，但网络分区下仍可能双写；核心数据仍建议单主或引入 DB 级别约束。
3. Fencing Token（围栏令牌）
   - 获取锁时同时拿一个单调递增的 token（如 `INCR lock:seq`），把 token 传给下游，资源方仅接受“更大 token”的请求：
```sql
-- 例如在订单表上用版本号校验
UPDATE `order` SET status = ?, version = version + 1
WHERE id = ? AND version = ?; -- 旧版本更新会失败
```

#### 四、何时不该只靠分布式锁
- 跨多个资源更新且无法在资源侧做幂等等语义校验时；
- 强一致事务需求，优先考虑数据库约束/队列串行/单分片写入。
</details>

### 5. 为什么用跳表实现有序集合？原理与典型场景（字节/滴滴）

<details>
<summary>答案与解析</summary> 

#### 一、为什么不是平衡树/堆
- 有序集合需要按 score 排序，支持按排名、按分数区间、前后向遍历；
- 跳表（SkipList）在实现上简单、插入/删除无需旋转，支持有序范围查询；
- 相比堆：堆只支持极值，不支持任意范围；相比红黑树：实现复杂且不便于按排名快速定位。

#### 二、跳表原理（用于 ZSet 的主要结构）
- 由多层索引 + 底层有序链表组成，元素随机提升层级（概率 p≈1/4）；
- 查找：自顶向下逐层前进，时间复杂度 O(log N)；
- 插入/删除：同样 O(log N)；
- ZSet 实际由“跳表 + 哈希字典”组成：字典用于成员->score 的快速查找，跳表用于按 score 有序遍历和范围操作。

#### 三、复杂度与能力
- 单元素增删查：O(log N)；范围查询：O(log N + K)；排名定位：O(log N)；
- 支持重复 score（稳定性由 member 比较决定）。

#### 四、典型场景
- 排行榜（取 Top N、取自己排名、区间滑动窗口）；
- 延时队列（score=时间戳）；
- 限流与权重调度（score=权重或动态分值）。

#### 五、内存与实现细节
- 大量小 member 适配压缩列表/列表包（老版本）或 listpack（新版本）；
- score 精度为双精度浮点，需要注意精度误差对排序的影响；
- 当元素较少时可能退化为紧凑结构以节省内存。

#### 六、实践建议
- 大量范围查询时优先 ZRANGEBYSCORE/ZREMRANGEBYSCORE，避免全量扫描；
- 排名查询尽量用 ZREVRANGE/ZRANK/ZSCORE 组合；
- 控制集合大小（如只保留 Top N），并开启过期/清理策略。
</details>

### 6. 主从同步原理，哨兵与集群的区别（滴滴）

<details>
<summary>答案与解析</summary> 

#### 一、复制原理（主从/副本）
- 建立阶段：从节点执行 REPLICAOF host port（旧版 SLAVEOF），与主节点握手交换 runid 与复制偏移量 offset。
- 全量复制：首次或无法部分重同步时，主进程 fork 子进程执行 BGSAVE 生成 RDB（或开启无盘复制直接通过套接字流），期间主将新写命令写入复制缓冲；RDB 传给从节点加载后，主把缓冲内增量命令补齐，从此进入命令传播阶段。
- 部分重同步（PSYNC）：从节点带上上次记录的 runid 和 offset 请求，若主节点复制积压缓冲区（repl-backlog）仍保留该区间，直接增量续传；否则退回全量。
- 关键参数：
  - repl-backlog-size：积压环形缓冲大小，决定可覆盖的断连时长；
  - repl-diskless-sync yes：无盘复制，减少磁盘 IO；
  - repl-timeout、client-output-buffer-limit replica：控制超时与从节点输出缓冲；
  - min-replicas-to-write / min-replicas-max-lag：半同步写约束，保障可用副本数与最大延迟。

#### 二、一致性与延迟
- Redis 复制默认异步，写入先返回后传播，读从可能读到旧数据（最终一致）。
- 若业务需要“写后读”更强保证，可在关键写后执行 WAIT &lt;replicas&gt; &lt;ms&gt;，等待 N 个副本确认（近似半同步，牺牲延迟换一致）。
- 读从注意：replica-read-only yes；避免强一致读要求直接读主或加版本校验/重试。

示例（PHP，关键路径写入后等待至少1个副本）：
```php
$r = new Redis();
$r->connect('127.0.0.1', 6379);
$r->set('k', 'v');
list($replicasAck) = $r->rawCommand('WAIT', 1, 1000); // 等1个副本在1s内确认
```

#### 三、哨兵（Sentinel）与集群（Cluster）的区别
1) 哨兵（高可用/主从切换，不分片）
- 职责：监控主从，判定主观下线（sdown）与客观下线（odown），多数派投票选出哨兵领导者执行故障转移；
- 故障转移：挑选最合适从库（优先级、复制偏移、延迟），SLAVEOF NO ONE 晋升为主，其他从库重定向到新主；
- 客户端：通过哨兵发现主地址变化，更新连接；
- 适用：单主多从的高可用，不做自动分片。

最简 sentinel.conf 片段：
```conf
port 26379
sentinel monitor mymaster 10.0.0.10 6379 2   # quorum=2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1
# 如果主有密码
sentinel auth-pass mymaster mypass
```

2) 集群（分片+高可用）
- 16384 个哈希槽自动分片，节点间通过集群总线交换 Gossip 信息与故障探测；
- 每个主节点可挂从节点，主故障由集群内部选举从库接管；
- 客户端需支持 MOVED/ASK 重定向与多节点路由，多 key 操作需在同槽（可用 hash tag）。

选择建议：
- 小规模、单主读多从、强依赖多 key 事务 => 哨兵；
- 大规模水平扩展、自动分片与内置故障转移 => 集群。
</details>

### 7. Redis Cluster 用什么协议同步数据？哨兵的选举机制（陌陌）

<details>
<summary>答案与解析</summary> 

#### 一、Cluster 的数据与元数据通道
- 数据复制：仍使用 Redis 复制协议（PSYNC）在常规 TCP 端口上传输主从数据，和单机主从一致；
- 元数据传播：节点状态、槽位分配、故障探测通过“集群总线”在 port+10000 端口走二进制协议进行 Gossip（Ping/Pong/Fail），不承载用户数据。

#### 二、哨兵的选举与故障转移流程
1) 下线判定：
- sdown：单个哨兵基于心跳超时判定主观下线；
- odown：多个哨兵相互通信，超过 quorum 后形成客观下线。
2) 领导者选举：
- 进入故障转移 epoch，每个哨兵对候选从库发起 is-master-down-by-addr 投票；
- 单个 epoch 内每个哨兵只能投一票，获得多数（>半数哨兵且满足 quorum）的哨兵成为 leader。
3) 执行转移：
- 选择从库规则：优先级 sentinel priority、复制 offset 最大、与主延迟最小、最近无断连；
- 向目标从库发送 SLAVEOF NO ONE 晋升为主；
- 其他从库 SLAVEOF 新主；
- 通告客户端更新路由。

常用命令与排障：
```bash
# 查看主节点状态与哨兵列表
redis-cli -p 26379 SENTINEL MASTER mymaster
redis-cli -p 26379 SENTINEL SENTINELS mymaster
# 手动触发一次故障转移（演练）
redis-cli -p 26379 SENTINEL FAILOVER mymaster
```

提示：Cluster 模式下的故障转移由集群自身完成（不依赖 Sentinel），但在无分片的主从架构里推荐用 Sentinel 做 HA。
</details>

### 8. RDB 与 AOF 的原理（滴滴/高德）

<details>
<summary>答案与解析</summary> 

#### 一、RDB（快照）
- 触发：SAVE（阻塞）/BGSAVE（fork 子进程异步）/自动 save 策略；
- 机制：fork 结合写时复制（CoW），子进程将内存快照持久化为紧凑的 RDB 文件；
- 优点：文件小、加载快、适合冷备/全量复制；
- 缺点：两次快照之间可能丢失最近秒级/分钟级写入。

#### 二、AOF（追加日志）
- 机制：将写命令以协议文本顺序追加到 AOF 文件，重启时重放恢复；
- fsync 策略：always（最安全，最慢）/everysec（常用，最多丢 1s）/no（由 OS 刷盘）；
- 重写：AOF 文件膨胀后后台重写（基于内存快照生成等价最小指令集），可与写入并行；
- 混合模式：aof-use-rdb-preamble yes，重写输出以 RDB 作为前导，恢复更快。

#### 三、如何选择
- 对数据安全敏感：开启 AOF everysec，再配合 RDB（双保险）；
- 启动速度优先：保留 RDB；
- 生产普遍：RDB + AOF 混合，兼顾恢复速度与持久性。

示例配置（redis.conf）：
```conf
# RDB
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes

# AOF
appendonly yes
appendfsync everysec
no-appendfsync-on-rewrite yes
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-use-rdb-preamble yes
```
</details>

### 9. 数据过期与淘汰策略（滴滴/高德/字节）

<details>
<summary>答案与解析</summary> 

#### 一、过期机制
- 懒惰删除：访问键时发现已过期才删除；
- 定期扫描：后台主动采样过期字典，按比例删除超时键（受 hz、active-expire-effort 影响）。

设置与查询：
```bash
SET k v EX 3600      # 或 PX 5000 设置毫秒级
TTL k                 # -2=不存在，-1=无过期，其它=剩余秒
PTTL k
```

#### 二、内存与淘汰策略（触发于达到 maxmemory）
- maxmemory-policy：
  - noeviction（默认）：写命令报错；
  - allkeys-lru / volatile-lru：LRU 近似淘汰（采样）；
  - allkeys-lfu / volatile-lfu：LFU 淘汰（更能反映访问频率）；
  - volatile-ttl：优先淘汰 TTL 近的键；
  - allkeys-random / volatile-random：随机淘汰。
- 建议：面向通用缓存优选 allkeys-lfu 或 allkeys-lru；为热点保护可适当增大样本数。

示例（运行时调整）：
```bash
CONFIG SET maxmemory 4gb
CONFIG SET maxmemory-policy allkeys-lfu
```

#### 三、实践要点
- 给缓存 TTL 加随机抖动（防雪崩），例如 3600 + rand(0,300)；
- 大 Key 拆分，小心集合类全量遍历；
- 对关键 Key 设置持久化或二级缓存兜底；
- 监控：used_memory、evicted_keys、expired_keys、keyspace_hits/misses。
</details>

### 10. 缓存雪崩、击穿、穿透（滴滴/陌陌）

<details>
<summary>答案与解析</summary> 

#### 一、概念与危害
- 雪崩：大量 Key 在同一时间过期或 Redis 故障，后端被压垮；
- 击穿：某个热点 Key 过期瞬间并发涌向后端；
- 穿透：请求的 Key 在缓存和数据库都不存在，直接打到后端。

#### 二、应对策略
1) 雪崩：
- TTL 加随机抖动；
- 预热与分批加载；
- 多级缓存（本地 Caffeine/ APCu + Redis + DB）；
- 限流降级，缓存隔离（不同业务空间）。

2) 击穿：
- 互斥锁重建：只有一个请求重建缓存，其它等待或返回旧值；
- 逻辑过期 + 异步刷新：缓存中保存数据与逻辑过期时间，过期后异步刷新同时返回旧值（stale-while-revalidate）。

3) 穿透：
- 布隆过滤器拦截不存在 Key；
- 缓存空值（短 TTL）并加防刷；
- 参数校验、黑白名单与限流。

#### 三、PHP 实战模板
1) TTL 抖动与空值缓存
```php
function cacheSetWithJitter(Redis $r, string $k, string $v, int $ttlBase = 3600): void {
    $jitter = random_int(0, (int)($ttlBase * 0.1)); // 10% 抖动
    $r->set($k, $v, ['ex' => $ttlBase + $jitter]);
}

function cacheGetOrNull(Redis $r, string $k, callable $loader, int $ttlShort = 60) {
    $val = $r->get($k);
    if ($val !== false) return json_decode($val, true);
    $data = $loader();
    if ($data === null) { // 缓存空值短 TTL
        $r->set($k, json_encode(null), ['ex' => $ttlShort]);
        return null;
    }
    $r->set($k, json_encode($data), ['ex' => 3600 + random_int(0,300)]);
    return $data;
}
```

2) 热点 Key 击穿防护（互斥锁 + 逻辑过期）
```php
function getHotKey(Redis $r, string $k, callable $loader, int $ttl = 3600) {
    $raw = $r->get($k);
    if ($raw !== false) {
        $payload = json_decode($raw, true);
        if ($payload && $payload['expireAt'] > time()) return $payload['data'];
    }
    $lockKey = "lock:$k";
    $token = bin2hex(random_bytes(8));
    if ($r->set($lockKey, $token, ['nx', 'ex' => 10])) {
        try {
            $data = $loader();
            $r->set($k, json_encode(['data' => $data, 'expireAt' => time()+$ttl]));
            return $data;
        } finally {
            $lua = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
            $r->eval($lua, [$lockKey, $token], 1);
        }
    } else {
        // 兜底：若有旧值则返回旧值；否则短暂休眠重试
        if ($raw !== false) {
            $payload = json_decode($raw, true);
            if ($payload) return $payload['data'];
        }
        usleep(100 * 1000);
        return getHotKey($r, $k, $loader, $ttl);
    }
}
```

3) 布隆过滤器（如启用 RedisBloom 模块）
```bash
BF.RESERVE bf:users 0.01 1000000   # 1% 误判率，100w 容量
BF.ADD bf:users u:1001
BF.EXISTS bf:users u:1002
```

#### 四、监控与演练
- 监控：命中率、后端 QPS、evicted_keys、latency、热点 Key 分布；
- 演练：下线/过期压测、哨兵/集群故障转移演练、降级策略检查。
</details>

---

## PHP

### 1. php-fpm 的生命周期，进程创建方式及优缺点（腾讯/百度/滴滴/陌陌）

<details>
<summary>答案与解析</summary> 

#### 一、生命周期与基本架构
- 进程模型：1 个 Master + N 个 Worker（每个 Worker 进程内 1 个请求，非线程）。
- 启动：Master 读取 php-fpm.conf 与各池 pool.d/*.conf，按池创建监听 socket（TCP/Unix）。
- 运行：Master 负责接受信号、监控/拉起/回收 Worker；Worker 从监听队列取连接，解析 FastCGI 请求，调用 Zend 引擎执行 PHP 脚本，返回响应。
- 退出：根据信号优雅停止（SIGQUIT）或快速停止（SIGTERM），reload（SIGHUP）重新加载配置平滑重启。

常用命令：
```bash
# 以 systemd 举例
systemctl reload php-fpm   # 平滑重载配置
systemctl restart php-fpm  # 重启
```

#### 二、进程创建方式（pm）与对比
1) pm = static
- 固定数量的 worker（pm.max_children）；
- 优点：性能稳定、无抖动；缺点：内存占用固定，大量空闲时浪费。

2) pm = dynamic（默认）
- 初始 pm.start_servers，按负载在 [pm.min_spare_servers, pm.max_spare_servers] 范围内调节，受 pm.max_children 上限约束；
- 优点：兼顾吞吐与资源；缺点：峰值突增时有进程扩容开销。

3) pm = ondemand
- 无请求时不保留进程，有请求时按需拉起，空闲超过 pm.process_idle_timeout 回收；
- 优点：闲时极省内存；缺点：首个请求冷启动开销，突发时易排队。

选择建议：
- 长期高并发、资源充足：static；
- 常规 Web：dynamic；
- 边缘/函数式、低 QPS：ondemand。

#### 三、关键参数与诊断
- pm.max_children：并发上限（受内存限制，估算：child_memory × max_children < 可用内存）。
- request_terminate_timeout：防止长请求占坑，超时杀进程重拉；
- pm.max_requests：每个 worker 处理 N 个请求后重启，缓解内存碎片/泄漏；
- slowlog 与 request_slowlog_timeout：输出慢脚本调用栈；
- access.log、ping.path、status_path（/status）结合 nginx stub_status 监控。

池配置示例（/etc/php-fpm.d/www.conf）：
```conf
[www]
user = www-data
group = www-data
listen = /run/php-fpm.sock
listen.backlog = 1024
pm = dynamic
pm.max_children = 80
pm.start_servers = 10
pm.min_spare_servers = 10
pm.max_spare_servers = 20
pm.max_requests = 5000
request_terminate_timeout = 60s
request_slowlog_timeout = 3s
slowlog = /var/log/php-fpm/slow.log
php_admin_value[error_log] = /var/log/php-fpm/error.log
php_admin_flag[log_errors] = on
```

#### 四、常见问题与优化
- 502/504：上游超时、脚本超时、后端阻塞；调大 fastcgi_read_timeout/pm.max_children 并优化脚本。
- 内存打满：估算 child 内存，合理设置 max_children 与 max_requests，排查泄漏扩展。
- 冷启动慢：ondemand 切 dynamic 或预热；
- 连接排队：listen.backlog 与内核队列（net.core.somaxconn）配合调优。
</details>

### 2. PHP 数组遍历为什么能保证有序（滴滴）

<details>
<summary>答案与解析</summary> 

#### 一、底层结构决定“有序哈希”
- PHP 数组 = HashTable（哈希表）+ 有序双向链表（Bucket 按插入顺序链接）。
- 哈希表用于 O(1) 平均时间的 key->bucket 查找；链表保存插入顺序，遍历时沿链表实现有序输出。
- 因此 foreach 默认按“插入顺序”遍历，而非按 key 的字母或数值排序。

#### 二、数值键与自动重排
- 数组混合键：字符串键哈希；整数键可作为索引使用，但仍走哈希表。
- unset 某元素不会打乱其他元素的链表相对顺序；新插入的元素追加到链表尾；
- 注意 array_values 会重建索引为 0..n-1，但遍历顺序仍按原链表顺序。

#### 三、示例
```php
$a = [];
$a['b'] = 1;
$a['a'] = 2;
$a[100] = 3;
$a[2] = 4;
foreach ($a as $k=>$v) { echo "$k:$v "; } // b:1 a:2 100:3 2:4
```

#### 四、注意事项与性能
- 插入顺序的维护需要额外内存（双向链表指针），大数组请关注内存占用；
- 随机删除/插入频繁时，链表也要维护指针，存在一定开销；
- 若需要按 key/值排序，请使用 ksort/sort 等显式排序，遍历本身不做排序；
- PHP7/8 对 HashTable 做了紧凑优化（packed array）以降低内存，但遍历顺序保持插入顺序语义不变。
</details>

### 3. PHP 怎么实现的弱类型；如何实现一个扩展（腾讯）

<details>
<summary>答案与解析</summary> 

#### 一、PHP 的弱类型实现机制
- 统一的 zval 结构表示动态类型，包含类型标签（IS_LONG/IS_STRING/IS_ARRAY/...）与值联合体；
- 变量赋值/传参时按需要进行类型转换（Type Juggling），如字符串转数字、布尔转换等；
- 哈希表用于数组/对象属性存储，函数调用参数通过 zval 传递，运行时决定行为；
- 严格模式（declare(strict_types=1)）只影响标量参数/返回类型的检查，不改变底层动态类型本质。

示例：
```php
function add($a, $b) { return $a + $b; }
var_dump(add('3', 4)); // int(7) 动态转换
```

#### 二、如何实现一个 PHP 扩展（Zend 扩展/模块）
1) 基本结构
- config.m4 / config.w32：构建配置；
- php_foo.h / foo.c：扩展头与实现；
- 声明模块入口 zend_module_entry，注册函数表 zend_function_entry；
- 生命周期钩子：MINIT/MINFO（模块初始化/信息）、RINIT/RSHUTDOWN（请求级）。

2) 函数注册与参数解析
- PHP_FUNCTION(foo_bar) 定义导出函数；
- ZEND_PARSE_PARAMETERS_START 宏解析参数类型；
- RETURN_LONG/RETURN_STR 等返回。

3) 构建与安装
- phpize && ./configure --enable-foo && make && make install；
- 在 php.ini 中 extension=foo。

#### 三、最小示例（C）
```c
// foo.c
#ifdef HAVE_CONFIG_H
# include "config.h"
#endif
#include "php.h"

PHP_FUNCTION(foo_add) {
    zend_long a, b;
    ZEND_PARSE_PARAMETERS_START(2, 2)
        Z_PARAM_LONG(a)
        Z_PARAM_LONG(b)
    ZEND_PARSE_PARAMETERS_END();
    RETURN_LONG(a + b);
}

static const zend_function_entry foo_functions[] = {
    PHP_FE(foo_add, NULL)
    PHP_FE_END
};

zend_module_entry foo_module_entry = {
    STANDARD_MODULE_HEADER,
    "foo",               // 扩展名
    foo_functions,        // 函数表
    NULL,                 // MINIT
    NULL,                 // MSHUTDOWN
    NULL,                 // RINIT
    NULL,                 // RSHUTDOWN
    NULL,                 // MINFO
    NO_VERSION_YET,
    STANDARD_MODULE_PROPERTIES
};

ZEND_GET_MODULE(foo)
```

使用：
```bash
phpize && ./configure --enable-foo && make -j && sudo make install
echo "extension=foo" > /etc/php.d/20-foo.ini
php -r 'var_dump(foo_add(2,5));'
```

提示：生产扩展需注意线程安全（TSRM）、引用计数/垃圾回收（zval_refcounted）、内存分配（emalloc/efree）。
</details>

### 4. 常见魔术方法和函数（腾讯/滴滴）

<details>
<summary>答案与解析</summary> 

#### 一、类相关魔术方法
- __construct/__destruct：构造与析构；
- __get/__set/__isset/__unset：访问不可见/不存在属性时触发；
- __call/__callStatic：调用不可见/不存在的方法时触发；
- __invoke：对象当函数调用时触发；
- __toString：对象转字符串时触发；
- __clone：对象 clone 时触发（可做深拷贝）；
- __sleep/__wakeup：序列化/反序列化时触发；
- __serialize/__unserialize：新序列化协议；
- __debugInfo：var_dump 时自定义输出。

示例：
```php
class Box {
    private $x = 1;
    public function __get($n){ if($n==='x') return $this->x; throw new Exception('no'); }
    public function __toString(){ return 'Box('.$this->x.')'; }
}
$b = new Box();
echo $b;          // Box(1)
echo $b->x;       // 1
```

#### 二、常用魔术常量
- __FILE__、__DIR__、__LINE__、__FUNCTION__、__METHOD__、__CLASS__、__TRAIT__、__NAMESPACE__。

#### 三、其他常见“魔术函数/特性”
- spl_autoload_register：自动加载类；
- Closure::fromCallable / use 绑定；
- Iterator/Generator：yield 实现协程式生成器；
- 属性/方法可见性、Final/Abstract、Trait 组合。

注意：魔术方法应慎用，避免过度魔法影响可读性与性能；__get/__set 大量触发会降低性能。
</details>

---

## Elasticsearch（ES）

### 1. 深度分页会有什么问题（滴滴/百度/陌陌）

<details>
<summary>答案与解析</summary> 

#### 一、问题本质与危害
- from/size 深度分页会让每个分片都要产生并维护一个大小为 from+size 的优先队列（堆）用于排序与剪裁，内存与 CPU 成本呈线性增长（近似 O(shards × (from+size))）；
- 默认 index.max_result_window=10000，超过会报错（Result window is too large），即使放开该限制，仍会引发严重的 heap 占用、GC 频繁、响应时间指数级上升；
- 深度页排序的稳定性问题：如果用 _score 或非唯一字段排序，跨页可能出现重复/丢失（由于索引刷新、并发写入导致结果集边界不稳定）。

#### 二、常见现象
- QPS 正常但 p95/p99 延迟暴涨，ES 节点出现频繁 Young/Old GC；
- 查询返回慢甚至 429/超时；运维侧可观察到查询阶段 CPU 飙高、heap 使用逼近高水位。

#### 三、正确的解决思路
1) 改 from/size 为 search_after
- 原理：基于“上一页最后一条的排序值”继续向后扫描，不再跳过前面的 N 条，避免堆中维护 from+size 个候选。
- 要求：提供稳定的排序字段组合，且包含全局唯一的 tiebreaker（如 _id 或 _shard_doc）；不支持随机排序；不支持直接跳转第 N 页（需逐页推进）。

2) 结合 PIT（Point In Time）保证跨页一致性
- PIT 可在多次分页请求间固定一个数据快照，避免边分页边刷新导致重复/漏数；搭配 search_after 使用，效果最佳。

3) Scroll 仅用于离线批处理/导出
- Scroll 创建快照并顺序滚动，适合全量导出或批处理，不适合用户交互式分页（会占用资源、上下页跳转不灵活）。

4) 其他优化
- 控制单页 size（如≤100），限制最大可翻页深度；
- 业务侧避免“无限下拉到底”，提供搜索条件缩小结果集；
- 需要大范围排序时可考虑 Index Sorting（索引预排序）、合适的 doc_values、冷热分层、必要时做汇总/快照表。

#### 四、请求示例
1) 使用 PIT + search_after（按时间+_id 稳定排序）
```bash
# 打开 PIT，有效期 1 分钟
curl -X POST "http://localhost:9200/my_index/_pit?keep_alive=1m"

# 第1页：size 20，按 create_time 降序 + _id 作为并列消解
curl -H 'Content-Type: application/json' -X POST \
  "http://localhost:9200/_search" -d '{
  "size": 20,
  "pit": {"id": "PIT_ID_FROM_PREV_CALL", "keep_alive": "1m"},
  "sort": [
    {"create_time": "desc"},
    {"_id": "desc"}
  ],
  "query": {"term": {"status": "ok"}}
}'

# 取上一页最后一条的 sort 值继续请求
curl -H 'Content-Type: application/json' -X POST \
  "http://localhost:9200/_search" -d '{
  "size": 20,
  "pit": {"id": "PIT_ID_FROM_PREV_CALL", "keep_alive": "1m"},
  "search_after": [1693999999000, "last_id_from_prev_page"],
  "sort": [
    {"create_time": "desc"},
    {"_id": "desc"}
  ],
  "query": {"term": {"status": "ok"}}
}'

# 使用完成后关闭 PIT
curl -X DELETE "http://localhost:9200/_pit" -H 'Content-Type: application/json' -d '{"id":"PIT_ID_FROM_PREV_CALL"}'
```

2) PHP（Elasticsearch 客户端伪代码）
```php
$pit = $client->openPointInTime([
    'index' => 'my_index',
    'keep_alive' => '1m',
]);
$pitId = $pit['id'];

$params = [
    'body' => [
        'size' => 20,
        'pit' => ['id' => $pitId, 'keep_alive' => '1m'],
        'sort' => [ ['create_time' => 'desc'], ['_id' => 'desc'] ],
        'query' => ['term' => ['status' => 'ok']],
    ]
];
$res = $client->search($params);
$lastSort = end($res['hits']['hits'])['sort'];

// 下一页
$params['body']['search_after'] = $lastSort;
$res2 = $client->search($params);

// 结束
$client->closePointInTime(['body' => ['id' => $pitId]]);
```

#### 五、实践建议与避坑
- 交互分页：优先 PIT + search_after + 稳定排序（含唯一 tiebreaker）。
- 批量导出/全表扫描：使用 Scroll（或 PIT+search_after），注意设置合理 keep_alive，处理完尽快清理上下文。
- 谨慎放大 index.max_result_window，症状会缓解但无法从根本上解决深度分页的堆内存与 CPU 问题。
- 若确需“跳转第 N 页”，在业务层通过“锚点记录”（last_sort）逐页推进或改为“时间游标”模式。
</details>

### 2. 倒排索引的原理（字节/高德）

<details>
<summary>答案与解析</summary> 

#### 一、什么是倒排索引（Inverted Index）
- 面向“词 → 文档”的索引结构：为每个 Term 建立一份倒排表（Postings List），记录出现该词的文档列表以及位置信息；
- 用于快速全文搜索、短语匹配、布尔组合检索等，是 Lucene/Elasticsearch 检索的核心。

#### 二、核心组成
- 词典（Term Dictionary）：存储所有词项，常用有限状态机/前缀压缩（Lucene 使用 FST）以减少内存与磁盘占用；
- 倒排表（Postings）：按文档编号 docID 升序存储，包含：
  - docID：文档标识；
  - tf（term frequency）：词频，BM25/TF-IDF 评分使用；
  - positions：词在文档中的位置列表（支持短语/邻近查询 slop）；
  - offsets/payload：可选，用于高亮、载荷等；
- 列式存储 doc_values：非倒排结构，面向排序/聚合/脚本，按列保存字段值，避免将字段值加载进堆内存（fielddata）。

#### 三、建立过程（分析链）
- 文档写入时经过 Analyzer：char_filter → tokenizer → token_filter（大小写、停用词、同义词、拼音/中文分词等），生成 Term 序列；
- 对 text 字段走 Analyzer，keyword 字段不分词，适合精确匹配/聚合；
- 索引由多个不可变 Segment 组成，增量写入生成新 Segment，后台合并（merge）以减少段的数量与删除标记。

#### 四、查询如何命中
- Term/Terms Query：精确词项匹配（通常用于 keyword）；
- Match/Match Phrase：先用同 Analyzer 将查询文本分词；短语匹配依赖 positions，slop 控制词间距；
- Prefix/Wildcard：基于词典前缀/通配符匹配，范围大时成本高，谨慎使用；
- Range（数值/日期）/Geo：非倒排路径，Lucene 使用 BKD 树专门结构；
- 排序/聚合：基于 doc_values，避免启用 fielddata（高堆内存）。

#### 五、更新与删除
- 文档“更新”= 新增一个版本并打标删除旧版本（段不可变）；
- 删除通过删除标记实现，合并时物理清理；
- 段合并与刷新：写入先进内存缓冲，周期性 refresh 生成可见的新段；commit/flush 将事务日志落盘以保证持久性。

#### 六、实践与建模建议
- 文本字段：text + keyword 多字段映射，既支持全文检索又支持聚合/排序；
- 选择合适的 Analyzer（中英文混合可分字段配置）；keyword 使用 normalizer（如 lowercase）做规范化；
- 控制字段数量与存储选项：对不需要检索/聚合的字段关闭 index/doc_values/store；
- 注意通配符/模糊匹配成本，能改为前缀/边界 ngram 更好；
- 高频更新场景关注段合并 IO 与写入放大，适当调大 refresh_interval、批量写入。

#### 七、示例
1) Mapping 与分析
```bash
# 创建索引：title 为 text（含 keyword 子字段），tags 为 keyword，created_at 为 date
curl -H 'Content-Type: application/json' -X PUT \
  "http://localhost:9200/articles" -d '{
  "settings": {
    "analysis": {
      "normalizer": {
        "lower_norm": { "type": "custom", "char_filter": [], "filter": ["lowercase"] }
      }
    }
  },
  "mappings": {
    "properties": {
      "title": { "type": "text", "fields": { "raw": { "type": "keyword", "ignore_above": 256 } } },
      "tags":  { "type": "keyword", "normalizer": "lower_norm" },
      "created_at": { "type": "date" }
    }
  }
}'

# 查看分词效果
curl -H 'Content-Type: application/json' -X POST \
  "http://localhost:9200/articles/_analyze" -d '{
  "analyzer": "standard",
  "text": "Quick Brown Fox"
}'
```

2) 检索示例（短语/排序/聚合）
```bash
# 短语查询（允许 1 个词距）
curl -H 'Content-Type: application/json' -X GET \
  "http://localhost:9200/articles/_search" -d '{
  "query": {"match_phrase": {"title": {"query": "quick fox", "slop": 1}}},
  "sort": [{"created_at": {"order": "desc"}}, {"_id": {"order": "desc"}}]
}'

# 按标签聚合（基于 doc_values）
curl -H 'Content-Type: application/json' -X GET \
  "http://localhost:9200/articles/_search" -d '{
  "size": 0,
  "aggs": {"by_tag": {"terms": {"field": "tags"}}}
}'
```

3) PHP 客户端伪代码
```php
$client->indices()->create([
  'index' => 'articles',
  'body' => [
    'settings' => [
      'analysis' => [
        'normalizer' => [
          'lower_norm' => ['type' => 'custom', 'char_filter' => [], 'filter' => ['lowercase']],
        ]
      ]
    ],
    'mappings' => [
      'properties' => [
        'title' => ['type' => 'text', 'fields' => ['raw' => ['type' => 'keyword', 'ignore_above' => 256]]],
        'tags'  => ['type' => 'keyword', 'normalizer' => 'lower_norm'],
        'created_at' => ['type' => 'date'],
      ]
    ]
  ]
]);
```

提示：数值/日期/地理类型在 Lucene 内部不走倒排，而使用专门结构（BKD 等）；设计 Mapping 时按查询类型选择合适字段类型和索引选项。
</details>

### 3. LSM 树原理（字节）

<details>
<summary>答案与解析</summary> 

#### 一、概念与适用场景
- LSM（Log-Structured Merge）树是一种为高写入吞吐优化的存储结构，核心思想是“顺序写 + 批量合并”，避免随机写放大；
- 典型应用：LevelDB/RocksDB/HBase 等 KV/列式存储；在搜索引擎领域，Lucene 的“段（segment）+ 合并（merge）”与 LSM 思想同源（不可变段、后台合并）。

#### 二、写入流程（典型实现）
1) 写前日志（WAL）
- 先将操作追加到 WAL，保证崩溃恢复；
2) 内存表（MemTable）
- 同步写入可排序的内存结构（如跳表），达到阈值后转为 Immutable MemTable；
3) 刷盘生成 SSTable（Sorted String Table）
- 将 Immutable MemTable 顺序写出为有序、不可变的 SSTable 文件，通常先写临时文件后原子替换；
4) 多层存储与合并（Compaction）
- 新文件进入 Level 0，后台不断将上层与下层重叠键范围的文件合并，生成更低层级的有序文件，清理删除标记（tombstone），压缩、重写索引。

#### 三、读路径与加速结构
- Point Get：从 L0→L1→… 查找，利用每个 SSTable 的 Bloom Filter 快速判定“可能存在/一定不存在”，再使用稀疏索引 + 数据块索引定位；
- Range Scan：构造多路归并迭代器（各层/各文件各自按顺序），归并去重/处理删除标记；
- 缓存：块缓存（block cache）/页缓存减少磁盘 IO；索引/元数据常驻内存以加速定位。

#### 四、合并策略
- Tiered（Size/Level-0 多文件合并）：先积累再大批量合并，写放大较低，但读放大较高；
- Leveled（分层且低层不重叠）：更频繁的合并以保证低层文件不重叠，读放大更低，写放大更高；
- 触发条件：层级大小阈值、文件数阈值、后台压力与速率限制（compaction throttle）。

#### 五、删除与更新
- 更新=写入新版本（后写覆盖前写）；删除=写入 tombstone，查询/合并时跳过；
- 压实（compaction）过程中清理过期数据与 tombstone，控制空间放大。

#### 六、优缺点与对比
- 优点：
  - 写入顺序化，充分利用顺序 IO 与批量压缩；
  - 更易于批量写、压缩、数据重排；
- 缺点：
  - 读放大：Point Get 可能触及多层多文件；
  - 写放大：后台合并会反复重写数据块；
  - 空间放大：多层同时保留相同键的不同版本；
- 与 B+ 树对比：
  - B+ 树更新为“就地修改”，点查/范围查询性能稳定，适合读多写少；
  - LSM 顺序写吞吐高，适合写多；但需 Bloom、块缓存与足够的后台合并能力以抑制读放大。

#### 七、与 Elasticsearch/Lucene 的关系
- Lucene 写入会生成不可变 segment，后台进行合并（merge）以减少段的数量与删除标记，这与 LSM 的“不可变文件 + 后台合并”高度相似；
- Lucene 中的 term dictionary、倒排表、列式 doc_values 对应于 LSM 中的“按键（或按倒排项）排序的不可变块”，合并时重写段与索引结构；
- 虽实现细节不同（如是否使用 Bloom Filter），但在“写顺序化 + 段不可变 + 后台合并”这一思想上是一致的。

#### 八、实践调优要点（以 RocksDB/HBase 为例）
- 写入：合并风格（leveled/tiered）、目标文件大小、写缓冲区（memtable）大小与数目、WAL 同步策略（fsync 频率）；
- 查询：Bloom bits/FP 率、block size、cache 大小；
- 合并：限速与并发、压缩算法（LZ4/ZSTD）、冷热分层；
- 数据：大 value 拆分、合理 TTL、控制小文件，避免 L0 堵塞；
- 运维：监控写/读/合并放大，观测延迟与 backlog，必要时限流生产者。
</details>

---

## Kafka

### 1. Kafka 的架构与大致存储结构（高德/字节/滴滴）

<details>
<summary>答案与解析</summary> 

#### 一、核心组件
- Broker：消息服务器，存储分区数据并对外提供读写；
- Controller：负责分区 Leader 选举、分配与集群元数据管理（新版本使用 KRaft 内置共识，无需 ZooKeeper）；
- Topic/Partition：主题由多个分区组成，分区是并发与扩展的最小单元；
- Replica/ISR：分区在多个 Broker 上有副本，Leader 负责读写，Follower 复制；ISR（in-sync replicas）为与 Leader 同步的副本集合；
- Producer/Consumer：生产者写入分区，消费者以消费组为单位订阅分区，组内同一分区同一时刻只会被一个消费者实例消费（保证分区内顺序）。

#### 二、存储结构（日志分段 + 索引）
- 每个分区对应一个“追加写”的日志目录，内部按多个 segment（段）组织：
  - .log：数据文件，文件名为该段的 baseOffset（如 00000000000000000000.log），记录批量（RecordBatch）顺序追加；
  - .index：稀疏索引，映射相对 offset → 物理位置（byte position），便于快速定位；
  - .timeindex：时间索引，映射 timestamp → offset，用于按时间查找；
- 读写路径高度依赖 OS Page Cache：
  - 写入：Producer 数据经网络到达后，Broker 追加写到 .log（顺序写），由 OS 缓存并异步落盘；
  - 读取：Consumer Fetch 时优先从 Page Cache 读取，Broker 使用零拷贝（sendfile/transferTo）直接从文件发送到 socket。

#### 三、可靠性与高可用
- 副本与选举：replication.factor≥3，优先在不同机架（rack awareness）分布；
- 同步策略：acks=all + min.insync.replicas≥2，避免只有 Leader 写入成功导致的丢失；
- 关闭不干净选举（unclean.leader.election.enable=false）以避免脑裂数据回退；
- 保留与清理：
  - 基于时间/大小的保留（log.retention.hours / log.retention.bytes），到期后删除过老 segment；
  - Compaction（log.cleanup.policy=compact）按 key 保留最新版本，适合存放状态快照/去重数据；
- 日志刷盘：通常依赖 OS；可用 log.flush.* 或磁盘同步策略控制 fsync 频率（吞吐与持久性的权衡）。

#### 四、性能关键点
- 顺序写 + 批量（batch.size、linger.ms）提升吞吐；
- 零拷贝 sendfile/transferTo 降低内核/用户态复制与上下文切换；
- mmapped index/日志有利于随机索引访问；
- 分区并行：增加分区数提升并发上限，注意与消费者组规模匹配。

#### 五、常用命令
```bash
# 创建主题（3 副本，6 分区）
kafka-topics.sh --create --topic orders --partitions 6 --replication-factor 3 --bootstrap-server localhost:9092

# 查看主题描述（Leader/ISR/副本分布）
kafka-topics.sh --describe --topic orders --bootstrap-server localhost:9092

   # 查看消费组位点
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group g1 --describe

# 修改保留策略为按 key 压缩
kafka-configs.sh --bootstrap-server localhost:9092 --entity-type topics --entity-name orders \
  --alter --add-config cleanup.policy=compact
```

提示：
- 分区内消息有序，但跨分区无全局顺序；需要全局顺序请使用单分区或上层排序缓冲；
- 增加分区可能导致 Key 到分区映射变化，跨分区的“顺序/去重”需谨慎设计。
</details>

### 2. 如果消费者数超过分区数会怎么样？（顺丰/滴滴）

<details>
<summary>答案与解析</summary> 

#### 结论先行
- 同一消费组内，一个分区同一时刻只会被一个消费者实例消费；
- 当消费者实例数 > 分区数时，多出来的消费者会处于空闲状态（无分配的分区），资源浪费；
- 扩容消费能力通常靠“增加分区数”而不是盲目增加消费者实例数。

#### 分配与重平衡
- 分配策略：Range、RoundRobin、Sticky 等，围绕“分区数≥消费者数时尽量均衡”目标；
- 触发重平衡：组成员变化（实例上下线）、订阅主题变化、心跳超时等，会导致分区重新分配并短暂不可消费；
- Hot partition：按 Key 分区可能产生热点，纵使消费者数充分，也会被热点分区拖累。

#### 实战建议
- 规划并发：以“分区数 ≥ 预期并发度”为目标；如果是 HTTP 处理线程池，建议分区数设置为 CPU 核数的 2~3 倍起步，并按监控逐步调整；
- 增加消费者实例前，优先评估：是否存在慢消费、批量/并发度不足、消息处理瓶颈（外部依赖）等；
- StickyAssignor 可减少重平衡时的分区迁移；
- 多组消费：同一主题可以通过不同消费组实现“广播”式独立消费，互不影响。

#### 监控与排查
- 指标：consumer lag、rebalance 次数、处理耗时（p99）、失败率、空闲消费者比例；
- 现象：消费者很多但整体吞吐不升反降，多为分区数不足、热点或外部依赖慢导致；
- 方案：增加分区或拆分热点 key，优化批量与并发，启用异步 I/O 或本地缓存解耦。
</details>

### 3. 怎么保证数据的可靠投递？（陌陌/字节）

<details>
<summary>答案与解析</summary> 

#### 目标与语义
- At-most-once：最多一次，可能丢但不会重复；
- At-least-once：至少一次，不丢但可能重复（生产中最常见，需幂等处理）；
- Exactly-once：端到端恰好一次（Kafka 通过幂等生产者 + 事务 + 读已提交实现）。

#### 生产者侧（写入不丢、尽量不重）
- enable.idempotence=true 开启幂等生产者，避免重试导致的重复写；
- acks=all + retries>0 + delivery.timeout.ms 合理设置重试上限与超时；
- max.in.flight.requests.per.connection≤5（幂等约束，越小越稳，可能降低吞吐）；
- linger.ms/batch.size 增大批量以提升吞吐并降低小包；
- 有序性：同一 key 保证分区内顺序，避免跨分区破坏顺序。

#### Broker/集群侧（不丢与高可用）
- replication.factor≥3，min.insync.replicas≥2，acks=all 三件套；
- 关闭不干净选举 unclean.leader.election.enable=false，避免脑裂回退；
- 机架感知 rack awareness，提升故障域隔离；
- 合理磁盘/刷盘策略：顺序写 + OS Page Cache，必要时权衡 fsync 频率。

#### 消费者侧（不重复消费或可容忍重复）
- 手动提交位点：enable.auto.commit=false，处理成功后 commitSync；
- 幂等消费：按业务 key 去重（如写 DB UPSERT/唯一键）、幂等接口；
- 失败重试与死信：重试主题 + backoff；无法处理入 DLQ；
- 再均衡处理：在 onPartitionsRevoked 前提交已处理位点，避免重复。

#### 事务与 EOS（端到端）
- 生产者设置 transactional.id，初始化事务；按批 begin → 发送 → commit；
- 下游消费者 isolation.level=read_committed 只读已提交消息；
- 写入多个主题/分区或“读-处理-写”链路可用事务确保原子性（Kafka Streams/事务生产者）。

#### 常用检查
- kafka-consumer-groups.sh --describe 观察 lag 与已提交 offset；
- 关注重试/超时/幂等冲突日志；必要时使用幂等消费与 DLQ 配套。
</details>

### 4. 消费者的 offset 存在哪里？（字节/腾讯/陌陌）

<details>
<summary>答案与解析</summary> 

#### 存储位置
- 现代 Kafka（0.9+）将消费者位点存储在内部压缩主题 __consumer_offsets 中；键包含 group、topic、partition，值为 committed offset 与元数据；
- 该主题开启 compaction，周期性合并仅保留最新位点；保留时间由 offsets.retention.minutes 等控制；
- 早期版本（0.8 及更早）曾将位点存 ZooKeeper，现已淘汰。

#### 提交方式
- 自动提交：enable.auto.commit=true，按 auto.commit.interval.ms 周期提交（简单但易在再均衡/异常时产生重复或丢失风险）；
- 手动提交：enable.auto.commit=false，处理完成后 commitSync（或 commitAsync）提交更安全；
- 提交粒度：可按分区单独提交（精准控制）、或批量提交（减少开销）。

#### 观察与排查
- kafka-consumer-groups.sh --bootstrap-server ... --group `<g>` --describe 查看各分区 committed offset 与 lag；
- 发生位点错乱：先暂停分区消费，校验/回溯位点（seek），再恢复；
- __consumer_offsets 为 compacted topic，可用 kafka-dump-log.sh（线下）诊断历史提交记录。
</details>

### 5. 如何通过 offset 定位消息？（字节）

<details>
<summary>答案与解析</summary> 

#### 基于 offset 的定位
- Kafka 为每个分区维护单调递增的逻辑 offset；Broker 通过分段文件（.log）与稀疏 .index（相对 offset → 文件位置）快速定位；
- 当客户端指定 offset 读取时，Broker 先命中 .index 得到近似位置，再顺序扫描到精确消息并返回。

#### 客户端常用方式
- 编程 API：消费者先 assign/subscribe，然后对每个分区执行 seek(partition, offset) 后 poll；
- 指定时间转 offset：使用 ListOffsets/offsetsForTimes(timestamps) 先取时间对应起始 offset，再 seek；
- 命令行/工具：
  - kcat（kafkacat）读取指定 offset：kcat -b localhost:9092 -t orders -p 0 -o 12345 -c 10；
  - 查看日志段：kafka-dump-log.sh --files .../00000000000000000000.log --print-data-log（线下排查）。

#### 注意
- offset 仅在分区内有意义；跨分区无全局含义；
- 压缩（compact）主题仅保留最新同 key 版本，历史版本可能不可达；
- 被删除的过老 segment 中的 offset 将不可读取（受保留策略影响）。
</details>

### 6. 时间轮的原理（陌陌/顺丰）

<details>
<summary>答案与解析</summary> 

#### 核心思想
- Hashed Timing Wheel/分层时间轮：把时间轴离散为固定 tick 的槽（bucket），每个槽挂链表存放到期时间落在该槽的定时任务；
- 指针按 tick 步进，仅检查当前槽的任务；若任务过期时间不在当前轮，则记录 rounds（还需转几圈）或放入更高层轮；
- 摊还 O(1) 插入/取消/触发，适合海量定时器，避免最小堆 O(logN) 的竞争与抖动。

#### Kafka 中的应用
- 基于时间轮实现的延迟/超时调度（例如 DelayedOperationPurgatory）：
  - 生产请求超时、Fetch/事务超时、Group 协调器 Session 超时等；
  - 多层轮应对长/短多尺度超时，降低 CPU 与锁竞争。

#### 参数与取舍
- tickMs：时间精度；wheelSize：每轮槽数；levels：层数；
- 精度越高、槽越多内存越大；层数越多可覆盖更长时间跨度；
- 单槽任务过多可能产生“抖动”，可拆分负载或引入随机抖动。
</details>

### 7. 写入高性能的原因：sendfile 与 mmap 原理；为什么不用 splice（滴滴）

<details>
<summary>答案与解析</summary>

#### 为什么快：顺序写 + 零拷贝
- 顺序写入日志段：Producer 请求在 Broker 侧被批量追加到 .log，充分利用磁盘顺序写与 OS Page Cache；
- 零拷贝网络发送：Broker 使用 FileChannel.transferTo（底层常为 sendfile）将文件数据从内核页缓存直接送至 socket/NIC，避免用户态拷贝与上下文切换；
- 索引/元数据多采用内存映射（mmap），提升随机定位效率。

#### sendfile 与 mmap
- sendfile/transferTo 适合“文件 → socket”的大批量顺序传输（如 Broker 将日志发送给消费者）；
- mmap 适合频繁随机访问的小文件（如 .index/.timeindex），也可用于顺序读，但需注意页缺失与 TLB 压力；
- TLS 场景下传统 sendfile 可能退化（需要入用户态加密），可结合内核 TLS/卸载或允许明文内网传输以保持零拷贝路径。

#### 为什么不用 splice
- splice 是 Linux 特有的零拷贝接口，通常“文件→管道→socket”链路才可用，复杂度高；
- Java 标准库无稳定跨平台的 splice 封装，Kafka 采用的 FileChannel.transferTo 可在多平台获得稳定收益；
- 兼容性与可维护性权衡：splice 依赖内核版本/行为差异，收益相对 sendfile 并不显著。

#### 实战建议
- 批量发送（linger.ms/batch.size）与压缩（compression.type）提升吞吐；
- 控制页缓存与 I/O 调度：为日志磁盘单独挂盘；
- 避免 TLS 瓶颈：必要时使用内核 TLS 或网卡卸载，否则评估 CPU 开销；
- 监控：Broker 网络出站带宽、磁盘 I/O、请求队列延时、页缓存命中率。
</details>
---

## 网络

### 1. HTTPS 原理；TLS 握手需要几个 RTT？（滴滴/百度）

<details>
<summary>答案与解析</summary> 

#### 一、HTTPS 与 TLS 工作流程
- 解析 URL → DNS 解析（含缓存/hosts/SNI 影响）→ TCP 三次握手 → TLS 握手（协商版本/套件、证书验证、密钥交换、ALPN）→ 加密的 HTTP 请求/响应；
- 关键机制：
  - 证书链与信任（CA/中间证书/OCSP Stapling）；
  - ECDHE 前向安全（密钥泄露不影响历史流量）；
  - ALPN 协商 HTTP/1.1 或 HTTP/2（h2）；
  - HSTS 强制 HTTPS，避免降级攻击。

#### 二、TLS 握手需要几个 RTT？
- 只考虑 TLS 层：
  - TLS 1.2 完整握手：约 2 RTT；
  - TLS 1.3 完整握手：约 1 RTT；
  - 会话恢复：TLS 1.2（Session/Session Ticket）约 1 RTT；TLS 1.3 可 0-RTT（存在重放风险，谨慎用于幂等请求）。
- 若连同 TCP：需再加 1 个 RTT（TCP 三次握手）。

#### 三、常见问题
- 证书链不完整或 SNI 缺失导致握手失败；
- 时间误差影响证书有效性；
- 仅启用弱套件/旧版本导致安全告警或失败。

#### 四、命令/配置示例
- 查看 TLS/ALPN/证书：
```powershell
# Windows PowerShell
Resolve-DnsName example.com
Test-NetConnection example.com -Port 443 -InformationLevel Detailed
```
```bash
# OpenSSL：查看 TLS/ALPN 与证书链
openssl s_client -connect example.com:443 -servername example.com -tls1_3 -alpn h2 -state -msg < /dev/null | sed -n '1,50p'
```
```bash
# curl：检查 HTTP/2/TLS1.3
curl -I --http2 https://example.com
curl -I --tlsv1.3 https://example.com
```
- Nginx（启用 TLS1.2/1.3 与 HTTP/2）
```nginx
server {
  listen 443 ssl http2;
  server_name example.com;
  ssl_protocols TLSv1.2 TLSv1.3;
  ssl_ciphers HIGH:!aNULL:!MD5;
  ssl_certificate      /etc/ssl/certs/fullchain.pem;
  ssl_certificate_key  /etc/ssl/private/privkey.pem;
}
```
提示：TLS1.3 0-RTT 可能被重放，慎用于非幂等业务；合理开启 HSTS/OCSP Stapling 改善安全与时延。
</details>

### 2. 浏览器访问某个网址的详细过程；TCP 四次挥手（腾讯/滴滴）

<details>
<summary>答案与解析</summary> 

#### 一、浏览器访问流程（简化）
1) 输入 URL → 统一资源定位解析（协议/主机/端口/路径/参数/片段）
2) DNS 解析：缓存 → hosts → DoH/系统 DNS 递归解析，支持 SNI/ALPN 决策；
3) 建立连接：TCP 三次握手（可能含 TFO）→ TLS 握手（1RTT/2RTT/0RTT）；
4) 发起请求：构造请求行/头/体；HTTP/2 多路复用并依赖 HPACK/QPACK 压缩；
5) 服务端处理：负载均衡 → 反向代理 → 应用 → 存储/缓存；
6) 响应与渲染：状态码、头部（缓存、压缩、CSP、HSTS）、内容；浏览器解析构建 DOM/CSSOM/渲染树、JS 执行、回流重绘；
7) 复用与连接生命周期：HTTP/2 复用或 Keep-Alive，连接池管理。

常见优化：
- DNS 预解析、预连接（preconnect）、预加载（preload）、HTTP/2 push（慎用）；
- 缓存策略（强缓存/协商缓存）；压缩（gzip/br）；
- 前端性能：关键请求链、资源内联/分片、延迟加载、CRP 优化。

#### 二、TCP 四次挥手
1) 主动方发送 FIN（主动关闭）
2) 被动方回 ACK（进入 CLOSE_WAIT）
3) 被动方处理完后发送 FIN（进入 LAST_ACK）
4) 主动方回 ACK 并进入 TIME_WAIT，等待 2MSL 以确保对端收到并清理老包

关键参数与现象：
- TIME_WAIT 过多：短连接或高并发关闭；可优化端口复用/连接复用/延长 Keep-Alive；
- CLOSE_WAIT 堆积：应用未及时关闭 fd；
- RST 异常：半关闭处理不当或超时。

#### 命令示例
```powershell
# Windows 检查 TCP 连接
test-netconnection example.com -Port 443 -InformationLevel Detailed
Get-NetTCPConnection | ? { $_.RemotePort -eq 443 } | ft -AutoSize
```
```bash
# Linux 观察连接与 TIME_WAIT
ss -tanp | head
netstat -tan | head
```
```bash
# 抓包分析（需 root）
sudo tcpdump -i any -nn "host 93.184.216.34 and port 443" -vv -c 100
```
</details>

### 3. HTTP/2 与 QUIC 原理（字节）

<details>
<summary>答案与解析</summary> 

#### HTTP/2 核心
- 多路复用（单 TCP 连接多并发流），头部压缩（HPACK），二进制帧，服务器推送（已在部分浏览器弱化/弃用趋势），优先级与流量控制；
- 痛点：队头阻塞（HoL）仍存在于传输层（TCP 丢包导致整个连接受阻）。

#### QUIC/HTTP/3 核心
- 基于 UDP 的传输层协议，集成 TLS1.3，连接迁移（IP/端口变化不重建连接），更细粒度的流量控制与独立丢包恢复；
- 彻底解决 TCP 级别的队头阻塞：单连接多流，某流丢包不阻塞其他流；
- 快速握手：1-RTT/0-RTT；内建 FEC/拥塞控制可迭代升级（BBR 等）。

#### 对比与实践
- HTTP/2 适合链路稳定、时延较低的场景；
- HTTP/3/QUIC 在弱网/移动网络/跨运营商场景显著改善握手与丢包下的延迟；
- 负载均衡与观测：需支持 UDP/QUIC 透传与五元组保持；抓包分析从 tcpdump/tshark 转向 quic-aware 工具。

#### 命令/抓包
```bash
# curl 指定 HTTP/2 与 HTTP/3（curl 7.66+ & nghttp2/quiche 支持）
curl -I --http2 https://example.com
curl -I --http3 https://cloudflare-quic.com
```
```bash
# Wireshark 抓取 QUIC：选择协议 quic 与 http3 进行过滤；
# tshark 示例
sudo tshark -i any -f "udp port 443" -Y quic -c 50
```
```nginx
# Nginx/OpenResty（示意，HTTP/3 需支持 QUIC 的分支/构建）
server {
  listen 443 ssl http2;
  listen 443 quic reuseport;  # 需要带 quic 的构建
  add_header Alt-Svc 'h3=\":443\"; ma=86400';
}
```
</details>

---

## 分布式系统

### 1. 分布式事务怎么处理（高德/陌陌）

<details>
<summary>答案与解析</summary> 

#### 典型模式
- 2PC（两阶段提交）：协调者 Prepare/Commit；强一致但阻塞、单点与长事务风险；
- TCC：Try/Confirm/Cancel 三步显式资源预留与补偿；需要业务定义幂等与补偿接口；
- Saga：长事务拆分为有向步骤链条，每步成功则继续，失败触发逆向补偿；适合跨服务长链路；
- 本地消息表/Outbox + 最终一致：业务库写入与消息写入同本地事务，异步投递消息至 MQ；
- 事务消息（MQ）：生产者先半消息，确认后投递，消费者处理失败可重试与补偿。

#### 选型建议
- 强一致+短事务：优先 2PC/分布式数据库原生事务；
- 最终一致：优先 Outbox/本地消息表 或 Saga；跨服务资源耗时长用 TCC/Saga；
- 幂等与防重：所有侧写与回滚接口需具备幂等键（如业务唯一键/事务ID）。

#### 关键要点
- 幂等：去重表/唯一键、状态机、token/幂等键；
- 防悬挂/空回滚：Try 未达 Confirm 不应成功；Cancel 只对 Try 成功的资源；
- 超时/回查：协调者或事务管理器对长耗时步骤进行回查或超时回滚；
- 观测与补偿：失败事件记录、死信、人工兜底通道。

#### 示例
- Outbox（以订单→库存为例）：
  1) 本地事务同时插入订单与 outbox 表；
  2) 后台任务轮询 outbox 表发送到 MQ（幂等发送）；
  3) 库存服务消费并更新库存，成功后回写确认（或使用消费端幂等表）。

SQL 片段：
```sql
CREATE TABLE outbox (
  id BIGINT PRIMARY KEY,
  aggregate_type VARCHAR(50),
  aggregate_id VARCHAR(50),
  payload JSON,
  status TINYINT DEFAULT 0,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
-- 业务表与 outbox 同事务写入
```
</details>

### 2. 简述 Raft 原理（陌陌）

<details>
<summary>答案与解析</summary> 

#### 核心角色与术语
- 角色：Leader、Follower、Candidate；任期 term；
- 日志：index、term、commitIndex、lastApplied；
- 投票与心跳：RequestVote、AppendEntries。

#### 流程
- 领导选举：Follower 选举超时 → 成为 Candidate，自增 term，向多数派请求投票；收到多数票后成为 Leader；
- 日志复制：客户端请求经 Leader 追加日志，Leader 通过 AppendEntries 复制到 Follower；当条目存活于多数派即提交；
- 一致性保证：term/index 作为日志新旧判据；较新的日志拒绝投给落后的候选者；
- 安全性：日志匹配性质（Log Matching）、领导者完全性（Leader Completeness）。

#### 成员变更与快照
- Joint Consensus：过渡期用新老配置双多数派；
- 快照：压缩已提交日志，安装快照以截断过旧日志；
- 重启恢复：依赖持久化的当前任期、投票、日志与快照元数据。

#### 实践要点
- 选举超时随机化避免震荡；心跳间隔较短；
- 磁盘落盘：提交点持久化；
- 网络分区：Follower 落后可通过快照安装快速追上。

示意命令/伪代码：
```text
onElectionTimeout():
  becomeCandidate()
  currentTerm++
  voteFor=self
  send RequestVote to all

onReceiveMajorityVotes():
  becomeLeader()
  startHeartbeats()
```
</details>

### 3. 分布式 ID 的几种实现和优缺点（滴滴）

<details>
<summary>答案与解析</summary> 

#### 常见方案与优缺点
- UUID：无中心、冲突概率可忽略；v4 随机无序（索引/聚簇写入差），v1 暴露 MAC/时间；v7（基于时间）兼顾趋势递增与分布性，适合数据库主键；
- 数据库自增/序列：实现简单、强一致；单点瓶颈；可用“号段”（segment）批量分配减少 DB 压力；
- Snowflake（64bit 常见）：timeBits + datacenterId + workerId + sequence；高吞吐、趋势递增；需解决时钟回拨与 workerId 分配；
- Redis INCR：原子、高性能、易水平扩展（按业务 key 分片）；需要持久化与备份策略；
- ZooKeeper 递增序列：顺序节点天然单调；写性能有限，适用于低 QPS 关键序列；
- Leaf（美团开源）：segment（DB 号段）/snowflake 两种模式，工程化完善：缓存预热、双 buffer、监控与漂移检测。

#### 设计要点
- 唯一性与趋势递增：避免热点页/页分裂；
- 高可用：多机容错（worker 冲突检测、心跳租约）、漂移/回拨检测；
- 时钟回拨：等待下一个毫秒、启用备用 sequence 区间、或拒绝服务报警；
- 扩展性：多业务 namespace/前缀；多集群隔离；
- 位宽规划：例如 1 符号位 + 41 时间戳毫秒 + 5 机房 + 5 机器 + 12 序列。

#### 示例一：号段（Segment）分配（MySQL）
建表与初始化：
```sql
CREATE TABLE id_alloc (
  biz_tag VARCHAR(64) PRIMARY KEY,
  max_id BIGINT NOT NULL,
  step INT NOT NULL DEFAULT 1000,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
INSERT INTO id_alloc(biz_tag, max_id, step) VALUES ('order', 0, 1000)
  ON DUPLICATE KEY UPDATE biz_tag=biz_tag;
```
拉取一个号段并返回 [start, end]：
```sql
START TRANSACTION;
SELECT max_id, step FROM id_alloc WHERE biz_tag='order' FOR UPDATE;
-- 假设返回 max_id=10000, step=1000
UPDATE id_alloc SET max_id = max_id + step WHERE biz_tag='order';
COMMIT;
-- 应用持有可用区间 (10001, 11000]，内存发号，耗尽再拉取
```

#### 示例二：Redis INCR 限流与分片
```bash
# 按业务分片的自增 key，避免单点热点
redis-cli INCR id:order
redis-cli INCR id:order:shard:3
```
Lua 原子批量获取 N 个 ID：
```lua
-- KEYS[1]=id:order; ARGV[1]=n
local cur = redis.call('INCRBY', KEYS[1], ARGV[1])
local start = cur - ARGV[1] + 1
return {start, cur}
```

#### 示例三：Snowflake Java 精简实现
```java
public class SnowflakeId {
  private final long twepoch = 1672531200000L; // 2023-01-01
  private final long workerIdBits = 5L, datacenterIdBits = 5L, sequenceBits = 12L;
  private final long maxWorkerId = ~(-1L << workerIdBits);
  private final long maxDatacenterId = ~(-1L << datacenterIdBits);
  private final long workerIdShift = sequenceBits;
  private final long datacenterIdShift = sequenceBits + workerIdBits;
  private final long timestampLeftShift = sequenceBits + workerIdBits + datacenterIdBits;
  private final long sequenceMask = ~(-1L << sequenceBits);
  private long workerId, datacenterId, sequence = 0L, lastTs = -1L;
  public SnowflakeId(long workerId, long datacenterId) {
    if (workerId > maxWorkerId || workerId < 0) throw new IllegalArgumentException();
    if (datacenterId > maxDatacenterId || datacenterId < 0) throw new IllegalArgumentException();
    this.workerId = workerId; this.datacenterId = datacenterId;
  }
  public synchronized long nextId() {
    long ts = System.currentTimeMillis();
    if (ts < lastTs) throw new RuntimeException("clock moved backwards");
    if (ts == lastTs) sequence = (sequence + 1) & sequenceMask; else sequence = 0L;
    if (sequence == 0L && ts == lastTs) while ((ts = System.currentTimeMillis()) <= lastTs) {}
    lastTs = ts;
    return ((ts - twepoch) << timestampLeftShift)
      | (datacenterId << datacenterIdShift)
      | (workerId << workerIdShift)
      | sequence;
  }
}
```

实战建议：
- 优先选择“趋势递增 + 高吞吐 + 易观测”的方案：Leaf-segment 或 Snowflake（配合 NTP、漂移监控）；
- DB 方案务必“批量/号段”减少锁竞争；Redis 方案注意持久化与冷启动雪崩；
- 对外暴露十进制字符串可携带业务前缀以便排障，如 O20250911000001。
</details>

### 4. 降级、限流、熔断的实现原理（高德/陌陌）

<details>
<summary>答案与解析</summary> 

#### 概念
- 限流：限制请求速率/并发，保护系统稳定（令牌桶、漏桶、滑动窗口）。
- 降级：在部分功能不可用或高压时，提供简化/兜底返回（缓存、默认值、异步排队）。
- 熔断：下游不稳定/高错误率时，短路调用，快速失败，待恢复后再尝试。

#### 限流原理与实现
- 常见算法：
  - 令牌桶（Token Bucket）：按速率发放令牌，请求需拿到令牌；支持突发（burst）。
  - 漏桶（Leaky Bucket）：固定速率出水，平滑突发；易实现但对突发吸收差。
  - 固定窗口/滑动窗口：统计窗口内计数；滑动窗口粒度更细，精度高。
- 分层限流：入口网关/CDN、边缘 Nginx、服务网关、应用内部（按用户、IP、接口、租户、多维度）。
- Nginx 示例（基于客户端 IP 的 QPS 限制）：
```nginx
# http 块中：定义共享内存与限速
limit_req_zone $binary_remote_addr zone=ip_qps:10m rate=10r/s;
limit_conn_zone $binary_remote_addr zone=ip_con:10m;
server {
  listen 80;
  location /api/ {
    limit_req zone=ip_qps burst=20 nodelay;  # 突发 20
    limit_conn ip_con 50;                    # 同 IP 最多并发 50
    proxy_pass http://backend;
  }
}
```
- Redis 滑动窗口（ZSET）限流（原子 Lua）：
```lua
-- KEYS[1]=rl:{uid}, ARGV[1]=now_ms, ARGV[2]=window_ms, ARGV[3]=limit
redis.call('ZREMRANGEBYSCORE', KEYS[1], 0, ARGV[1]-ARGV[2])
local c = redis.call('ZCARD', KEYS[1])
if c < tonumber(ARGV[3]) then
  redis.call('ZADD', KEYS[1], ARGV[1], ARGV[1])
  redis.call('PEXPIRE', KEYS[1], ARGV[2])
  return 1
else
  return 0
end
```
- Go 令牌桶中间件：
```go
import (
  "net/http"
  "golang.org/x/time/rate"
)
var limiter = rate.NewLimiter(20, 40) // 20 rps, burst 40
func middleware(next http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    if !limiter.Allow() {
      w.WriteHeader(http.StatusTooManyRequests)
      w.Write([]byte("429 Too Many Requests"))
      return
    }
    next.ServeHTTP(w, r)
  })
}
```

#### 熔断原理与实现
- 状态机：Closed（正常统计错误率/慢调用率）→ Open（短路一段时间）→ Half-Open（探测少量请求）。
- 触发条件：错误率阈值、慢调用比例、异常类型；恢复策略：冷却时间后半开探测成功比例达标转 Closed。
- Java Resilience4j 配置与使用：
```yaml
resilience4j.circuitbreaker:
  instances:
    backendA:
      slidingWindowType: COUNT_BASED
      slidingWindowSize: 100
      failureRateThreshold: 50
      slowCallRateThreshold: 50
      slowCallDurationThreshold: 2s
      permittedNumberOfCallsInHalfOpenState: 5
      waitDurationInOpenState: 10s
```
```java
CircuitBreaker cb = CircuitBreakerRegistry.ofDefaults().circuitBreaker("backendA");
Supplier<String> supplier = CircuitBreaker.decorateSupplier(cb, () -> remoteCall());
Try<String> result = Try.ofSupplier(supplier)
  .recover(ex -> fallback());
```
- Go gobreaker：
```go
st := gobreaker.Settings{Name: "downstream", Interval: time.Minute, Timeout: 10 * time.Second}
cb := gobreaker.NewCircuitBreaker(st)
res, err := cb.Execute(func() (interface{}, error) { return call(), nil })
```

#### 降级策略与示例
- 主动降级触发：高错误率/高延迟、依赖不可用、队列堆积、手动开关（开关放在配置中心）。
- 兜底返回：
  - 静态/缓存内容：热点接口返回缓存或“服务繁忙”；
  - 读降级：只读路径保留，写入改异步；
  - 功能降级：关闭非核心功能（推荐、统计）。
- Nginx stale 缓存示例：
```nginx
proxy_cache mycache;
proxy_cache_use_stale error timeout http_500 http_502 http_504 updating;
```
- 应用侧兜底：在熔断/超时异常捕获后返回默认值/缓存。

#### 压测与观测
- 压测命令：
```bash
wrk -t4 -c200 -d30s http://localhost/api/
# 或 ab
ab -n 5000 -c 200 http://localhost/api/
```
- 验证：观察 429/503 比例、p99 时延、下游错误率；
- 监控：QPS、限流命中、熔断状态、降级开关、队列堆积；
- 告警：连续熔断、错误率阈值、限流率接近上限。

实战建议：
- 入口层“粗粒度”限流 + 应用内“细粒度”限流；
- 所有降级/熔断/限流决策输出埋点，保留灰度开关；
- 与重试协同：限流返回应避免客户端无脑重试（指数退避、抖动）。
</details>

---

## 其他

### 1. 布隆过滤器的实现原理和使用场景（滴滴）

<details>
<summary>答案与解析</summary> 

#### 一、布隆过滤器原理
布隆过滤器（Bloom Filter）是一种**空间高效的概率型数据结构**，用于快速判断一个元素是否**可能存在**于集合中。核心特点：
- **一定不存在**：如果布隆过滤器返回 false，元素一定不在集合中；
- **可能存在**：如果返回 true，元素可能在集合中（存在误判率）；
- **只能添加，不能删除**：传统布隆过滤器不支持删除操作。

##### 1. 数据结构
- **位数组**：长度为 m 的二进制数组，初始全为 0；
- **哈希函数**：k 个独立的哈希函数 h1, h2, ..., hk；
- **误判率**：p ≈ (1 - e^(-kn/m))^k，其中 n 为插入元素数量。

##### 2. 操作流程
**添加元素**：
1. 对元素 x 计算 k 个哈希值：h1(x), h2(x), ..., hk(x)；
2. 将位数组中对应位置设为 1：bits[h1(x) % m] = 1, bits[h2(x) % m] = 1, ...；

**查询元素**：
1. 对元素 x 计算 k 个哈希值；
2. 检查位数组对应位置：如果任一位置为 0，则元素一定不存在；如果全为 1，则可能存在。

#### 二、参数选择与优化
##### 1. 最优参数公式
- **最优哈希函数数量**：k = (m/n) * ln(2) ≈ 0.693 * (m/n)；
- **最优位数组长度**：m = -n * ln(p) / (ln(2))^2；
- **实际误判率**：p = (1 - e^(-kn/m))^k。

##### 2. 实践建议
- **误判率 1%**：每个元素需约 9.6 位，k=7；
- **误判率 0.1%**：每个元素需约 14.4 位，k=10；
- **内存占用**：100万元素，1% 误判率约需 1.2MB。

#### 三、使用场景
##### 1. 缓存穿透防护
```php
<?php
// Redis + 布隆过滤器防止缓存穿透
class BloomCacheGuard {
    private $redis;
    private $bloomKey;
    
    public function __construct($redis, $bloomKey = 'bf:cache') {
        $this->redis = $redis;
        $this->bloomKey = $bloomKey;
        // 初始化布隆过滤器（需 RedisBloom 模块）
        $this->redis->rawCommand('BF.RESERVE', $this->bloomKey, 0.01, 1000000);
    }
    
    public function get($key, $loader) {
        // 1. 布隆过滤器快速判断
        if (!$this->redis->rawCommand('BF.EXISTS', $this->bloomKey, $key)) {
            return null; // 一定不存在，直接返回
        }
        
        // 2. 查询缓存
        $cached = $this->redis->get("cache:$key");
        if ($cached !== false) {
            return json_decode($cached, true);
        }
        
        // 3. 查询数据库
        $data = $loader($key);
        if ($data !== null) {
            $this->redis->setex("cache:$key", 3600, json_encode($data));
            $this->redis->rawCommand('BF.ADD', $this->bloomKey, $key);
        }
        
        return $data;
    }
}

// 使用示例
$guard = new BloomCacheGuard($redis);
$user = $guard->get("user:1001", function($key) {
    return getUserFromDB(substr($key, 5)); // 去掉 "user:" 前缀
});
?>
```

##### 2. 大数据去重
```bash
# 使用 Redis 布隆过滤器处理大规模 URL 去重
BF.RESERVE bf:urls 0.001 10000000  # 0.1% 误判率，1000万容量

# 批量检查 URL 是否已爬取
BF.MEXISTS bf:urls url1 url2 url3

# 添加新 URL
BF.MADD bf:urls new_url1 new_url2
```

##### 3. 分布式系统中的预过滤
```go
// Go 实现简单布隆过滤器
package main

import (
    "hash/fnv"
    "math"
)

type BloomFilter struct {
    bits []bool
    m    uint // 位数组长度
    k    uint // 哈希函数数量
}

func NewBloomFilter(n uint, p float64) *BloomFilter {
    m := uint(-float64(n) * math.Log(p) / (math.Log(2) * math.Log(2)))
    k := uint(float64(m) / float64(n) * math.Log(2))
    return &BloomFilter{
        bits: make([]bool, m),
        m:    m,
        k:    k,
    }
}

func (bf *BloomFilter) hash(data []byte, seed uint) uint {
    h := fnv.New32a()
    h.Write(data)
    return uint(h.Sum32()+seed*0x9e3779b9) % bf.m
}

func (bf *BloomFilter) Add(data []byte) {
    for i := uint(0); i < bf.k; i++ {
        bf.bits[bf.hash(data, i)] = true
    }
}

func (bf *BloomFilter) Test(data []byte) bool {
    for i := uint(0); i < bf.k; i++ {
        if !bf.bits[bf.hash(data, i)] {
            return false
        }
    }
    return true
}

// 使用示例：分布式爬虫 URL 去重
func main() {
    bf := NewBloomFilter(1000000, 0.01) // 100万 URL，1% 误判率
    
    urls := []string{"http://example.com", "http://test.com"}
    for _, url := range urls {
        if !bf.Test([]byte(url)) {
            bf.Add([]byte(url))
            // 爬取新 URL
            crawl(url)
        }
    }
}
```

#### 四、变种与扩展
##### 1. 计数布隆过滤器（Counting Bloom Filter）
- **支持删除**：用计数器替代位数组，删除时计数器减 1；
- **内存开销**：每个位置需 3-4 位计数器，内存增加 3-4 倍；
- **适用场景**：需要动态删除元素的场景。

##### 2. 布谷鸟过滤器（Cuckoo Filter）
- **支持删除**：基于布谷鸟哈希，存储指纹而非位数组；
- **更低误判率**：相同内存下误判率更低；
- **适用场景**：需要删除且对误判率要求严格。

#### 五、实践注意事项
##### 1. 容量规划
- **预估数据量**：根据业务增长预留 2-3 倍容量；
- **误判率权衡**：缓存场景可接受 1-5%，安全场景需 < 0.1%；
- **内存监控**：位数组占用内存 = m / 8 字节。

##### 2. 哈希函数选择
- **独立性**：使用不同种子的同一哈希函数（如 MurmurHash + 种子）；
- **均匀性**：避免哈希碰撞导致的热点位置；
- **性能**：选择计算快速的哈希函数（避免 MD5/SHA1）。

##### 3. 监控与告警
- **误判率监控**：实际误判率 vs 理论值，超出阈值告警；
- **容量监控**：已插入元素数量 vs 设计容量；
- **性能监控**：查询 QPS、平均响应时间。

#### 六、典型应用案例
1. **Google Chrome**：恶意 URL 检测，本地布隆过滤器预过滤；
2. **Cassandra**：SSTable 查询优化，避免无效磁盘读取；
3. **HBase**：RegionServer 中过滤不存在的行键；
4. **CDN**：热点内容预判，减少回源请求；
5. **爬虫系统**：URL 去重，避免重复爬取。

**总结**：布隆过滤器以极小的内存代价提供快速的存在性判断，是解决大规模数据预过滤问题的经典工具。关键在于合理设置参数平衡内存、性能和误判率。
</details>

### 2. 进程间通信的方式（腾讯）

<details>
<summary>答案与解析</summary> 

#### 一、进程间通信（IPC）概述
进程间通信是指运行在不同进程中的程序之间交换数据的机制。由于进程拥有独立的内存空间，无法直接访问其他进程的数据，需要通过操作系统提供的 IPC 机制实现数据交换。

#### 二、主要 IPC 方式
##### 1. 管道（Pipe）
**匿名管道（Anonymous Pipe）**：
- **原理**：内核提供的缓冲区，数据单向流动（FIFO）；
- **特点**：只能在父子进程间使用，数据读取后即消失；
- **容量限制**：Linux 默认 64KB（可通过 `/proc/sys/fs/pipe-max-size` 调整）。

```bash
# Shell 中的管道示例
ls -l | grep ".txt" | wc -l

# 等价的 C 代码逻辑
int pipefd[2];
pipe(pipefd);  // 创建管道
if (fork() == 0) {
    // 子进程：写入数据
    close(pipefd[0]);  // 关闭读端
    write(pipefd[1], "hello", 5);
    close(pipefd[1]);
} else {
    // 父进程：读取数据
    close(pipefd[1]);  // 关闭写端
    char buf[10];
    read(pipefd[0], buf, 10);
    close(pipefd[0]);
}
```

**命名管道（Named Pipe/FIFO）**：
- **原理**：文件系统中的特殊文件，支持无关进程通信；
- **创建**：`mkfifo /tmp/mypipe`；
- **使用**：一个进程写入，另一个进程读取。

```bash
# 终端 1：创建并写入
mkfifo /tmp/chat
echo "Hello from process 1" > /tmp/chat

# 终端 2：读取
cat /tmp/chat
```

##### 2. 消息队列（Message Queue）
- **原理**：内核维护的消息链表，支持按类型发送/接收；
- **特点**：消息有边界，支持优先级，持久化存储；
- **限制**：消息大小限制（Linux 默认 8KB），队列数量限制。

```c
// System V 消息队列示例
#include <sys/msg.h>

struct msgbuf {
    long mtype;     // 消息类型
    char mtext[100]; // 消息内容
};

// 发送进程
int msgid = msgget(1234, IPC_CREAT | 0666);
struct msgbuf msg = {1, "Hello World"};
msgsnd(msgid, &msg, sizeof(msg.mtext), 0);

// 接收进程
struct msgbuf recv_msg;
msgrcv(msgid, &recv_msg, sizeof(recv_msg.mtext), 1, 0);
printf("Received: %s\n", recv_msg.mtext);
```

##### 3. 共享内存（Shared Memory）
- **原理**：多个进程映射同一块物理内存，直接读写；
- **优势**：最快的 IPC 方式，无需数据拷贝；
- **缺点**：需要同步机制（信号量/互斥锁）防止竞态条件。

```c
// System V 共享内存示例
#include <sys/shm.h>
#include <sys/sem.h>

// 创建共享内存
int shmid = shmget(1234, 1024, IPC_CREAT | 0666);
char *shmaddr = (char*)shmat(shmid, NULL, 0);

// 写入数据（需要同步）
sem_wait(semid);  // P 操作
strcpy(shmaddr, "Shared data");
sem_post(semid);  // V 操作

// 分离共享内存
shmdt(shmaddr);
```

##### 4. 信号量（Semaphore）
- **原理**：计数器，用于进程同步和互斥；
- **操作**：P（wait/down）减 1，V（signal/up）加 1；
- **类型**：二进制信号量（互斥锁）、计数信号量（资源池）。

```c
// POSIX 信号量示例
#include <semaphore.h>

sem_t *sem = sem_open("/mysem", O_CREAT, 0644, 1);

// 临界区保护
sem_wait(sem);    // 获取锁
// 临界区代码
sem_post(sem);    // 释放锁

sem_close(sem);
```

##### 5. 信号（Signal）
- **原理**：异步通知机制，内核向进程发送软件中断；
- **特点**：轻量级，但信息量有限（只能传递信号类型）；
- **常用信号**：SIGTERM（终止）、SIGKILL（强制杀死）、SIGUSR1/SIGUSR2（用户自定义）。

```c
// 信号处理示例
#include <signal.h>

void signal_handler(int sig) {
    printf("Received signal %d\n", sig);
}

// 注册信号处理函数
signal(SIGUSR1, signal_handler);

// 发送信号
kill(target_pid, SIGUSR1);
```

##### 6. 套接字（Socket）
**本地套接字（Unix Domain Socket）**：
- **原理**：基于文件系统的套接字，进程间高效通信；
- **类型**：SOCK_STREAM（可靠）、SOCK_DGRAM（不可靠）；
- **优势**：支持文件描述符传递。

```c
// Unix Domain Socket 示例
#include <sys/socket.h>
#include <sys/un.h>

// 服务端
int server_fd = socket(AF_UNIX, SOCK_STREAM, 0);
struct sockaddr_un addr;
addr.sun_family = AF_UNIX;
strcpy(addr.sun_path, "/tmp/socket");
bind(server_fd, (struct sockaddr*)&addr, sizeof(addr));
listen(server_fd, 5);

// 客户端
int client_fd = socket(AF_UNIX, SOCK_STREAM, 0);
connect(client_fd, (struct sockaddr*)&addr, sizeof(addr));
```

**网络套接字（Network Socket）**：
- **原理**：基于 TCP/UDP 协议的网络通信；
- **适用**：跨主机进程通信、分布式系统；
- **示例**：HTTP 服务、RPC 调用。

##### 7. 内存映射文件（Memory-Mapped File）
- **原理**：将文件映射到进程虚拟内存，多进程可映射同一文件；
- **优势**：大文件高效访问，操作系统自动同步；
- **适用**：大数据共享、数据库存储引擎。

```c
// mmap 示例
#include <sys/mman.h>

int fd = open("data.txt", O_RDWR);
char *mapped = mmap(NULL, 1024, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);

// 直接操作内存
mapped[0] = 'A';

// 同步到文件
msync(mapped, 1024, MS_SYNC);
munmap(mapped, 1024);
```

#### 三、性能对比与选择
##### 1. 性能排序（从快到慢）
1. **共享内存** > **内存映射文件** > **Unix Domain Socket** > **命名管道** > **消息队列** > **网络套接字**
2. **信号量**：同步原语，不传输数据；
3. **信号**：异步通知，信息量最少。

##### 2. 选择建议
| 场景 | 推荐方式 | 原因 |
|------|----------|------|
| 父子进程简单通信 | 匿名管道 | 简单易用，自动清理 |
| 无关进程少量数据 | 命名管道/消息队列 | 有边界，易于解析 |
| 大量数据高频交换 | 共享内存 + 信号量 | 零拷贝，性能最高 |
| 跨网络通信 | 网络套接字 | 支持分布式 |
| 异步通知 | 信号 | 轻量级，响应快 |
| 大文件共享 | 内存映射文件 | 操作系统优化，支持持久化 |

#### 四、实际应用案例
##### 1. Web 服务器架构
```bash
# Nginx 多进程模型
# Master 进程通过信号管理 Worker 进程
kill -HUP nginx_master_pid    # 重新加载配置
kill -QUIT nginx_worker_pid   # 优雅关闭 Worker

# Worker 进程间通过共享内存共享统计信息
# 使用 Unix Domain Socket 与上游服务通信
```

##### 2. 数据库系统
```sql
-- MySQL 使用多种 IPC 方式：
-- 1. 共享内存：Buffer Pool、日志缓冲区
-- 2. 信号量：锁管理、线程同步
-- 3. 套接字：客户端连接（TCP/Unix Socket）
-- 4. 内存映射：数据文件、索引文件
```

##### 3. PHP 应用示例
```php
<?php
// PHP 中使用 System V IPC

// 1. 消息队列
$queue = msg_get_queue(1234);
msg_send($queue, 1, "Hello from PHP");
msg_receive($queue, 1, $msgtype, 1024, $message);

// 2. 共享内存
$shm = shm_attach(1234, 1024);
shm_put_var($shm, 1, "Shared data");
$data = shm_get_var($shm, 1);
shm_detach($shm);

// 3. 信号量
$sem = sem_get(1234);
sem_acquire($sem);
// 临界区操作
sem_release($sem);

// 4. Unix Socket（推荐用于 PHP-FPM）
$socket = socket_create(AF_UNIX, SOCK_STREAM, 0);
socket_connect($socket, '/tmp/php.sock');
socket_write($socket, "Request data");
$response = socket_read($socket, 1024);
socket_close($socket);
?>
```

#### 五、注意事项与最佳实践
##### 1. 资源管理
- **及时清理**：避免 IPC 资源泄漏（如未删除的消息队列、共享内存）；
- **权限控制**：设置合适的访问权限（0644、0666 等）；
- **容量限制**：关注系统限制（`/proc/sys/kernel/` 下的参数）。

##### 2. 错误处理
- **检查返回值**：所有 IPC 调用都可能失败；
- **信号中断**：慢系统调用可能被信号中断（EINTR）；
- **死锁预防**：使用信号量时注意获取顺序。

##### 3. 调试与监控
```bash
# 查看 IPC 资源使用情况
ipcs -a                    # 显示所有 IPC 资源
ipcs -m                    # 显示共享内存
ipcs -q                    # 显示消息队列
ipcs -s                    # 显示信号量

# 清理僵尸 IPC 资源
ipcrm -m shmid            # 删除共享内存
ipcrm -q msgid            # 删除消息队列
ipcrm -s semid            # 删除信号量
```

**总结**：选择合适的 IPC 方式需要综合考虑性能要求、数据量大小、进程关系、跨平台需求等因素。共享内存适合高性能场景，套接字适合灵活通信，管道适合简单数据流，信号适合异步通知。
</details>

### 3. 进程/线程/协程的区别（滴滴/知乎）

<details>
<summary>答案与解析</summary> 

#### 一、基本概念对比
| 维度 | 进程（Process） | 线程（Thread） | 协程（Coroutine） |
|------|----------------|----------------|------------------|
| **定义** | 操作系统资源分配的基本单位 | CPU 调度的基本单位 | 用户态轻量级线程 |
| **内存空间** | 独立的虚拟地址空间 | 共享进程地址空间 | 共享线程地址空间 |
| **创建开销** | 最大（MB 级别） | 中等（KB 级别） | 最小（字节级别） |
| **切换开销** | 最大（用户态↔内核态） | 中等（内核态调度） | 最小（用户态调度） |
| **通信方式** | IPC（管道、共享内存等） | 共享内存、锁 | 直接访问变量 |
| **调度方式** | 抢占式（操作系统） | 抢占式（操作系统） | 协作式（程序控制） |
| **并发数量** | 受限（通常几十到几百） | 受限（通常几千） | 很高（可达百万级） |

#### 二、进程（Process）详解
##### 1. 特点与优势
- **隔离性强**：进程间内存完全隔离，一个进程崩溃不影响其他进程；
- **安全性高**：恶意代码难以跨进程攻击；
- **稳定性好**：适合长时间运行的服务。

##### 2. 缺点与限制
- **资源消耗大**：每个进程需要独立的页表、文件描述符表等；
- **通信复杂**：需要通过 IPC 机制，有额外开销；
- **创建慢**：fork() 需要复制父进程资源。

##### 3. 适用场景
```bash
# Web 服务器多进程模型（如 Apache prefork）
# Master 进程管理，Worker 进程处理请求
ps aux | grep apache
# apache  1234  master process
# apache  1235  worker process
# apache  1236  worker process

# PHP-FPM 进程池
ps aux | grep php-fpm
# php-fpm: master process
# php-fpm: pool www, child 1
# php-fpm: pool www, child 2
```

#### 三、线程（Thread）详解
##### 1. 特点与优势
- **资源共享**：同一进程内线程共享代码段、数据段、堆内存；
- **创建快速**：只需分配栈空间和寄存器上下文；
- **通信简单**：通过共享内存直接通信。

##### 2. 缺点与风险
- **同步复杂**：需要锁、信号量等同步机制防止竞态条件；
- **调试困难**：多线程 bug 难以重现和定位；
- **一损俱损**：一个线程崩溃可能导致整个进程终止。

##### 3. 线程模型
```c
// POSIX 线程示例
#include <pthread.h>
#include <stdio.h>

void* worker_thread(void* arg) {
    int thread_id = *(int*)arg;
    printf("Thread %d is running\n", thread_id);
    
    // 模拟工作
    for (int i = 0; i < 1000000; i++) {
        // CPU 密集型任务
    }
    
    return NULL;
}

int main() {
    pthread_t threads[4];
    int thread_ids[4];
    
    // 创建 4 个工作线程
    for (int i = 0; i < 4; i++) {
        thread_ids[i] = i;
        pthread_create(&threads[i], NULL, worker_thread, &thread_ids[i]);
    }
    
    // 等待所有线程完成
    for (int i = 0; i < 4; i++) {
        pthread_join(threads[i], NULL);
    }
    
    return 0;
}
```

#### 四、协程（Coroutine）详解
##### 1. 特点与优势
- **用户态调度**：无需内核参与，切换开销极小；
- **高并发**：单线程可支持百万级协程；
- **同步编程模型**：避免回调地狱，代码逻辑清晰。

##### 2. 实现原理
- **栈切换**：保存/恢复寄存器状态和栈指针；
- **事件驱动**：基于 epoll/kqueue 等 I/O 多路复用；
- **调度器**：用户态调度器管理协程的创建、挂起、恢复。

##### 3. 协程示例
**Go 语言协程（Goroutine）**：
```go
package main

import (
    "fmt"
    "net/http"
    "time"
)

// 协程处理 HTTP 请求
func handleRequest(url string, ch chan<- string) {
    start := time.Now()
    resp, err := http.Get(url)
    if err != nil {
        ch <- fmt.Sprintf("%s: Error - %v", url, err)
        return
    }
    defer resp.Body.Close()
    
    duration := time.Since(start)
    ch <- fmt.Sprintf("%s: %s (%v)", url, resp.Status, duration)
}

func main() {
    urls := []string{
        "http://google.com",
        "http://github.com",
        "http://stackoverflow.com",
    }
    
    ch := make(chan string)
    
    // 启动多个协程并发请求
    for _, url := range urls {
        go handleRequest(url, ch) // 启动协程
    }
    
    // 收集结果
    for i := 0; i < len(urls); i++ {
        fmt.Println(<-ch)
    }
}
```

**PHP 协程（Swoole）**：
```php
<?php
use Swoole\Coroutine;
use Swoole\Coroutine\Http\Client;

// 启用协程
Co\run(function () {
    $urls = [
        'httpbin.org',
        'github.com',
        'google.com'
    ];
    
    $results = [];
    
    // 并发发起 HTTP 请求
    foreach ($urls as $url) {
        Coroutine::create(function () use ($url, &$results) {
            $client = new Client($url, 80);
            $client->get('/');
            
            $results[] = [
                'url' => $url,
                'status' => $client->statusCode,
                'length' => strlen($client->body)
            ];
            
            $client->close();
        });
    }
    
    // 等待所有协程完成
    Coroutine::sleep(2);
    
    foreach ($results as $result) {
        echo "URL: {$result['url']}, Status: {$result['status']}, Length: {$result['length']}\n";
    }
});
?>
```

**Python 协程（asyncio）**：
```python
import asyncio
import aiohttp
import time

async def fetch_url(session, url):
    """协程函数：获取单个 URL"""
    start = time.time()
    try:
        async with session.get(url) as response:
            content = await response.text()
            duration = time.time() - start
            return f"{url}: {response.status} ({duration:.2f}s)"
    except Exception as e:
        return f"{url}: Error - {e}"

async def main():
    """主协程：并发处理多个请求"""
    urls = [
        'http://httpbin.org/delay/1',
        'http://httpbin.org/delay/2',
        'http://httpbin.org/delay/3'
    ]
    
    async with aiohttp.ClientSession() as session:
        # 创建协程任务
        tasks = [fetch_url(session, url) for url in urls]
        
        # 并发执行所有任务
        results = await asyncio.gather(*tasks)
        
        for result in results:
            print(result)

# 运行协程
if __name__ == '__main__':
    asyncio.run(main())
```

#### 五、性能对比实测
##### 1. 创建开销对比
```bash
# 进程创建测试（1000 个进程）
time for i in {1..1000}; do /bin/true & done; wait
# real: ~2.5s, 内存占用: ~500MB

# 线程创建测试（1000 个线程）
# C 程序创建 1000 个线程
# real: ~0.8s, 内存占用: ~100MB

# 协程创建测试（100万个协程）
# Go 程序创建 100万个 goroutine
# real: ~0.1s, 内存占用: ~50MB
```

##### 2. 上下文切换开销
| 类型 | 切换时间 | 主要开销 |
|------|----------|----------|
| 进程 | 1-10μs | 页表切换、TLB 刷新、缓存失效 |
| 线程 | 0.1-1μs | 寄存器保存/恢复、栈切换 |
| 协程 | 10-100ns | 用户态栈切换、寄存器保存 |

#### 六、选择建议与最佳实践
##### 1. 使用场景选择
| 场景 | 推荐方案 | 理由 |
|------|----------|------|
| **CPU 密集型** | 多进程 | 充分利用多核，避免 GIL 限制 |
| **I/O 密集型** | 协程 | 高并发，低资源消耗 |
| **混合型负载** | 线程池 | 平衡性能和复杂度 |
| **高可靠性服务** | 多进程 | 故障隔离，稳定性好 |
| **高并发网络服务** | 协程 | 支持百万连接 |

##### 2. 架构设计模式
**多进程 + 协程混合模式**：
```python
# Gunicorn + FastAPI 架构
# gunicorn_config.py
workers = 4              # 4 个进程（CPU 核数）
worker_class = "uvicorn.workers.UvicornWorker"  # 协程 Worker
worker_connections = 1000  # 每个进程支持 1000 个并发连接

# 启动命令
# gunicorn -c gunicorn_config.py main:app
```

**线程池 + 异步 I/O**：
```java
// Java CompletableFuture + 线程池
ExecutorService executor = Executors.newFixedThreadPool(10);

List<CompletableFuture<String>> futures = urls.stream()
    .map(url -> CompletableFuture.supplyAsync(() -> {
        // HTTP 请求逻辑
        return httpGet(url);
    }, executor))
    .collect(Collectors.toList());

// 等待所有任务完成
CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
    .join();
```

##### 3. 注意事项
**进程间通信**：
- 选择合适的 IPC 机制（共享内存 > Unix Socket > 管道）；
- 注意序列化/反序列化开销；
- 考虑数据一致性和同步问题。

**线程安全**：
- 使用线程安全的数据结构（如 Java ConcurrentHashMap）；
- 避免共享可变状态，优先使用不可变对象；
- 合理使用锁，避免死锁和性能瓶颈。

**协程调度**：
- 避免在协程中执行阻塞操作（如同步 I/O、sleep）；
- 使用协程安全的库和框架；
- 注意协程泄漏和内存管理。

#### 七、实际应用案例
##### 1. Web 服务器架构演进
```
# Apache (多进程)
Prefork MPM: 每个请求一个进程
优点: 稳定性好，进程隔离
缺点: 内存消耗大，并发数受限

# Nginx (事件驱动)
Event-driven: 单进程多路复用
优点: 高并发，低内存
缺点: CPU 密集型任务性能差

# Node.js (单线程事件循环)
Event Loop + 协程
优点: 高并发 I/O，开发简单
缺点: CPU 密集型任务阻塞

# Go (协程池)
Goroutine + Channel
优点: 高并发，简单并发模型
缺点: 内存占用相对较高
```

##### 2. 数据库连接池设计
```php
<?php
// PHP 多进程 + 连接池
class DatabasePool {
    private $processes = [];
    private $connections_per_process = 10;
    
    public function __construct($process_count = 4) {
        for ($i = 0; $i < $process_count; $i++) {
            $pid = pcntl_fork();
            if ($pid == 0) {
                // 子进程：维护数据库连接池
                $this->worker_process($i);
                exit;
            } else {
                $this->processes[] = $pid;
            }
        }
    }
    
    private function worker_process($worker_id) {
        // 每个进程维护独立的连接池
        $connections = [];
        for ($i = 0; $i < $this->connections_per_process; $i++) {
            $connections[] = new PDO($dsn, $user, $pass);
        }
        
        // 处理请求...
    }
}
?>
```

**总结**：进程适合稳定性要求高的场景，线程适合 CPU 密集型任务，协程适合高并发 I/O 场景。现代应用通常采用混合架构，在不同层次使用不同的并发模型以达到最佳性能。
</details>

### 4. LVS 原理，如何保证高可用（滴滴）

<details>
<summary>答案与解析</summary> 

#### 一、LVS（Linux Virtual Server）概述
LVS 是基于 Linux 内核的**四层负载均衡器**，工作在传输层（TCP/UDP），通过修改数据包的目标地址实现请求分发。相比七层负载均衡（如 Nginx），LVS 性能更高但功能相对简单。

#### 二、LVS 核心组件
##### 1. 架构组成
- **Director Server（调度器）**：接收客户端请求，根据调度算法选择 Real Server；
- **Real Server（真实服务器）**：实际处理业务请求的后端服务器；
- **VIP（Virtual IP）**：对外提供服务的虚拟 IP 地址；
- **DIP（Director IP）**：调度器的内网 IP；
- **RIP（Real Server IP）**：后端服务器的 IP。

##### 2. 工作模式对比
| 模式 | NAT | TUN | DR |
|------|-----|-----|----|
| **全称** | Network Address Translation | IP Tunneling | Direct Routing |
| **数据流向** | 请求/响应都经过 Director | 请求经过，响应直达客户端 | 请求经过，响应直达客户端 |
| **网络要求** | Director 双网卡 | 支持 IP 隧道 | 同一网段 |
| **性能** | 低（瓶颈在 Director） | 中等 | 高（无响应瓶颈） |
| **配置复杂度** | 简单 | 中等 | 复杂 |
| **适用场景** | 小规模，简单部署 | 跨网段，中等规模 | 大规模，高性能 |

#### 三、三种工作模式详解
##### 1. NAT 模式（Network Address Translation）
**工作原理**：
1. 客户端请求发送到 VIP；
2. Director 修改数据包目标 IP 为 RIP，源 IP 保持不变；
3. Real Server 处理请求，响应发送给 Director；
4. Director 修改响应包源 IP 为 VIP，转发给客户端。

**配置示例**：
```bash
# Director Server 配置
# 1. 开启 IP 转发
echo 1 > /proc/sys/net/ipv4/ip_forward

# 2. 配置 VIP
ip addr add 192.168.1.100/24 dev eth0

# 3. 添加 LVS 规则
ipvsadm -A -t 192.168.1.100:80 -s rr
ipvsadm -a -t 192.168.1.100:80 -r 192.168.1.10:80 -m -w 1
ipvsadm -a -t 192.168.1.100:80 -r 192.168.1.11:80 -m -w 1

# 4. 配置 iptables NAT 规则
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -j MASQUERADE

# Real Server 配置（设置网关为 Director）
route add default gw 192.168.1.1  # Director 的内网 IP
```

##### 2. TUN 模式（IP Tunneling）
**工作原理**：
1. Director 将请求包封装在新的 IP 包中（IP-in-IP）；
2. Real Server 解封装后处理请求；
3. Real Server 直接响应客户端（不经过 Director）。

**配置示例**：
```bash
# Director Server 配置
ipvsadm -A -t 192.168.1.100:80 -s rr
ipvsadm -a -t 192.168.1.100:80 -r 192.168.1.10:80 -i -w 1
ipvsadm -a -t 192.168.1.100:80 -r 192.168.1.11:80 -i -w 1

# Real Server 配置
# 1. 创建 tunl0 接口
modprobe ipip
ip tunnel add tunl0 mode ipip remote 192.168.1.1 local 192.168.1.10
ip link set tunl0 up
ip addr add 192.168.1.100/32 dev tunl0

# 2. 配置路由
echo 1 > /proc/sys/net/ipv4/conf/tunl0/arp_ignore
echo 2 > /proc/sys/net/ipv4/conf/tunl0/arp_announce
```

##### 3. DR 模式（Direct Routing）
**工作原理**：
1. Director 和 Real Server 在同一网段；
2. Director 修改数据包的 MAC 地址为目标 Real Server；
3. Real Server 直接响应客户端。

**配置示例**：
```bash
# Director Server 配置
# 1. 配置 VIP 到网卡
ip addr add 192.168.1.100/24 dev eth0

# 2. 添加 LVS 规则
ipvsadm -A -t 192.168.1.100:80 -s rr
ipvsadm -a -t 192.168.1.100:80 -r 192.168.1.10:80 -g -w 1
ipvsadm -a -t 192.168.1.100:80 -r 192.168.1.11:80 -g -w 1

# Real Server 配置
# 1. 在 lo 接口配置 VIP（避免 ARP 冲突）
ip addr add 192.168.1.100/32 dev lo

# 2. 抑制 ARP 响应
echo 1 > /proc/sys/net/ipv4/conf/lo/arp_ignore
echo 2 > /proc/sys/net/ipv4/conf/lo/arp_announce
echo 1 > /proc/sys/net/ipv4/conf/all/arp_ignore
echo 2 > /proc/sys/net/ipv4/conf/all/arp_announce
```

#### 四、调度算法
##### 1. 静态调度算法
```bash
# 轮询（Round Robin）
ipvsadm -A -t VIP:PORT -s rr

# 加权轮询（Weighted Round Robin）
ipvsadm -A -t VIP:PORT -s wrr
ipvsadm -a -t VIP:PORT -r RIP1:PORT -g -w 3  # 权重 3
ipvsadm -a -t VIP:PORT -r RIP2:PORT -g -w 1  # 权重 1

# 最少连接（Least Connection）
ipvsadm -A -t VIP:PORT -s lc

# 加权最少连接（Weighted Least Connection）
ipvsadm -A -t VIP:PORT -s wlc

# 源地址哈希（Source Hash）
ipvsadm -A -t VIP:PORT -s sh

# 目标地址哈希（Destination Hash）
ipvsadm -A -t VIP:PORT -s dh
```

##### 2. 动态调度算法
```bash
# 最短期望延迟（Shortest Expected Delay）
ipvsadm -A -t VIP:PORT -s sed

# 永不排队（Never Queue）
ipvsadm -A -t VIP:PORT -s nq

# 基于局部性的最少连接（Locality-Based Least Connection）
ipvsadm -A -t VIP:PORT -s lblc

# 带复制的基于局部性最少连接（LBLC with Replication）
ipvsadm -A -t VIP:PORT -s lblcr
```

#### 五、高可用方案
##### 1. Keepalived + LVS 双机热备
**架构设计**：
```
客户端
   |
   v
[VIP] 192.168.1.100
   |
   +-- Master Director (Keepalived + LVS)
   +-- Backup Director (Keepalived + LVS)
   |
   v
[Real Servers]
192.168.1.10, 192.168.1.11, 192.168.1.12
```

**Keepalived 配置**：
```bash
# Master Director (/etc/keepalived/keepalived.conf)
vrrp_script chk_lvs {
    script "/etc/keepalived/check_lvs.sh"
    interval 2
    weight -2
    fall 3
    rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.1.100/24
    }
    track_script {
        chk_lvs
    }
    notify_master "/etc/keepalived/lvs_master.sh"
    notify_backup "/etc/keepalived/lvs_backup.sh"
}

# Backup Director 配置类似，但 state 为 BACKUP，priority 为 90
```

**健康检查脚本**：
```bash
#!/bin/bash
# /etc/keepalived/check_lvs.sh

# 检查 LVS 服务是否正常
if ! pgrep ipvsadm > /dev/null; then
    exit 1
fi

# 检查 VIP 是否可达
if ! ip addr show | grep -q "192.168.1.100"; then
    exit 1
fi

# 检查后端服务器健康状态
for server in 192.168.1.10 192.168.1.11 192.168.1.12; do
    if ! nc -z $server 80 2>/dev/null; then
        # 移除不健康的服务器
        ipvsadm -d -t 192.168.1.100:80 -r $server:80
    else
        # 添加健康的服务器
        ipvsadm -a -t 192.168.1.100:80 -r $server:80 -g -w 1 2>/dev/null
    fi
done

exit 0
```

##### 2. 多级负载均衡架构
```
[DNS 轮询]
     |
     v
[地域 LVS 集群]
     |
     +-- 华北 LVS (VIP1)
     +-- 华东 LVS (VIP2)
     +-- 华南 LVS (VIP3)
     |
     v
[应用层负载均衡]
     |
     +-- Nginx/HAProxy 集群
     |
     v
[应用服务器集群]
```

##### 3. 容器化 LVS 高可用
```yaml
# Docker Compose 示例
version: '3.8'
services:
  lvs-master:
    image: lvs-keepalived:latest
    network_mode: host
    privileged: true
    volumes:
      - ./keepalived-master.conf:/etc/keepalived/keepalived.conf
      - ./lvs-scripts:/etc/keepalived/scripts
    environment:
      - ROLE=MASTER
      - VIP=192.168.1.100
      - PRIORITY=100

  lvs-backup:
    image: lvs-keepalived:latest
    network_mode: host
    privileged: true
    volumes:
      - ./keepalived-backup.conf:/etc/keepalived/keepalived.conf
      - ./lvs-scripts:/etc/keepalived/scripts
    environment:
      - ROLE=BACKUP
      - VIP=192.168.1.100
      - PRIORITY=90
```

#### 六、性能优化与监控
##### 1. 内核参数优化
```bash
# /etc/sysctl.conf
# 增加连接跟踪表大小
net.netfilter.nf_conntrack_max = 1048576
net.netfilter.nf_conntrack_tcp_timeout_established = 300

# 优化网络缓冲区
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 65536 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216

# 启用 TCP 窗口缩放
net.ipv4.tcp_window_scaling = 1

# 优化 TIME_WAIT 状态
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 30

# 应用配置
sysctl -p
```

##### 2. LVS 性能监控
```bash
# 查看 LVS 状态
ipvsadm -L -n --stats
ipvsadm -L -n --rate

# 监控脚本
#!/bin/bash
# lvs_monitor.sh
while true; do
    echo "=== $(date) ==="
    echo "Active Connections:"
    ipvsadm -L -n | grep -E "TCP|UDP" | awk '{print $2, $5}'
    
    echo "Connection Stats:"
    ipvsadm -L -n --stats | grep -v "IP Virtual Server"
    
    echo "Rate Stats:"
    ipvsadm -L -n --rate | grep -v "IP Virtual Server"
    
    echo "System Load:"
    uptime
    
    sleep 10
done
```

##### 3. 告警与自动化
```python
#!/usr/bin/env python3
# lvs_health_check.py
import subprocess
import requests
import time
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def check_lvs_service():
    """检查 LVS 服务状态"""
    try:
        result = subprocess.run(['ipvsadm', '-L', '-n'], 
                              capture_output=True, text=True)
        return result.returncode == 0
    except Exception as e:
        logger.error(f"LVS check failed: {e}")
        return False

def check_real_servers():
    """检查后端服务器健康状态"""
    servers = ['192.168.1.10', '192.168.1.11', '192.168.1.12']
    healthy_servers = []
    
    for server in servers:
        try:
            response = requests.get(f'http://{server}/health', timeout=5)
            if response.status_code == 200:
                healthy_servers.append(server)
            else:
                logger.warning(f"Server {server} returned {response.status_code}")
        except Exception as e:
            logger.error(f"Server {server} health check failed: {e}")
    
    return healthy_servers

def send_alert(message):
    """发送告警"""
    # 集成钉钉、企业微信等告警系统
    webhook_url = "https://oapi.dingtalk.com/robot/send?access_token=xxx"
    payload = {
        "msgtype": "text",
        "text": {"content": f"LVS Alert: {message}"}
    }
    try:
        requests.post(webhook_url, json=payload, timeout=10)
    except Exception as e:
        logger.error(f"Alert send failed: {e}")

def main():
    while True:
        # 检查 LVS 服务
        if not check_lvs_service():
            send_alert("LVS service is down!")
        
        # 检查后端服务器
        healthy_servers = check_real_servers()
        if len(healthy_servers) < 2:
            send_alert(f"Only {len(healthy_servers)} servers are healthy")
        
        time.sleep(30)

if __name__ == '__main__':
    main()
```

#### 七、故障处理与最佳实践
##### 1. 常见故障排查
```bash
# 1. 检查 VIP 是否正确配置
ip addr show | grep VIP

# 2. 检查 LVS 规则
ipvsadm -L -n

# 3. 检查连接状态
ss -tuln | grep :80

# 4. 检查 ARP 表
arp -a | grep VIP

# 5. 抓包分析
tcpdump -i eth0 host VIP -n

# 6. 检查内核模块
lsmod | grep ip_vs

# 7. 查看系统日志
tail -f /var/log/messages | grep -i lvs
```

##### 2. 最佳实践
1. **网络规划**：
   - DR 模式下确保 Director 和 Real Server 在同一网段；
   - 合理规划 VLAN，避免广播风暴；
   - 配置合适的 MTU 值。

2. **安全加固**：
   - 限制管理接口访问（iptables 规则）；
   - 定期更新系统和 LVS 版本；
   - 监控异常连接和攻击行为。

3. **容量规划**：
   - 根据业务峰值配置足够的 Real Server；
   - 预留 30-50% 的性能余量；
   - 定期进行压力测试。

4. **运维自动化**：
   - 使用 Ansible/Puppet 管理配置；
   - 集成监控告警系统；
   - 建立故障自愈机制。

**总结**：LVS 通过内核级负载均衡提供高性能的四层分发能力，结合 Keepalived 实现高可用。DR 模式性能最佳但配置复杂，NAT 模式简单但性能受限。生产环境需要完善的监控、告警和自动化运维体系。
</details>

### 5. 502 / 504 的原因及处理（滴滴/百度/腾讯/顺丰）

<details>
<summary>答案与解析</summary> 

#### 一、502 与 504 错误概述
**502 Bad Gateway** 和 **504 Gateway Timeout** 都是网关错误，但原因不同：
- **502**：网关从上游服务器接收到**无效响应**
- **504**：网关等待上游服务器响应**超时**

#### 二、502 错误详解
##### 1. 常见原因
```bash
# 1. 后端服务未启动
sudo systemctl status nginx
sudo systemctl status php-fpm
ps aux | grep php-fpm

# 2. 端口未监听
netstat -tlnp | grep :9000
ss -tlnp | grep :8080

# 3. 权限问题
ls -la /var/run/php/
chmod 666 /var/run/php/php7.4-fpm.sock

# 4. 配置错误
nginx -t
php-fpm -t
```

##### 2. Nginx + PHP-FPM 场景
```nginx
# 常见错误配置
server {
    location ~ \.php$ {
        # 错误：端口不匹配
        fastcgi_pass 127.0.0.1:9001;  # PHP-FPM 实际监听 9000
        
        # 错误：socket 路径错误
        fastcgi_pass unix:/wrong/path/php-fpm.sock;
        
        # 错误：缺少必要参数
        # fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}

# 正确配置
server {
    location ~ \.php$ {
        fastcgi_pass 127.0.0.1:9000;
        # 或使用 socket
        # fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
        
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
        
        # 超时设置
        fastcgi_connect_timeout 300;
        fastcgi_send_timeout 300;
        fastcgi_read_timeout 300;
    }
}
```

##### 3. PHP-FPM 配置优化
```ini
; /etc/php/7.4/fpm/pool.d/www.conf
[www]
user = www-data
group = www-data

; 监听配置
listen = 127.0.0.1:9000
; 或使用 socket（性能更好）
; listen = /var/run/php/php7.4-fpm.sock
; listen.owner = www-data
; listen.group = www-data
; listen.mode = 0660

; 进程管理
pm = dynamic
pm.max_children = 50
pm.start_servers = 5
pm.min_spare_servers = 5
pm.max_spare_servers = 35
pm.max_requests = 500

; 超时设置
request_terminate_timeout = 300

; 日志配置
php_admin_value[error_log] = /var/log/php-fpm/www-error.log
php_admin_flag[log_errors] = on
```

##### 4. 应用层问题排查
```php
<?php
// PHP 应用常见 502 问题

// 1. 致命错误导致进程退出
ini_set('display_errors', 0);  // 生产环境关闭错误显示
ini_set('log_errors', 1);
ini_set('error_log', '/var/log/php/error.log');

// 2. 内存耗尽
ini_set('memory_limit', '256M');

// 3. 数据库连接失败
try {
    $pdo = new PDO($dsn, $username, $password, [
        PDO::ATTR_TIMEOUT => 5,
        PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION
    ]);
} catch (PDOException $e) {
    error_log("DB connection failed: " . $e->getMessage());
    http_response_code(503);  // 返回 503 而不是让 Nginx 返回 502
    exit('Service Unavailable');
}

// 4. 异常处理
set_exception_handler(function($exception) {
    error_log("Uncaught exception: " . $exception->getMessage());
    http_response_code(500);
    exit('Internal Server Error');
});

// 5. 优雅处理信号
pcntl_signal(SIGTERM, function() {
    // 清理资源
    exit(0);
});
?>
```

#### 三、504 错误详解
##### 1. 超时配置优化
```nginx
# Nginx 超时配置
http {
    # 客户端超时
    client_body_timeout 60s;
    client_header_timeout 60s;
    send_timeout 60s;
    
    # 代理超时
    proxy_connect_timeout 30s;
    proxy_send_timeout 60s;
    proxy_read_timeout 60s;
    
    # FastCGI 超时
    fastcgi_connect_timeout 30s;
    fastcgi_send_timeout 300s;
    fastcgi_read_timeout 300s;
    
    server {
        location / {
            proxy_pass http://backend;
            
            # 针对性超时设置
            proxy_connect_timeout 10s;
            proxy_send_timeout 30s;
            proxy_read_timeout 120s;  # 长时间处理的接口
            
            # 超时重试
            proxy_next_upstream timeout;
            proxy_next_upstream_tries 2;
            proxy_next_upstream_timeout 20s;
        }
        
        # 长时间任务单独处理
        location /api/export {
            proxy_pass http://backend;
            proxy_read_timeout 600s;  # 10分钟
            proxy_send_timeout 600s;
        }
    }
}
```

##### 2. 应用层超时处理
```python
# Python Flask 应用超时处理
from flask import Flask, request, jsonify
import signal
import time
from functools import wraps

app = Flask(__name__)

class TimeoutError(Exception):
    pass

def timeout_handler(signum, frame):
    raise TimeoutError("Request timeout")

def with_timeout(seconds):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # 设置超时信号
            signal.signal(signal.SIGALRM, timeout_handler)
            signal.alarm(seconds)
            
            try:
                result = func(*args, **kwargs)
                signal.alarm(0)  # 取消超时
                return result
            except TimeoutError:
                return jsonify({'error': 'Request timeout'}), 504
            finally:
                signal.alarm(0)
        return wrapper
    return decorator

@app.route('/api/slow-task')
@with_timeout(30)  # 30秒超时
def slow_task():
    # 模拟长时间任务
    time.sleep(25)
    return jsonify({'status': 'completed'})

# 异步任务处理
from celery import Celery

celery = Celery('app')

@app.route('/api/async-task')
def async_task():
    task = long_running_task.delay()
    return jsonify({
        'task_id': task.id,
        'status': 'processing',
        'check_url': f'/api/task-status/{task.id}'
    }), 202

@celery.task
def long_running_task():
    # 长时间任务在后台执行
    time.sleep(300)
    return {'result': 'completed'}

@app.route('/api/task-status/<task_id>')
def task_status(task_id):
    task = long_running_task.AsyncResult(task_id)
    if task.state == 'PENDING':
        return jsonify({'status': 'processing'})
    elif task.state == 'SUCCESS':
        return jsonify({'status': 'completed', 'result': task.result})
    else:
        return jsonify({'status': 'failed', 'error': str(task.info)})
```

##### 3. 数据库查询优化
```sql
-- 慢查询优化
-- 1. 添加索引
CREATE INDEX idx_user_created_at ON users(created_at);
CREATE INDEX idx_order_status_date ON orders(status, created_at);

-- 2. 查询优化
-- 避免全表扫描
SELECT * FROM users WHERE created_at > '2023-01-01' LIMIT 1000;

-- 使用分页
SELECT * FROM orders 
WHERE id > 1000000 
ORDER BY id 
LIMIT 100;

-- 3. 分区表
CREATE TABLE orders_2023 PARTITION OF orders 
FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');
```

```php
<?php
// PHP 数据库查询超时处理
class DatabaseManager {
    private $pdo;
    
    public function __construct($dsn, $username, $password) {
        $options = [
            PDO::ATTR_TIMEOUT => 30,  // 连接超时
            PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
            PDO::MYSQL_ATTR_READ_TIMEOUT => 60,  // 读取超时
        ];
        
        $this->pdo = new PDO($dsn, $username, $password, $options);
    }
    
    public function queryWithTimeout($sql, $params = [], $timeout = 30) {
        // 设置查询超时
        $this->pdo->setAttribute(PDO::MYSQL_ATTR_READ_TIMEOUT, $timeout);
        
        try {
            $stmt = $this->pdo->prepare($sql);
            $stmt->execute($params);
            return $stmt->fetchAll(PDO::FETCH_ASSOC);
        } catch (PDOException $e) {
            if (strpos($e->getMessage(), 'timeout') !== false) {
                throw new Exception('Database query timeout', 504);
            }
            throw $e;
        }
    }
    
    public function asyncQuery($sql, $params = []) {
        // 使用队列处理长时间查询
        $jobId = uniqid();
        
        // 添加到队列
        $this->addToQueue([
            'id' => $jobId,
            'sql' => $sql,
            'params' => $params,
            'created_at' => time()
        ]);
        
        return $jobId;
    }
}
?>
```

#### 四、监控与诊断
##### 1. 实时监控脚本
```bash
#!/bin/bash
# gateway_monitor.sh - 网关错误监控

LOG_FILE="/var/log/nginx/access.log"
ERROR_LOG="/var/log/nginx/error.log"
ALERT_THRESHOLD_502=10
ALERT_THRESHOLD_504=5

monitor_errors() {
    local timeframe="5 minutes ago"
    local timestamp=$(date -d "$timeframe" '+%d/%b/%Y:%H:%M')
    
    # 统计 502 错误
    local count_502=$(awk -v ts="$timestamp" '$4 > "["ts && $9 == "502" {count++} END {print count+0}' "$LOG_FILE")
    
    # 统计 504 错误
    local count_504=$(awk -v ts="$timestamp" '$4 > "["ts && $9 == "504" {count++} END {print count+0}' "$LOG_FILE")
    
    echo "$(date) - 502 errors: $count_502, 504 errors: $count_504"
    
    # 502 错误告警
    if [ "$count_502" -gt "$ALERT_THRESHOLD_502" ]; then
        echo "🚨 HIGH 502 ERROR RATE: $count_502 errors in 5 minutes"
        
        # 检查后端服务状态
        echo "Checking backend services..."
        systemctl is-active php-fpm || echo "PHP-FPM is not running"
        netstat -tlnp | grep :9000 || echo "Port 9000 not listening"
        
        # 显示最近的错误日志
        echo "Recent error logs:"
        tail -5 "$ERROR_LOG"
    fi
    
    # 504 错误告警
    if [ "$count_504" -gt "$ALERT_THRESHOLD_504" ]; then
        echo "🚨 HIGH 504 ERROR RATE: $count_504 errors in 5 minutes"
        
        # 检查系统负载
        echo "System load: $(uptime | awk -F'load average:' '{print $2}')"
        
        # 检查慢查询
        echo "Recent slow queries:"
        tail -5 /var/log/mysql/mysql-slow.log 2>/dev/null || echo "No slow query log"
    fi
}

# 主循环
while true; do
    monitor_errors
    sleep 60
done
```

##### 2. 性能分析工具
```bash
#!/bin/bash
# performance_analysis.sh - 性能分析脚本

echo "=== Gateway Performance Analysis ==="
echo "Timestamp: $(date)"
echo

# 1. 系统资源使用情况
echo "1. System Resources:"
echo "Memory: $(free -h | grep Mem | awk '{print $3"/"$2}')"
echo "CPU Load: $(uptime | awk -F'load average:' '{print $2}')"
echo "Disk I/O: $(iostat -x 1 1 | tail -n +4 | awk 'NR>1{print $1, $10"%"}')"
echo

# 2. 网络连接状态
echo "2. Network Connections:"
echo "ESTABLISHED: $(ss -t state established | wc -l)"
echo "TIME_WAIT: $(ss -t state time-wait | wc -l)"
echo "CLOSE_WAIT: $(ss -t state close-wait | wc -l)"
echo

# 3. Nginx 状态
echo "3. Nginx Status:"
if command -v curl >/dev/null; then
    curl -s http://localhost/nginx_status 2>/dev/null || echo "Nginx status not available"
fi
echo

# 4. PHP-FPM 状态
echo "4. PHP-FPM Status:"
if command -v curl >/dev/null; then
    curl -s http://localhost/fpm_status 2>/dev/null || echo "PHP-FPM status not available"
fi
echo

# 5. 数据库连接
echo "5. Database Connections:"
mysql -e "SHOW PROCESSLIST;" 2>/dev/null | wc -l || echo "Database not accessible"
echo

# 6. 最近的错误统计
echo "6. Recent Error Statistics (last hour):"
if [ -f "/var/log/nginx/access.log" ]; then
    awk -v hour_ago="$(date -d '1 hour ago' '+%d/%b/%Y:%H')" '
        $4 > "["hour_ago {
            status[$9]++
        }
        END {
            for (s in status) {
                if (s >= 400) print "HTTP " s ": " status[s]
            }
        }' /var/log/nginx/access.log
fi
```

#### 五、解决方案与最佳实践
##### 1. 架构优化
```yaml
# Docker Compose 高可用架构
version: '3.8'
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - app1
      - app2
    restart: unless-stopped
    
  app1:
    image: myapp:latest
    environment:
      - DB_HOST=db
      - REDIS_HOST=redis
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped
    
  app2:
    image: myapp:latest
    environment:
      - DB_HOST=db
      - REDIS_HOST=redis
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped
    
  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: password
    volumes:
      - db_data:/var/lib/mysql
    restart: unless-stopped
    
  redis:
    image: redis:alpine
    restart: unless-stopped
    
volumes:
  db_data:
```

##### 2. 负载均衡配置
```nginx
# nginx.conf - 负载均衡与故障转移
upstream backend {
    # 健康检查
    server app1:8080 max_fails=3 fail_timeout=30s;
    server app2:8080 max_fails=3 fail_timeout=30s;
    server app3:8080 max_fails=3 fail_timeout=30s backup;
    
    # 连接保持
    keepalive 32;
    keepalive_requests 100;
    keepalive_timeout 60s;
}

server {
    listen 80;
    
    # 健康检查端点
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
    
    location / {
        proxy_pass http://backend;
        
        # 请求头设置
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # 超时设置
        proxy_connect_timeout 10s;
        proxy_send_timeout 30s;
        proxy_read_timeout 60s;
        
        # 故障转移
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
        proxy_next_upstream_tries 2;
        proxy_next_upstream_timeout 30s;
        
        # 缓冲设置
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
        
        # 自定义错误页面
        error_page 502 503 504 /50x.html;
    }
    
    location = /50x.html {
        root /usr/share/nginx/html;
        internal;
    }
}
```

##### 3. 自动化恢复
```python
#!/usr/bin/env python3
# auto_recovery.py - 自动恢复系统

import subprocess
import requests
import time
import logging
from datetime import datetime

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class GatewayMonitor:
    def __init__(self):
        self.services = ['nginx', 'php-fpm', 'mysql']
        self.endpoints = [
            'http://localhost/health',
            'http://localhost:8080/health'
        ]
        self.max_retries = 3
        self.retry_interval = 30
    
    def check_service(self, service):
        """检查系统服务状态"""
        try:
            result = subprocess.run(
                ['systemctl', 'is-active', service],
                capture_output=True, text=True
            )
            return result.returncode == 0
        except Exception as e:
            logger.error(f"Failed to check {service}: {e}")
            return False
    
    def check_endpoint(self, url):
        """检查HTTP端点健康状态"""
        try:
            response = requests.get(url, timeout=10)
            return response.status_code == 200
        except Exception as e:
            logger.error(f"Failed to check {url}: {e}")
            return False
    
    def restart_service(self, service):
        """重启服务"""
        try:
            subprocess.run(['systemctl', 'restart', service], check=True)
            logger.info(f"Restarted {service} successfully")
            return True
        except subprocess.CalledProcessError as e:
            logger.error(f"Failed to restart {service}: {e}")
            return False
    
    def send_alert(self, message):
        """发送告警"""
        webhook_url = "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
        payload = {
            "text": f"🚨 Gateway Alert: {message}",
            "username": "Gateway Monitor"
        }
        
        try:
            requests.post(webhook_url, json=payload, timeout=10)
        except Exception as e:
            logger.error(f"Failed to send alert: {e}")
    
    def monitor_and_recover(self):
        """主监控循环"""
        while True:
            try:
                # 检查系统服务
                for service in self.services:
                    if not self.check_service(service):
                        logger.warning(f"{service} is not running")
                        
                        # 尝试重启
                        for attempt in range(self.max_retries):
                            logger.info(f"Attempting to restart {service} (attempt {attempt + 1})")
                            
                            if self.restart_service(service):
                                time.sleep(10)  # 等待服务启动
                                if self.check_service(service):
                                    logger.info(f"{service} recovered successfully")
                                    break
                            
                            if attempt == self.max_retries - 1:
                                self.send_alert(f"Failed to recover {service} after {self.max_retries} attempts")
                            
                            time.sleep(self.retry_interval)
                
                # 检查HTTP端点
                for endpoint in self.endpoints:
                    if not self.check_endpoint(endpoint):
                        logger.warning(f"Endpoint {endpoint} is not healthy")
                        self.send_alert(f"Endpoint {endpoint} health check failed")
                
                time.sleep(60)  # 每分钟检查一次
                
            except KeyboardInterrupt:
                logger.info("Monitor stopped by user")
                break
            except Exception as e:
                logger.error(f"Monitor error: {e}")
                time.sleep(60)

if __name__ == '__main__':
    monitor = GatewayMonitor()
    monitor.monitor_and_recover()
```

**总结**：502/504 错误主要由服务不可用和超时引起。解决方案包括：优化配置、增加超时时间、实施负载均衡、建立监控告警、自动化恢复机制。关键是建立完善的监控体系和故障自愈能力，确保服务高可用。
</details>

### 6. 两个相同的玻璃球，求 100 层楼的临界层（腾讯）

<details>
<summary>答案与解析</summary> 

#### 一、问题描述
有两个相同的玻璃球，需要确定在 100 层楼中，从第几层开始扔下玻璃球会摔碎（临界层）。要求用**最少的尝试次数**找到这个临界层。

#### 二、问题分析
##### 1. 约束条件
- 有且仅有 **2 个玻璃球**
- 楼层范围：**1-100 层**
- 玻璃球特性：
  - 如果在第 N 层摔碎，那么在 N 层以上都会摔碎
  - 如果在第 N 层没摔碎，那么在 N 层以下都不会摔碎
  - 摔碎的球无法再使用

##### 2. 策略思考
- **暴力策略**：从第 1 层开始逐层测试，最坏情况需要 100 次
- **二分策略**：如果只有 1 个球不可行（摔碎后无法继续）
- **优化策略**：需要平衡第一个球的测试间隔

#### 三、最优解法：数学推导
##### 1. 核心思想
设第一个球每次上升的楼层数为递减序列，使得最坏情况下的总尝试次数最小。

##### 2. 数学模型
假设最优解需要 **n** 次尝试：
- 第一个球最多扔 **n** 次
- 第二个球最多扔 **n-1** 次

第一个球的测试楼层序列：
- 第 1 次：第 **n** 层
- 第 2 次：第 **n + (n-1)** 层
- 第 3 次：第 **n + (n-1) + (n-2)** 层
- ...
- 第 k 次：第 **n + (n-1) + ... + (n-k+1)** 层

##### 3. 求解过程
需要满足：**n + (n-1) + (n-2) + ... + 1 ≥ 100**

即：**n(n+1)/2 ≥ 100**

解得：**n² + n ≥ 200**

通过计算：
- n = 13: 13² + 13 = 169 + 13 = 182 < 200 ❌
- n = 14: 14² + 14 = 196 + 14 = 210 ≥ 200 ✅

因此，**最少需要 14 次尝试**。

##### 4. 最优策略
第一个球的测试楼层：
- 第 1 次：第 **14** 层
- 第 2 次：第 **14 + 13 = 27** 层
- 第 3 次：第 **27 + 12 = 39** 层
- 第 4 次：第 **39 + 11 = 50** 层
- 第 5 次：第 **50 + 10 = 60** 层
- 第 6 次：第 **60 + 9 = 69** 层
- 第 7 次：第 **69 + 8 = 77** 层
- 第 8 次：第 **77 + 7 = 84** 层
- 第 9 次：第 **84 + 6 = 90** 层
- 第 10 次：第 **90 + 5 = 95** 层
- 第 11 次：第 **95 + 4 = 99** 层
- 第 12 次：第 **99 + 3 = 102** 层（超过100，实际测试第100层）

#### 四、算法实现
##### 1. Python 实现
```python
def find_critical_floor_optimal(total_floors=100):
    """
    使用最优策略找到临界层
    返回：(策略序列, 最大尝试次数)
    """
    import math
    
    # 计算最少尝试次数
    n = math.ceil((-1 + math.sqrt(1 + 8 * total_floors)) / 2)
    
    # 生成第一个球的测试序列
    test_floors = []
    current_floor = n
    step = n - 1
    
    while current_floor <= total_floors:
        test_floors.append(current_floor)
        current_floor += step
        step -= 1
        if step <= 0:
            break
    
    return test_floors, n

def simulate_test(critical_floor, strategy):
    """
    模拟测试过程
    """
    attempts = 0
    ball1_broken = False
    last_safe_floor = 0
    
    print(f"临界层设定为: {critical_floor}")
    print(f"测试策略: {strategy}")
    print("\n开始测试:")
    
    # 第一个球的测试
    for floor in strategy:
        attempts += 1
        print(f"尝试 {attempts}: 第一个球从第 {floor} 层扔下", end="")
        
        if floor >= critical_floor:
            print(" -> 摔碎了！")
            ball1_broken = True
            break
        else:
            print(" -> 没摔碎")
            last_safe_floor = floor
    
    # 第二个球的测试
    if ball1_broken:
        print(f"\n第一个球摔碎，开始用第二个球从第 {last_safe_floor + 1} 层逐层测试:")
        for floor in range(last_safe_floor + 1, floor + 1):
            attempts += 1
            print(f"尝试 {attempts}: 第二个球从第 {floor} 层扔下", end="")
            
            if floor >= critical_floor:
                print(" -> 摔碎了！")
                print(f"\n找到临界层: {floor}")
                break
            else:
                print(" -> 没摔碎")
    else:
        print(f"\n第一个球没有摔碎，临界层在 {strategy[-1]} 层以上")
    
    print(f"总尝试次数: {attempts}")
    return attempts

# 测试不同的临界层
def test_all_scenarios():
    strategy, max_attempts = find_critical_floor_optimal(100)
    print(f"最优策略: {strategy}")
    print(f"理论最大尝试次数: {max_attempts}")
    print("\n" + "="*50)
    
    # 测试几个关键场景
    test_cases = [1, 14, 27, 50, 84, 99, 100]
    
    for critical in test_cases:
        print(f"\n测试场景 - 临界层: {critical}")
        print("-" * 30)
        actual_attempts = simulate_test(critical, strategy)
        print(f"实际尝试次数: {actual_attempts}")
        print()

if __name__ == "__main__":
    test_all_scenarios()
```

##### 2. JavaScript 实现
```javascript
class GlassBallSolver {
    constructor(totalFloors = 100) {
        this.totalFloors = totalFloors;
        this.strategy = this.calculateOptimalStrategy();
    }
    
    calculateOptimalStrategy() {
        // 计算最少尝试次数
        const n = Math.ceil((-1 + Math.sqrt(1 + 8 * this.totalFloors)) / 2);
        
        // 生成测试序列
        const testFloors = [];
        let currentFloor = n;
        let step = n - 1;
        
        while (currentFloor <= this.totalFloors) {
            testFloors.push(currentFloor);
            currentFloor += step;
            step--;
            if (step <= 0) break;
        }
        
        return { floors: testFloors, maxAttempts: n };
    }
    
    simulate(criticalFloor) {
        let attempts = 0;
        let ball1Broken = false;
        let lastSafeFloor = 0;
        const log = [];
        
        log.push(`临界层: ${criticalFloor}`);
        log.push(`策略: [${this.strategy.floors.join(', ')}]`);
        
        // 第一个球测试
        for (const floor of this.strategy.floors) {
            attempts++;
            const result = floor >= criticalFloor ? '摔碎' : '安全';
            log.push(`尝试 ${attempts}: 球1 从 ${floor} 层 -> ${result}`);
            
            if (floor >= criticalFloor) {
                ball1Broken = true;
                break;
            }
            lastSafeFloor = floor;
        }
        
        // 第二个球测试
        if (ball1Broken) {
            const startFloor = lastSafeFloor + 1;
            const endFloor = this.strategy.floors.find(f => f >= criticalFloor) || this.totalFloors;
            
            for (let floor = startFloor; floor <= endFloor; floor++) {
                attempts++;
                const result = floor >= criticalFloor ? '摔碎' : '安全';
                log.push(`尝试 ${attempts}: 球2 从 ${floor} 层 -> ${result}`);
                
                if (floor >= criticalFloor) {
                    log.push(`找到临界层: ${floor}`);
                    break;
                }
            }
        }
        
        log.push(`总尝试次数: ${attempts}`);
        return { attempts, log };
    }
    
    testAllScenarios() {
        console.log('=== 玻璃球问题最优解 ===');
        console.log(`策略: [${this.strategy.floors.join(', ')}]`);
        console.log(`最大尝试次数: ${this.strategy.maxAttempts}`);
        console.log();
        
        const testCases = [1, 14, 27, 50, 84, 99, 100];
        const results = [];
        
        testCases.forEach(critical => {
            const result = this.simulate(critical);
            results.push({ critical, attempts: result.attempts });
            
            console.log(`临界层 ${critical}: ${result.attempts} 次尝试`);
        });
        
        const maxActual = Math.max(...results.map(r => r.attempts));
        console.log(`\n实际最大尝试次数: ${maxActual}`);
        console.log(`理论最大尝试次数: ${this.strategy.maxAttempts}`);
        
        return results;
    }
}

// 使用示例
const solver = new GlassBallSolver(100);
solver.testAllScenarios();
```

##### 3. Java 实现
```java
import java.util.*;

public class GlassBallProblem {
    private int totalFloors;
    private List<Integer> strategy;
    private int maxAttempts;
    
    public GlassBallProblem(int totalFloors) {
        this.totalFloors = totalFloors;
        calculateOptimalStrategy();
    }
    
    private void calculateOptimalStrategy() {
        // 计算最少尝试次数
        maxAttempts = (int) Math.ceil((-1 + Math.sqrt(1 + 8 * totalFloors)) / 2);
        
        // 生成测试序列
        strategy = new ArrayList<>();
        int currentFloor = maxAttempts;
        int step = maxAttempts - 1;
        
        while (currentFloor <= totalFloors) {
            strategy.add(currentFloor);
            currentFloor += step;
            step--;
            if (step <= 0) break;
        }
    }
    
    public SimulationResult simulate(int criticalFloor) {
        int attempts = 0;
        boolean ball1Broken = false;
        int lastSafeFloor = 0;
        List<String> log = new ArrayList<>();
        
        log.add(String.format("临界层: %d", criticalFloor));
        log.add(String.format("策略: %s", strategy.toString()));
        
        // 第一个球测试
        for (int floor : strategy) {
            attempts++;
            String result = floor >= criticalFloor ? "摔碎" : "安全";
            log.add(String.format("尝试 %d: 球1 从 %d 层 -> %s", attempts, floor, result));
            
            if (floor >= criticalFloor) {
                ball1Broken = true;
                break;
            }
            lastSafeFloor = floor;
        }
        
        // 第二个球测试
        if (ball1Broken) {
            int startFloor = lastSafeFloor + 1;
            int endFloor = strategy.stream()
                .filter(f -> f >= criticalFloor)
                .findFirst()
                .orElse(totalFloors);
            
            for (int floor = startFloor; floor <= endFloor; floor++) {
                attempts++;
                String result = floor >= criticalFloor ? "摔碎" : "安全";
                log.add(String.format("尝试 %d: 球2 从 %d 层 -> %s", attempts, floor, result));
                
                if (floor >= criticalFloor) {
                    log.add(String.format("找到临界层: %d", floor));
                    break;
                }
            }
        }
        
        log.add(String.format("总尝试次数: %d", attempts));
        return new SimulationResult(attempts, log);
    }
    
    public void testAllScenarios() {
        System.out.println("=== 玻璃球问题最优解 ===");
        System.out.println("策略: " + strategy);
        System.out.println("最大尝试次数: " + maxAttempts);
        System.out.println();
        
        int[] testCases = {1, 14, 27, 50, 84, 99, 100};
        int maxActual = 0;
        
        for (int critical : testCases) {
            SimulationResult result = simulate(critical);
            maxActual = Math.max(maxActual, result.attempts);
            System.out.printf("临界层 %d: %d 次尝试%n", critical, result.attempts);
        }
        
        System.out.printf("%n实际最大尝试次数: %d%n", maxActual);
        System.out.printf("理论最大尝试次数: %d%n", maxAttempts);
    }
    
    public static class SimulationResult {
        public final int attempts;
        public final List<String> log;
        
        public SimulationResult(int attempts, List<String> log) {
            this.attempts = attempts;
            this.log = log;
        }
    }
    
    public static void main(String[] args) {
        GlassBallProblem solver = new GlassBallProblem(100);
        solver.testAllScenarios();
    }
}
```

#### 五、复杂度分析
##### 1. 时间复杂度
- **最优策略**: O(√n)，其中 n 为楼层数
- **空间复杂度**: O(√n)，存储策略序列

##### 2. 与其他策略对比
| 策略 | 最坏情况尝试次数 | 平均尝试次数 | 适用场景 |
|------|------------------|--------------|----------|
| **逐层测试** | 100 | 50 | 只有1个球 |
| **固定间隔(10层)** | 19 | ~10 | 简单策略 |
| **二分法变种** | 50 | ~25 | 球数充足 |
| **最优策略** | **14** | **~7** | **2个球最优** |

#### 六、扩展问题
##### 1. N 个球，M 层楼
```python
def solve_n_balls_m_floors(n_balls, m_floors):
    """
    动态规划解决 N 个球 M 层楼问题
    dp[i][j] = 用 i 个球测试 j 层楼的最少尝试次数
    """
    # dp[i][j] 表示 i 个球 j 层楼的最少尝试次数
    dp = [[0] * (m_floors + 1) for _ in range(n_balls + 1)]
    
    # 初始化
    for j in range(1, m_floors + 1):
        dp[1][j] = j  # 1个球只能逐层测试
    
    for i in range(1, n_balls + 1):
        dp[i][1] = 1  # 1层楼只需1次尝试
    
    # 填充DP表
    for i in range(2, n_balls + 1):
        for j in range(2, m_floors + 1):
            dp[i][j] = float('inf')
            
            # 尝试在第k层扔球
            for k in range(1, j + 1):
                # 摔碎：用 i-1 个球测试 k-1 层
                # 没摔碎：用 i 个球测试 j-k 层
                worst_case = 1 + max(dp[i-1][k-1], dp[i][j-k])
                dp[i][j] = min(dp[i][j], worst_case)
    
    return dp[n_balls][m_floors]

# 测试不同配置
configs = [
    (2, 100),  # 经典问题
    (3, 100),  # 3个球100层
    (2, 200),  # 2个球200层
    (4, 100),  # 4个球100层
]

for balls, floors in configs:
    result = solve_n_balls_m_floors(balls, floors)
    print(f"{balls} 个球，{floors} 层楼：最少 {result} 次尝试")
```

##### 2. 实际应用场景
```python
class SoftwareTestingOptimizer:
    """
    软件测试中的版本回归问题
    类似玻璃球问题：找到引入bug的版本
    """
    
    def __init__(self, total_versions, test_cost):
        self.total_versions = total_versions
        self.test_cost = test_cost  # 每次测试的成本
    
    def optimize_testing_strategy(self):
        # 使用玻璃球算法优化测试策略
        n = math.ceil((-1 + math.sqrt(1 + 8 * self.total_versions)) / 2)
        
        strategy = []
        current = n
        step = n - 1
        
        while current <= self.total_versions:
            strategy.append(current)
            current += step
            step -= 1
            if step <= 0:
                break
        
        return {
            'strategy': strategy,
            'max_tests': n,
            'max_cost': n * self.test_cost,
            'savings': (self.total_versions - n) * self.test_cost
        }

# 示例：1000个版本，每次测试成本100元
optimizer = SoftwareTestingOptimizer(1000, 100)
result = optimizer.optimize_testing_strategy()

print(f"测试策略: {result['strategy'][:10]}...")  # 显示前10个
print(f"最大测试次数: {result['max_tests']}")
print(f"最大成本: {result['max_cost']}元")
print(f"相比逐个测试节省: {result['savings']}元")
```

**总结**：两个玻璃球100层楼问题的最优解是**14次尝试**，策略是第一个球按递减间隔测试（14, 27, 39, 50, 60, 69, 77, 84, 90, 95, 99层），第二个球在确定区间内逐层测试。这个问题体现了动态规划和贪心算法的思想，在软件测试、版本控制等领域有实际应用价值。
</details>

### 7. 一致性哈希原理，节点少导致数据倾斜如何解决（滴滴/陌陌）

<details>
<summary>答案与解析</summary> 

#### 一、一致性哈希原理
##### 1. 什么是一致性哈希
**一致性哈希（Consistent Hashing）**是一种特殊的哈希算法，主要用于解决分布式系统中节点动态增减时的数据重新分布问题。

##### 2. 传统哈希的问题
```python
# 传统取模哈希
class TraditionalHash:
    def __init__(self, node_count):
        self.node_count = node_count
        self.nodes = [f"node_{i}" for i in range(node_count)]
    
    def get_node(self, key):
        hash_value = hash(key)
        return self.nodes[hash_value % self.node_count]

# 问题演示：节点变化导致大量数据迁移
def demonstrate_traditional_hash_problem():
    # 初始3个节点
    th3 = TraditionalHash(3)
    
    # 数据分布
    keys = [f"key_{i}" for i in range(12)]
    distribution_3_nodes = {}
    
    for key in keys:
        node = th3.get_node(key)
        if node not in distribution_3_nodes:
            distribution_3_nodes[node] = []
        distribution_3_nodes[node].append(key)
    
    print("3个节点时的数据分布:")
    for node, data in distribution_3_nodes.items():
        print(f"{node}: {data}")
    
    # 增加到4个节点
    th4 = TraditionalHash(4)
    distribution_4_nodes = {}
    
    for key in keys:
        node = th4.get_node(key)
        if node not in distribution_4_nodes:
            distribution_4_nodes[node] = []
        distribution_4_nodes[node].append(key)
    
    print("\n4个节点时的数据分布:")
    for node, data in distribution_4_nodes.items():
        print(f"{node}: {data}")
    
    # 计算需要迁移的数据
    migration_count = 0
    for key in keys:
        old_node = th3.get_node(key)
        new_node = th4.get_node(key)
        if old_node != new_node:
            migration_count += 1
    
    print(f"\n需要迁移的数据: {migration_count}/{len(keys)} = {migration_count/len(keys)*100:.1f}%")

demonstrate_traditional_hash_problem()
```

##### 3. 一致性哈希原理
```python
import hashlib
import bisect
from typing import List, Dict, Optional

class ConsistentHash:
    def __init__(self, nodes: List[str] = None, replicas: int = 3):
        """
        初始化一致性哈希环
        :param nodes: 初始节点列表
        :param replicas: 虚拟节点数量（解决数据倾斜）
        """
        self.replicas = replicas
        self.ring: Dict[int, str] = {}  # 哈希环：{hash_value: node_name}
        self.sorted_keys: List[int] = []  # 排序的哈希值列表
        
        if nodes:
            for node in nodes:
                self.add_node(node)
    
    def _hash(self, key: str) -> int:
        """计算哈希值"""
        return int(hashlib.md5(key.encode('utf-8')).hexdigest(), 16)
    
    def add_node(self, node: str) -> None:
        """添加节点到哈希环"""
        for i in range(self.replicas):
            # 创建虚拟节点
            virtual_key = f"{node}:{i}"
            key = self._hash(virtual_key)
            
            self.ring[key] = node
            bisect.insort(self.sorted_keys, key)
        
        print(f"节点 {node} 已添加到哈希环（{self.replicas} 个虚拟节点）")
    
    def remove_node(self, node: str) -> None:
        """从哈希环移除节点"""
        for i in range(self.replicas):
            virtual_key = f"{node}:{i}"
            key = self._hash(virtual_key)
            
            if key in self.ring:
                del self.ring[key]
                self.sorted_keys.remove(key)
        
        print(f"节点 {node} 已从哈希环移除")
    
    def get_node(self, key: str) -> Optional[str]:
        """获取数据应该存储的节点"""
        if not self.ring:
            return None
        
        hash_key = self._hash(key)
        
        # 在哈希环上顺时针查找第一个节点
        idx = bisect.bisect_right(self.sorted_keys, hash_key)
        
        # 如果超出范围，回到环的开始
        if idx == len(self.sorted_keys):
            idx = 0
        
        return self.ring[self.sorted_keys[idx]]
    
    def get_nodes_for_key(self, key: str, count: int = 1) -> List[str]:
        """获取数据的多个副本节点（用于数据冗余）"""
        if not self.ring or count <= 0:
            return []
        
        hash_key = self._hash(key)
        idx = bisect.bisect_right(self.sorted_keys, hash_key)
        
        nodes = []
        seen_nodes = set()
        
        for _ in range(len(self.sorted_keys)):
            if idx >= len(self.sorted_keys):
                idx = 0
            
            node = self.ring[self.sorted_keys[idx]]
            if node not in seen_nodes:
                nodes.append(node)
                seen_nodes.add(node)
                
                if len(nodes) >= count:
                    break
            
            idx += 1
        
        return nodes
    
    def get_distribution(self, keys: List[str]) -> Dict[str, List[str]]:
        """获取数据分布情况"""
        distribution = {}
        
        for key in keys:
            node = self.get_node(key)
            if node not in distribution:
                distribution[node] = []
            distribution[node].append(key)
        
        return distribution
    
    def print_ring_info(self):
        """打印哈希环信息"""
        print(f"\n哈希环信息:")
        print(f"总节点数: {len(set(self.ring.values()))}")
        print(f"虚拟节点数: {len(self.ring)}")
        print(f"副本因子: {self.replicas}")
        
        # 统计每个物理节点的虚拟节点分布
        node_positions = {}
        for hash_val, node in self.ring.items():
            if node not in node_positions:
                node_positions[node] = []
            node_positions[node].append(hash_val)
        
        for node, positions in node_positions.items():
            print(f"{node}: {len(positions)} 个虚拟节点")
```

#### 二、数据倾斜问题及解决方案
##### 1. 数据倾斜的原因
```python
# 演示节点少时的数据倾斜问题
def demonstrate_data_skew():
    print("=== 数据倾斜问题演示 ===")
    
    # 只有2个节点，无虚拟节点
    ch_no_virtual = ConsistentHash(['node_A', 'node_B'], replicas=1)
    
    # 生成测试数据
    keys = [f"key_{i}" for i in range(100)]
    distribution = ch_no_virtual.get_distribution(keys)
    
    print("\n无虚拟节点时的分布:")
    for node, data in distribution.items():
        print(f"{node}: {len(data)} 个key ({len(data)/len(keys)*100:.1f}%)")
    
    # 计算负载均衡度
    counts = [len(data) for data in distribution.values()]
    max_count = max(counts)
    min_count = min(counts)
    balance_ratio = min_count / max_count if max_count > 0 else 0
    
    print(f"负载均衡度: {balance_ratio:.3f} (1.0为完全均衡)")
    print(f"最大偏差: {(max_count - min_count) / len(keys) * 100:.1f}%")

demonstrate_data_skew()
```

##### 2. 虚拟节点解决方案
```python
def solve_data_skew_with_virtual_nodes():
    print("\n=== 虚拟节点解决数据倾斜 ===")
    
    keys = [f"key_{i}" for i in range(1000)]
    
    # 测试不同虚拟节点数量的效果
    replica_counts = [1, 5, 10, 50, 100, 200]
    
    for replicas in replica_counts:
        ch = ConsistentHash(['node_A', 'node_B'], replicas=replicas)
        distribution = ch.get_distribution(keys)
        
        counts = [len(data) for data in distribution.values()]
        max_count = max(counts)
        min_count = min(counts)
        balance_ratio = min_count / max_count
        
        print(f"虚拟节点数 {replicas:3d}: 负载均衡度 {balance_ratio:.3f}, "
              f"偏差 {(max_count - min_count)/len(keys)*100:5.1f}%")

solve_data_skew_with_virtual_nodes()
```

##### 3. 其他解决数据倾斜的方案
```python
class AdvancedConsistentHash(ConsistentHash):
    """高级一致性哈希，包含多种数据倾斜解决方案"""
    
    def __init__(self, nodes=None, replicas=3, weight_enabled=False):
        super().__init__(nodes, replicas)
        self.weight_enabled = weight_enabled
        self.node_weights = {}  # 节点权重
    
    def set_node_weight(self, node: str, weight: float):
        """设置节点权重"""
        self.node_weights[node] = weight
    
    def add_weighted_node(self, node: str, weight: float = 1.0):
        """添加带权重的节点"""
        self.set_node_weight(node, weight)
        
        if self.weight_enabled:
            # 根据权重调整虚拟节点数量
            virtual_replicas = int(self.replicas * weight)
        else:
            virtual_replicas = self.replicas
        
        for i in range(virtual_replicas):
            virtual_key = f"{node}:{i}"
            key = self._hash(virtual_key)
            self.ring[key] = node
            bisect.insort(self.sorted_keys, key)
        
        print(f"节点 {node} (权重: {weight}) 已添加 {virtual_replicas} 个虚拟节点")
    
    def adaptive_rebalance(self):
        """自适应重平衡"""
        # 统计当前负载
        test_keys = [f"test_key_{i}" for i in range(10000)]
        distribution = self.get_distribution(test_keys)
        
        # 计算每个节点的负载率
        total_keys = len(test_keys)
        expected_load = total_keys / len(set(self.ring.values()))
        
        print("\n当前负载分布:")
        for node, keys in distribution.items():
            load_ratio = len(keys) / expected_load
            print(f"{node}: {len(keys)} keys (负载率: {load_ratio:.2f})")
            
            # 如果负载过高，增加虚拟节点
            if load_ratio > 1.2:
                self._add_virtual_nodes(node, 10)
            # 如果负载过低，减少虚拟节点
            elif load_ratio < 0.8:
                self._remove_virtual_nodes(node, 5)
    
    def _add_virtual_nodes(self, node: str, count: int):
        """为节点添加虚拟节点"""
        current_virtual_count = sum(1 for n in self.ring.values() if n == node)
        
        for i in range(count):
            virtual_key = f"{node}:extra_{current_virtual_count + i}"
            key = self._hash(virtual_key)
            self.ring[key] = node
            bisect.insort(self.sorted_keys, key)
        
        print(f"为 {node} 添加了 {count} 个虚拟节点")
    
    def _remove_virtual_nodes(self, node: str, count: int):
        """移除节点的部分虚拟节点"""
        node_keys = [k for k, v in self.ring.items() if v == node]
        
        # 移除最后添加的虚拟节点
        for i in range(min(count, len(node_keys) - self.replicas)):
            key_to_remove = node_keys[-(i+1)]
            del self.ring[key_to_remove]
            self.sorted_keys.remove(key_to_remove)
        
        print(f"为 {node} 移除了 {min(count, len(node_keys) - self.replicas)} 个虚拟节点")

# 使用示例
def advanced_consistent_hash_demo():
    print("\n=== 高级一致性哈希演示 ===")
    
    # 创建带权重的一致性哈希
    ach = AdvancedConsistentHash(replicas=50, weight_enabled=True)
    
    # 添加不同性能的节点
    ach.add_weighted_node('high_perf_node', weight=2.0)  # 高性能节点
    ach.add_weighted_node('medium_perf_node', weight=1.0)  # 中等性能节点
    ach.add_weighted_node('low_perf_node', weight=0.5)  # 低性能节点
    
    # 测试负载分布
    test_keys = [f"key_{i}" for i in range(1000)]
    distribution = ach.get_distribution(test_keys)
    
    print("\n权重分配后的负载分布:")
    for node, keys in distribution.items():
        weight = ach.node_weights.get(node, 1.0)
        print(f"{node}: {len(keys)} keys (权重: {weight})")
    
    # 自适应重平衡
    ach.adaptive_rebalance()

advanced_consistent_hash_demo()
```

#### 三、一致性哈希的优化策略
##### 1. 跳跃一致性哈希
```python
import random

class JumpConsistentHash:
    """跳跃一致性哈希 - Google提出的算法"""
    
    @staticmethod
    def jump_consistent_hash(key: int, num_buckets: int) -> int:
        """
        跳跃一致性哈希算法
        时间复杂度: O(ln(num_buckets))
        空间复杂度: O(1)
        """
        random.seed(key)
        b, j = -1, 0
        
        while j < num_buckets:
            b = j
            r = random.random()
            j = int((b + 1) / r)
        
        return b
    
    @staticmethod
    def distribute_keys(keys: list, num_buckets: int) -> dict:
        """分布数据到桶中"""
        distribution = {i: [] for i in range(num_buckets)}
        
        for key in keys:
            bucket = JumpConsistentHash.jump_consistent_hash(hash(key), num_buckets)
            distribution[bucket].append(key)
        
        return distribution

# 跳跃一致性哈希演示
def jump_consistent_hash_demo():
    print("\n=== 跳跃一致性哈希演示 ===")
    
    keys = [f"key_{i}" for i in range(1000)]
    
    # 3个桶的分布
    dist_3 = JumpConsistentHash.distribute_keys(keys, 3)
    print("3个桶的分布:")
    for bucket, bucket_keys in dist_3.items():
        print(f"桶 {bucket}: {len(bucket_keys)} keys")
    
    # 增加到4个桶
    dist_4 = JumpConsistentHash.distribute_keys(keys, 4)
    print("\n4个桶的分布:")
    for bucket, bucket_keys in dist_4.items():
        print(f"桶 {bucket}: {len(bucket_keys)} keys")
    
    # 计算迁移量
    migration_count = 0
    for key in keys:
        old_bucket = JumpConsistentHash.jump_consistent_hash(hash(key), 3)
        new_bucket = JumpConsistentHash.jump_consistent_hash(hash(key), 4)
        if old_bucket != new_bucket:
            migration_count += 1
    
    print(f"\n需要迁移: {migration_count}/{len(keys)} = {migration_count/len(keys)*100:.1f}%")
    print(f"理论最优迁移率: {1/4*100:.1f}%")

jump_consistent_hash_demo()
```

##### 2. 有界负载一致性哈希
```python
class BoundedLoadConsistentHash(ConsistentHash):
    """有界负载一致性哈希 - 限制单个节点的最大负载"""
    
    def __init__(self, nodes=None, replicas=3, load_factor=1.25):
        super().__init__(nodes, replicas)
        self.load_factor = load_factor  # 负载因子
        self.node_loads = {}  # 节点当前负载
        self.total_keys = 0
    
    def add_key(self, key: str) -> str:
        """添加key，考虑负载均衡"""
        self.total_keys += 1
        
        # 计算平均负载和最大允许负载
        num_nodes = len(set(self.ring.values()))
        avg_load = self.total_keys / num_nodes
        max_load = int(avg_load * self.load_factor)
        
        # 获取候选节点（按哈希环顺序）
        candidates = self.get_nodes_for_key(key, count=num_nodes)
        
        # 选择负载未超限的第一个节点
        for node in candidates:
            current_load = self.node_loads.get(node, 0)
            if current_load < max_load:
                self.node_loads[node] = current_load + 1
                return node
        
        # 如果所有节点都超载，选择负载最小的
        min_load_node = min(candidates, 
                           key=lambda n: self.node_loads.get(n, 0))
        self.node_loads[min_load_node] = self.node_loads.get(min_load_node, 0) + 1
        
        return min_load_node
    
    def remove_key(self, key: str):
        """移除key"""
        node = self.get_node(key)
        if node and self.node_loads.get(node, 0) > 0:
            self.node_loads[node] -= 1
            self.total_keys -= 1
    
    def get_load_stats(self):
        """获取负载统计"""
        if not self.node_loads:
            return {}
        
        num_nodes = len(set(self.ring.values()))
        avg_load = self.total_keys / num_nodes if num_nodes > 0 else 0
        
        stats = {
            'total_keys': self.total_keys,
            'avg_load': avg_load,
            'max_allowed_load': int(avg_load * self.load_factor),
            'node_loads': dict(self.node_loads),
            'load_balance_ratio': min(self.node_loads.values()) / max(self.node_loads.values()) if self.node_loads else 0
        }
        
        return stats

# 有界负载演示
def bounded_load_demo():
    print("\n=== 有界负载一致性哈希演示 ===")
    
    # 创建有界负载哈希
    blch = BoundedLoadConsistentHash(['node_A', 'node_B', 'node_C'], 
                                    replicas=10, load_factor=1.25)
    
    # 添加大量key
    keys = [f"key_{i}" for i in range(300)]
    
    print("添加300个key...")
    for key in keys:
        blch.add_key(key)
    
    # 显示负载统计
    stats = blch.get_load_stats()
    print(f"\n负载统计:")
    print(f"总key数: {stats['total_keys']}")
    print(f"平均负载: {stats['avg_load']:.1f}")
    print(f"最大允许负载: {stats['max_allowed_load']}")
    print(f"负载均衡度: {stats['load_balance_ratio']:.3f}")
    
    print("\n各节点负载:")
    for node, load in stats['node_loads'].items():
        print(f"{node}: {load} keys")

bounded_load_demo()
```

#### 四、实际应用场景
##### 1. 分布式缓存
```python
class DistributedCache:
    """基于一致性哈希的分布式缓存"""
    
    def __init__(self, cache_nodes):
        self.ch = ConsistentHash(cache_nodes, replicas=100)
        self.caches = {node: {} for node in cache_nodes}  # 模拟缓存节点
    
    def put(self, key: str, value: any, replicas: int = 2):
        """存储数据到缓存"""
        nodes = self.ch.get_nodes_for_key(key, replicas)
        
        for node in nodes:
            self.caches[node][key] = value
        
        return nodes
    
    def get(self, key: str):
        """从缓存获取数据"""
        node = self.ch.get_node(key)
        return self.caches[node].get(key)
    
    def delete(self, key: str):
        """删除缓存数据"""
        # 从所有可能的副本节点删除
        nodes = self.ch.get_nodes_for_key(key, len(self.caches))
        
        for node in nodes:
            if key in self.caches[node]:
                del self.caches[node][key]
    
    def add_cache_node(self, node: str):
        """添加缓存节点"""
        self.ch.add_node(node)
        self.caches[node] = {}
        
        # 数据迁移（简化版）
        self._migrate_data_for_new_node(node)
    
    def _migrate_data_for_new_node(self, new_node: str):
        """为新节点迁移数据"""
        migration_count = 0
        
        # 检查所有现有数据，看是否需要迁移到新节点
        for node, cache in self.caches.items():
            if node == new_node:
                continue
            
            keys_to_migrate = []
            for key in list(cache.keys()):
                # 重新计算key应该在哪个节点
                target_node = self.ch.get_node(key)
                if target_node == new_node:
                    keys_to_migrate.append(key)
            
            # 执行迁移
            for key in keys_to_migrate:
                value = cache.pop(key)
                self.caches[new_node][key] = value
                migration_count += 1
        
        print(f"为新节点 {new_node} 迁移了 {migration_count} 个key")
    
    def get_cache_stats(self):
        """获取缓存统计信息"""
        stats = {}
        total_keys = 0
        
        for node, cache in self.caches.items():
            key_count = len(cache)
            stats[node] = key_count
            total_keys += key_count
        
        return {
            'node_stats': stats,
            'total_keys': total_keys,
            'avg_keys_per_node': total_keys / len(self.caches) if self.caches else 0
        }

# 分布式缓存演示
def distributed_cache_demo():
    print("\n=== 分布式缓存演示 ===")
    
    # 创建分布式缓存
    cache = DistributedCache(['cache_1', 'cache_2', 'cache_3'])
    
    # 存储数据
    print("存储100个key-value对...")
    for i in range(100):
        cache.put(f"key_{i}", f"value_{i}")
    
    # 显示初始分布
    stats = cache.get_cache_stats()
    print("\n初始数据分布:")
    for node, count in stats['node_stats'].items():
        print(f"{node}: {count} keys")
    
    # 添加新的缓存节点
    print("\n添加新缓存节点 cache_4...")
    cache.add_cache_node('cache_4')
    
    # 显示迁移后的分布
    stats = cache.get_cache_stats()
    print("\n迁移后数据分布:")
    for node, count in stats['node_stats'].items():
        print(f"{node}: {count} keys")
    
    # 测试数据访问
    print("\n测试数据访问:")
    for i in range(0, 100, 20):
        key = f"key_{i}"
        value = cache.get(key)
        node = cache.ch.get_node(key)
        print(f"{key} -> {value} (存储在 {node})")

distributed_cache_demo()
```

##### 2. 负载均衡
```python
class ConsistentHashLoadBalancer:
    """基于一致性哈希的负载均衡器"""
    
    def __init__(self, servers, health_check_interval=30):
        self.ch = ConsistentHash(servers, replicas=50)
        self.servers = {server: {'healthy': True, 'connections': 0} 
                       for server in servers}
        self.health_check_interval = health_check_interval
    
    def route_request(self, session_id: str, backup_count: int = 2):
        """路由请求到服务器"""
        # 获取候选服务器（主服务器 + 备用服务器）
        candidates = self.ch.get_nodes_for_key(session_id, backup_count + 1)
        
        # 选择健康的服务器
        for server in candidates:
            if self.servers[server]['healthy']:
                self.servers[server]['connections'] += 1
                return server
        
        # 如果没有健康的服务器，返回None或抛出异常
        raise Exception("No healthy servers available")
    
    def release_connection(self, server: str):
        """释放连接"""
        if server in self.servers and self.servers[server]['connections'] > 0:
            self.servers[server]['connections'] -= 1
    
    def mark_server_unhealthy(self, server: str):
        """标记服务器为不健康"""
        if server in self.servers:
            self.servers[server]['healthy'] = False
            print(f"服务器 {server} 标记为不健康")
    
    def mark_server_healthy(self, server: str):
        """标记服务器为健康"""
        if server in self.servers:
            self.servers[server]['healthy'] = True
            print(f"服务器 {server} 恢复健康")
    
    def add_server(self, server: str):
        """添加新服务器"""
        self.ch.add_node(server)
        self.servers[server] = {'healthy': True, 'connections': 0}
        print(f"新服务器 {server} 已添加")
    
    def remove_server(self, server: str):
        """移除服务器"""
        self.ch.remove_node(server)
        if server in self.servers:
            del self.servers[server]
        print(f"服务器 {server} 已移除")
    
    def get_server_stats(self):
        """获取服务器统计信息"""
        return dict(self.servers)

# 负载均衡演示
def load_balancer_demo():
    print("\n=== 一致性哈希负载均衡演示 ===")
    
    # 创建负载均衡器
    lb = ConsistentHashLoadBalancer(['server_1', 'server_2', 'server_3'])
    
    # 模拟请求
    sessions = [f"session_{i}" for i in range(50)]
    
    print("处理50个会话请求...")
    session_server_map = {}
    
    for session in sessions:
        try:
            server = lb.route_request(session)
            session_server_map[session] = server
        except Exception as e:
            print(f"路由失败: {e}")
    
    # 显示初始分布
    stats = lb.get_server_stats()
    print("\n初始请求分布:")
    for server, info in stats.items():
        print(f"{server}: {info['connections']} 连接 (健康: {info['healthy']})")
    
    # 模拟服务器故障
    print("\n模拟 server_2 故障...")
    lb.mark_server_unhealthy('server_2')
    
    # 重新路由受影响的会话
    affected_sessions = [s for s, srv in session_server_map.items() if srv == 'server_2']
    print(f"需要重新路由 {len(affected_sessions)} 个会话")
    
    for session in affected_sessions:
        lb.release_connection('server_2')
        try:
            new_server = lb.route_request(session)
            session_server_map[session] = new_server
            print(f"{session}: server_2 -> {new_server}")
        except Exception as e:
            print(f"重新路由失败: {e}")
    
    # 显示故障后的分布
    stats = lb.get_server_stats()
    print("\n故障后请求分布:")
    for server, info in stats.items():
        print(f"{server}: {info['connections']} 连接 (健康: {info['healthy']})")
    
    # 添加新服务器
    print("\n添加新服务器 server_4...")
    lb.add_server('server_4')
    
    # 模拟新请求
    new_sessions = [f"new_session_{i}" for i in range(20)]
    for session in new_sessions:
        server = lb.route_request(session)
        session_server_map[session] = server
    
    # 最终分布
    stats = lb.get_server_stats()
    print("\n添加服务器后的分布:")
    for server, info in stats.items():
        print(f"{server}: {info['connections']} 连接 (健康: {info['healthy']})")

load_balancer_demo()
```

**总结**：一致性哈希通过虚拟节点、权重分配、有界负载等技术有效解决了节点少导致的数据倾斜问题。在实际应用中，需要根据具体场景选择合适的策略：

1. **虚拟节点数量**：通常50-200个虚拟节点能很好平衡性能和均匀性
2. **权重机制**：根据节点性能差异调整虚拟节点数量
3. **有界负载**：限制单节点最大负载，防止热点问题
4. **动态调整**：监控负载分布，动态增减虚拟节点
5. **故障处理**：快速检测和处理节点故障，保证服务可用性
</details>

---

## 系统设计

### 1. 设计秒杀系统，需要支持 100W+ QPS（滴滴）

<details>
<summary>答案与解析</summary> 

#### 一、秒杀系统核心挑战
##### 1. 高并发冲击
- **瞬时流量**：100W+ QPS 集中在秒杀开始的几秒内
- **热点数据**：所有用户抢购同一商品，数据库单点压力巨大
- **库存超卖**：并发扣减库存时的数据一致性问题
- **系统雪崩**：下游服务承受不住压力导致连锁故障

##### 2. 用户体验要求
- **响应速度**：页面加载和下单响应要快（<200ms）
- **公平性**：先到先得，不能出现插队
- **稳定性**：系统不能崩溃，失败要有友好提示

#### 二、整体架构设计
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   用户端        │    │   CDN/静态资源   │    │   负载均衡       │
│  (APP/Web)      │───▶│   (商品详情页)   │───▶│   (Nginx/LVS)   │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                                        │
                                                        ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   接入层        │    │   业务层        │    │   数据层        │
│  (API Gateway)  │───▶│  (秒杀服务)     │───▶│ (Redis/MySQL)   │
│  - 限流         │    │  - 库存扣减     │    │ - 库存缓存      │
│  - 鉴权         │    │  - 订单创建     │    │ - 订单数据      │
│  - 熔断         │    │  - 支付调用     │    │ - 用户数据      │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │   消息队列      │
                       │  (Kafka/RabbitMQ)│
                       │  - 异步处理     │
                       │  - 削峰填谷     │
                       └─────────────────┘
```

#### 三、核心技术方案
##### 1. 多级缓存架构
```php
<?php
// 秒杀商品缓存管理
class SeckillCache {
    private $redis;
    private $localCache;
    
    public function __construct() {
        $this->redis = new Redis();
        $this->redis->connect('127.0.0.1', 6379);
        $this->localCache = new ArrayCache(); // 本地缓存
    }
    
    /**
     * 多级缓存获取商品信息
     */
    public function getProduct($productId) {
        // L1: 本地缓存 (1秒过期)
        $cacheKey = "product:{$productId}";
        $product = $this->localCache->get($cacheKey);
        if ($product !== null) {
            return $product;
        }
        
        // L2: Redis缓存 (60秒过期)
        $product = $this->redis->get($cacheKey);
        if ($product !== false) {
            $product = json_decode($product, true);
            $this->localCache->set($cacheKey, $product, 1);
            return $product;
        }
        
        // L3: 数据库 (带分布式锁防击穿)
        $lockKey = "lock:product:{$productId}";
        if ($this->redis->set($lockKey, 1, ['nx', 'ex' => 10])) {
            try {
                $product = $this->loadFromDatabase($productId);
                if ($product) {
                    $this->redis->setex($cacheKey, 60, json_encode($product));
                    $this->localCache->set($cacheKey, $product, 1);
                }
                return $product;
            } finally {
                $this->redis->del($lockKey);
            }
        }
        
        // 获取锁失败，短暂等待后重试
        usleep(50000); // 50ms
        return $this->redis->get($cacheKey) ?: null;
    }
    
    /**
     * 预热缓存
     */
    public function warmupCache($productIds) {
        foreach ($productIds as $productId) {
            $product = $this->loadFromDatabase($productId);
            if ($product) {
                $this->redis->setex("product:{$productId}", 300, json_encode($product));
            }
        }
    }
}
```

##### 2. 库存扣减方案
```php
<?php
// 基于Redis的原子库存扣减
class StockManager {
    private $redis;
    
    public function __construct() {
        $this->redis = new Redis();
        $this->redis->connect('127.0.0.1', 6379);
    }
    
    /**
     * 初始化库存
     */
    public function initStock($productId, $stock) {
        $key = "stock:{$productId}";
        $this->redis->set($key, $stock);
        
        // 设置库存预警阈值
        $this->redis->set("stock:threshold:{$productId}", max(1, $stock * 0.1));
    }
    
    /**
     * 原子扣减库存 (Lua脚本保证原子性)
     */
    public function decrStock($productId, $quantity = 1) {
        $luaScript = '
            local key = KEYS[1]
            local quantity = tonumber(ARGV[1])
            local current = tonumber(redis.call("GET", key) or 0)
            
            if current >= quantity then
                redis.call("DECRBY", key, quantity)
                return {1, current - quantity} -- 成功，返回剩余库存
            else
                return {0, current} -- 失败，库存不足
            end
        ';
        
        $result = $this->redis->eval($luaScript, ["stock:{$productId}", $quantity], 1);
        
        return [
            'success' => $result[0] == 1,
            'remaining' => $result[1]
        ];
    }
    
    /**
     * 获取当前库存
     */
    public function getStock($productId) {
        return (int)$this->redis->get("stock:{$productId}") ?: 0;
    }
    
    /**
     * 库存回滚 (订单取消时)
     */
    public function rollbackStock($productId, $quantity = 1) {
        $this->redis->incrby("stock:{$productId}", $quantity);
    }
}
```

##### 3. 限流与熔断
```php
<?php
// 令牌桶限流器
class TokenBucketLimiter {
    private $redis;
    private $capacity;    // 桶容量
    private $refillRate;  // 令牌补充速率(个/秒)
    
    public function __construct($capacity = 1000, $refillRate = 100) {
        $this->redis = new Redis();
        $this->redis->connect('127.0.0.1', 6379);
        $this->capacity = $capacity;
        $this->refillRate = $refillRate;
    }
    
    /**
     * 尝试获取令牌
     */
    public function tryAcquire($key, $tokens = 1) {
        $luaScript = '
            local key = KEYS[1]
            local capacity = tonumber(ARGV[1])
            local refill_rate = tonumber(ARGV[2])
            local tokens_requested = tonumber(ARGV[3])
            local now = tonumber(ARGV[4])
            
            local bucket = redis.call("HMGET", key, "tokens", "last_refill")
            local tokens = tonumber(bucket[1]) or capacity
            local last_refill = tonumber(bucket[2]) or now
            
            -- 计算需要补充的令牌数
            local elapsed = math.max(0, now - last_refill)
            local tokens_to_add = math.floor(elapsed * refill_rate / 1000)
            tokens = math.min(capacity, tokens + tokens_to_add)
            
            if tokens >= tokens_requested then
                tokens = tokens - tokens_requested
                redis.call("HMSET", key, "tokens", tokens, "last_refill", now)
                redis.call("EXPIRE", key, 3600)
                return 1
            else
                redis.call("HMSET", key, "tokens", tokens, "last_refill", now)
                redis.call("EXPIRE", key, 3600)
                return 0
            end
        ';
        
        $result = $this->redis->eval($luaScript, 
            [$key, $this->capacity, $this->refillRate, $tokens, time() * 1000], 1);
        
        return $result == 1;
    }
}

// 熔断器
class CircuitBreaker {
    private $redis;
    private $failureThreshold;
    private $recoveryTimeout;
    private $requestVolumeThreshold;
    
    public function __construct($failureThreshold = 50, $recoveryTimeout = 60, $requestVolumeThreshold = 20) {
        $this->redis = new Redis();
        $this->redis->connect('127.0.0.1', 6379);
        $this->failureThreshold = $failureThreshold;
        $this->recoveryTimeout = $recoveryTimeout;
        $this->requestVolumeThreshold = $requestVolumeThreshold;
    }
    
    /**
     * 检查是否允许请求通过
     */
    public function allowRequest($serviceName) {
        $key = "circuit_breaker:{$serviceName}";
        $state = $this->redis->hget($key, 'state') ?: 'CLOSED';
        
        switch ($state) {
            case 'OPEN':
                $lastFailTime = $this->redis->hget($key, 'last_fail_time');
                if (time() - $lastFailTime > $this->recoveryTimeout) {
                    $this->redis->hset($key, 'state', 'HALF_OPEN');
                    return true;
                }
                return false;
                
            case 'HALF_OPEN':
                return true;
                
            case 'CLOSED':
            default:
                return true;
        }
    }
    
    /**
     * 记录请求结果
     */
    public function recordResult($serviceName, $success) {
        $key = "circuit_breaker:{$serviceName}";
        $now = time();
        
        if ($success) {
            $this->redis->hset($key, 'state', 'CLOSED');
            $this->redis->hdel($key, 'failure_count', 'last_fail_time');
        } else {
            $failureCount = $this->redis->hincrby($key, 'failure_count', 1);
            $this->redis->hset($key, 'last_fail_time', $now);
            
            $requestCount = $this->redis->hincrby($key, 'request_count', 1);
            
            if ($requestCount >= $this->requestVolumeThreshold) {
                $failureRate = ($failureCount / $requestCount) * 100;
                if ($failureRate >= $this->failureThreshold) {
                    $this->redis->hset($key, 'state', 'OPEN');
                }
            }
        }
        
        $this->redis->expire($key, 300);
    }
}
```

##### 4. 秒杀核心业务逻辑
```php
<?php
class SeckillService {
    private $stockManager;
    private $limiter;
    private $circuitBreaker;
    private $cache;
    private $queue;
    
    public function __construct() {
        $this->stockManager = new StockManager();
        $this->limiter = new TokenBucketLimiter(10000, 1000); // 10k容量，1k/s补充
        $this->circuitBreaker = new CircuitBreaker();
        $this->cache = new SeckillCache();
        $this->queue = new RabbitMQProducer();
    }
    
    /**
     * 秒杀下单
     */
    public function seckill($userId, $productId, $quantity = 1) {
        try {
            // 1. 限流检查
            if (!$this->limiter->tryAcquire("seckill:{$productId}", 1)) {
                return $this->error('系统繁忙，请稍后重试', 'RATE_LIMITED');
            }
            
            // 2. 熔断检查
            if (!$this->circuitBreaker->allowRequest('seckill_service')) {
                return $this->error('服务暂时不可用', 'SERVICE_UNAVAILABLE');
            }
            
            // 3. 用户资格检查 (防刷)
            if (!$this->checkUserEligibility($userId, $productId)) {
                return $this->error('您已参与过该商品秒杀', 'ALREADY_PARTICIPATED');
            }
            
            // 4. 商品信息检查
            $product = $this->cache->getProduct($productId);
            if (!$product || $product['status'] !== 'active') {
                return $this->error('商品不存在或已下架', 'PRODUCT_NOT_FOUND');
            }
            
            // 5. 时间窗口检查
            if (!$this->isInSeckillTime($product)) {
                return $this->error('不在秒杀时间内', 'NOT_IN_SECKILL_TIME');
            }
            
            // 6. 库存扣减 (原子操作)
            $stockResult = $this->stockManager->decrStock($productId, $quantity);
            if (!$stockResult['success']) {
                $this->circuitBreaker->recordResult('seckill_service', false);
                return $this->error('商品已售罄', 'OUT_OF_STOCK');
            }
            
            // 7. 创建预订单 (异步处理)
            $orderId = $this->generateOrderId();
            $orderData = [
                'order_id' => $orderId,
                'user_id' => $userId,
                'product_id' => $productId,
                'quantity' => $quantity,
                'price' => $product['seckill_price'],
                'created_at' => time()
            ];
            
            // 8. 发送到消息队列异步处理
            $this->queue->publish('seckill.order.create', $orderData);
            
            // 9. 设置用户参与标记 (防重复)
            $this->markUserParticipated($userId, $productId);
            
            $this->circuitBreaker->recordResult('seckill_service', true);
            
            return $this->success([
                'order_id' => $orderId,
                'message' => '抢购成功，请在15分钟内完成支付',
                'remaining_stock' => $stockResult['remaining']
            ]);
            
        } catch (Exception $e) {
            // 库存回滚
            if (isset($stockResult) && $stockResult['success']) {
                $this->stockManager->rollbackStock($productId, $quantity);
            }
            
            $this->circuitBreaker->recordResult('seckill_service', false);
            
            return $this->error('系统异常，请重试', 'SYSTEM_ERROR');
        }
    }
    
    /**
     * 检查用户资格
     */
    private function checkUserEligibility($userId, $productId) {
        $key = "seckill:user:{$userId}:product:{$productId}";
        return !$this->cache->redis->exists($key);
    }
    
    /**
     * 标记用户已参与
     */
    private function markUserParticipated($userId, $productId) {
        $key = "seckill:user:{$userId}:product:{$productId}";
        $this->cache->redis->setex($key, 86400, 1); // 24小时过期
    }
    
    /**
     * 检查是否在秒杀时间内
     */
    private function isInSeckillTime($product) {
        $now = time();
        return $now >= $product['start_time'] && $now <= $product['end_time'];
    }
    
    private function generateOrderId() {
        return 'SK' . date('YmdHis') . mt_rand(1000, 9999);
    }
    
    private function success($data) {
        return ['code' => 0, 'message' => 'success', 'data' => $data];
    }
    
    private function error($message, $errorCode) {
        return ['code' => -1, 'message' => $message, 'error_code' => $errorCode];
    }
}
```

#### 四、数据库设计
```sql
-- 秒杀商品表
CREATE TABLE seckill_products (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    product_id BIGINT NOT NULL,
    title VARCHAR(255) NOT NULL,
    original_price DECIMAL(10,2) NOT NULL,
    seckill_price DECIMAL(10,2) NOT NULL,
    total_stock INT NOT NULL,
    remaining_stock INT NOT NULL,
    start_time TIMESTAMP NOT NULL,
    end_time TIMESTAMP NOT NULL,
    status ENUM('pending', 'active', 'ended') DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_product_id (product_id),
    INDEX idx_status_time (status, start_time, end_time)
);

-- 秒杀订单表
CREATE TABLE seckill_orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_id VARCHAR(32) NOT NULL UNIQUE,
    user_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INT NOT NULL DEFAULT 1,
    unit_price DECIMAL(10,2) NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL,
    status ENUM('pending', 'paid', 'cancelled', 'expired') DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    expired_at TIMESTAMP NOT NULL, -- 支付过期时间
    
    INDEX idx_order_id (order_id),
    INDEX idx_user_id (user_id),
    INDEX idx_product_id (product_id),
    INDEX idx_status_expired (status, expired_at)
);

-- 用户参与记录表 (防刷)
CREATE TABLE seckill_participations (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    participated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE KEY uk_user_product (user_id, product_id),
    INDEX idx_user_id (user_id),
    INDEX idx_product_id (product_id)
);
```

#### 五、性能优化策略
##### 1. 读写分离与分库分表
```php
<?php
// 订单分表策略
class OrderSharding {
    private $writeDb;
    private $readDbs;
    
    public function __construct() {
        $this->writeDb = new PDO('mysql:host=master;dbname=seckill', $user, $pass);
        $this->readDbs = [
            new PDO('mysql:host=slave1;dbname=seckill', $user, $pass),
            new PDO('mysql:host=slave2;dbname=seckill', $user, $pass),
        ];
    }
    
    /**
     * 根据用户ID分表
     */
    private function getTableSuffix($userId) {
        return $userId % 10; // 分10张表
    }
    
    /**
     * 插入订单 (写主库)
     */
    public function insertOrder($orderData) {
        $tableSuffix = $this->getTableSuffix($orderData['user_id']);
        $tableName = "seckill_orders_{$tableSuffix}";
        
        $sql = "INSERT INTO {$tableName} (order_id, user_id, product_id, quantity, unit_price, total_amount, expired_at) 
                VALUES (:order_id, :user_id, :product_id, :quantity, :unit_price, :total_amount, :expired_at)";
        
        $stmt = $this->writeDb->prepare($sql);
        return $stmt->execute($orderData);
    }
    
    /**
     * 查询订单 (读从库)
     */
    public function getOrder($userId, $orderId) {
        $tableSuffix = $this->getTableSuffix($userId);
        $tableName = "seckill_orders_{$tableSuffix}";
        
        $readDb = $this->readDbs[array_rand($this->readDbs)];
        $sql = "SELECT * FROM {$tableName} WHERE user_id = :user_id AND order_id = :order_id";
        
        $stmt = $readDb->prepare($sql);
        $stmt->execute(['user_id' => $userId, 'order_id' => $orderId]);
        return $stmt->fetch(PDO::FETCH_ASSOC);
    }
}
```

##### 2. 异步处理与消息队列
```php
<?php
// 订单处理消费者
class SeckillOrderConsumer {
    private $db;
    private $paymentService;
    private $stockManager;
    
    public function __construct() {
        $this->db = new PDO('mysql:host=master;dbname=seckill', $user, $pass);
        $this->paymentService = new PaymentService();
        $this->stockManager = new StockManager();
    }
    
    /**
     * 处理秒杀订单创建
     */
    public function handleOrderCreate($orderData) {
        try {
            $this->db->beginTransaction();
            
            // 1. 创建订单记录
            $orderData['expired_at'] = date('Y-m-d H:i:s', time() + 900); // 15分钟过期
            $this->insertOrder($orderData);
            
            // 2. 发送支付通知
            $this->sendPaymentNotification($orderData);
            
            // 3. 设置订单过期处理
            $this->scheduleOrderExpiration($orderData['order_id']);
            
            $this->db->commit();
            
        } catch (Exception $e) {
            $this->db->rollback();
            
            // 回滚库存
            $this->stockManager->rollbackStock($orderData['product_id'], $orderData['quantity']);
            
            // 记录错误日志
            error_log("Order creation failed: " . $e->getMessage());
        }
    }
    
    /**
     * 处理订单过期
     */
    public function handleOrderExpiration($orderId) {
        $order = $this->getOrderById($orderId);
        if ($order && $order['status'] === 'pending') {
            // 取消订单
            $this->cancelOrder($orderId);
            
            // 回滚库存
            $this->stockManager->rollbackStock($order['product_id'], $order['quantity']);
        }
    }
}
```

#### 六、监控与告警
```php
<?php
// 秒杀系统监控
class SeckillMonitor {
    private $redis;
    private $alertService;
    
    public function __construct() {
        $this->redis = new Redis();
        $this->redis->connect('127.0.0.1', 6379);
        $this->alertService = new AlertService();
    }
    
    /**
     * 监控关键指标
     */
    public function monitor() {
        // 1. QPS监控
        $qps = $this->getQPS();
        if ($qps > 100000) { // 超过10W QPS告警
            $this->alertService->send('HIGH_QPS', "当前QPS: {$qps}");
        }
        
        // 2. 库存监控
        $this->monitorStock();
        
        // 3. 错误率监控
        $errorRate = $this->getErrorRate();
        if ($errorRate > 5) { // 错误率超过5%告警
            $this->alertService->send('HIGH_ERROR_RATE', "错误率: {$errorRate}%");
        }
        
        // 4. 响应时间监控
        $avgResponseTime = $this->getAvgResponseTime();
        if ($avgResponseTime > 200) { // 响应时间超过200ms告警
            $this->alertService->send('SLOW_RESPONSE', "平均响应时间: {$avgResponseTime}ms");
        }
    }
    
    private function monitorStock() {
        $products = $this->getActiveProducts();
        foreach ($products as $product) {
            $remaining = $this->stockManager->getStock($product['id']);
            $threshold = $this->redis->get("stock:threshold:{$product['id']}") ?: 0;
            
            if ($remaining <= $threshold) {
                $this->alertService->send('LOW_STOCK', 
                    "商品 {$product['title']} 库存不足，剩余: {$remaining}");
            }
        }
    }
}
```

#### 七、压测与容量规划
##### 1. 压测脚本
```bash
#!/bin/bash
# 使用wrk进行压力测试

# 测试秒杀接口
wrk -t20 -c1000 -d60s --timeout 10s \
    -s seckill_test.lua \
    http://api.example.com/seckill

# Lua脚本 (seckill_test.lua)
echo '
request = function()
    local user_id = math.random(1, 100000)
    local product_id = 12345
    local body = string.format(
        "{\"user_id\":%d,\"product_id\":%d,\"quantity\":1}", 
        user_id, product_id
    )
    
    return wrk.format("POST", "/seckill", {
        ["Content-Type"] = "application/json",
        ["Authorization"] = "Bearer test_token"
    }, body)
end
' > seckill_test.lua
```

##### 2. 容量规划
```
目标: 100W QPS

计算:
- 单机QPS: 5000 (经验值)
- 需要机器数: 100W / 5000 = 200台
- 考虑冗余: 200 * 1.5 = 300台

数据库:
- 主库: 写QPS 1W (订单创建)
- 从库: 读QPS 10W (商品查询)
- 需要读库: 10W / 2W = 5台

Redis:
- 单实例QPS: 10W
- 需要实例: 100W / 10W = 10个
- 考虑主从: 10 * 2 = 20个实例

带宽:
- 单请求: 1KB
- 总带宽: 100W * 1KB = 1GB/s
- 需要带宽: 1GB/s * 8 = 8Gbps
```

**总结**：秒杀系统的核心是"削峰填谷"和"快速失败"。通过多级缓存、原子库存扣减、限流熔断、异步处理等技术手段，可以有效应对高并发冲击。关键是要做好容量规划、性能测试和实时监控，确保系统在极限压力下仍能稳定运行。
</details>

### 2. 设计微博首页，拉取所有关注用户的最近 20 条微博（百度）

<details>
<summary>答案与解析</summary> 

#### 一、需求分析
##### 1. 功能需求
- **Timeline展示**：显示关注用户的最新20条微博
- **实时更新**：新微博能够及时出现在首页
- **排序规则**：按时间倒序排列，支持个性化排序
- **交互功能**：点赞、评论、转发、收藏

##### 2. 非功能需求
- **性能要求**：首页加载时间 < 500ms
- **并发支持**：支持百万级并发用户
- **可用性**：99.9% 可用性
- **扩展性**：支持用户和数据量的快速增长

#### 二、系统架构设计
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   移动端/Web    │    │   CDN           │    │   负载均衡       │
│   - iOS App     │───▶│  - 静态资源     │───▶│   - Nginx       │
│   - Android App │    │  - 图片/视频    │    │   - LVS         │
│   - Web页面     │    │  - 缓存页面     │    │   - HAProxy     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                                        │
                                                        ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   网关层        │    │   业务服务层    │    │   数据存储层    │
│  - API Gateway  │───▶│ - Timeline服务  │───▶│ - MySQL主从     │
│  - 鉴权认证     │    │ - 用户服务      │    │ - Redis集群     │
│  - 限流熔断     │    │ - 微博服务      │    │ - MongoDB       │
│  - 路由转发     │    │ - 推荐服务      │    │ - Elasticsearch │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │   消息中间件    │
                       │  - Kafka        │
                       │  - RabbitMQ     │
                       │  - 事件驱动     │
                       └─────────────────┘
```

#### 三、核心实现方案
##### 1. Timeline服务核心逻辑
```php
<?php
class TimelineService {
    private $redis;
    private $mysql;
    private $userService;
    private $weiboService;
    
    public function __construct() {
        $this->redis = new RedisCluster(['127.0.0.1:7000', '127.0.0.1:7001']);
        $this->mysql = new MySQLCluster();
        $this->userService = new UserService();
        $this->weiboService = new WeiboService();
    }
    
    /**
     * 获取用户Timeline
     */
    public function getUserTimeline($userId, $page = 1, $pageSize = 20) {
        try {
            // 1. 参数验证
            if ($page < 1 || $pageSize > 50) {
                throw new InvalidArgumentException('Invalid page parameters');
            }
            
            // 2. 检查缓存
            $cacheKey = "timeline:{$userId}:page:{$page}";
            $cachedTimeline = $this->redis->get($cacheKey);
            
            if ($cachedTimeline !== false) {
                return json_decode($cachedTimeline, true);
            }
            
            // 3. 获取关注列表
            $followingUsers = $this->userService->getFollowingUsers($userId);
            
            if (empty($followingUsers)) {
                return $this->getRecommendedTimeline($userId, $pageSize);
            }
            
            // 4. 拉取关注用户的微博
            $timeline = $this->fetchTimelineFromFollowing($followingUsers, $page, $pageSize);
            
            // 5. 个性化排序
            $sortedTimeline = $this->personalizedSort($userId, $timeline);
            
            // 6. 补充微博详细信息
            $enrichedTimeline = $this->enrichTimelineData($sortedTimeline);
            
            // 7. 缓存结果
            $this->redis->setex($cacheKey, 300, json_encode($enrichedTimeline)); // 5分钟缓存
            
            return $enrichedTimeline;
            
        } catch (Exception $e) {
            error_log("Get timeline failed: " . $e->getMessage());
            return $this->getFallbackTimeline($userId, $pageSize);
        }
    }
    
    /**
     * 从关注用户拉取微博
     */
    private function fetchTimelineFromFollowing($followingUsers, $page, $pageSize) {
        $offset = ($page - 1) * $pageSize;
        $limit = $pageSize * 2; // 多拉取一些，用于排序后截取
        
        // 构建SQL查询
        $userIds = implode(',', array_map('intval', $followingUsers));
        
        $sql = "
            SELECT w.*, u.nickname, u.avatar, u.verified
            FROM weibos w
            INNER JOIN users u ON w.user_id = u.id
            WHERE w.user_id IN ({$userIds})
              AND w.status = 'published'
              AND w.created_at >= DATE_SUB(NOW(), INTERVAL 7 DAY)
            ORDER BY w.created_at DESC
            LIMIT {$limit} OFFSET {$offset}
        ";
        
        return $this->mysql->query($sql);
    }
    
    /**
     * 个性化排序算法
     */
    private function personalizedSort($userId, $weibos) {
        if (empty($weibos)) {
            return [];
        }
        
        // 获取用户画像
        $userProfile = $this->getUserProfile($userId);
        
        // 计算每条微博的个性化得分
        $scoredWeibos = [];
        foreach ($weibos as $weibo) {
            $score = $this->calculatePersonalizedScore($userId, $weibo, $userProfile);
            $scoredWeibos[] = [
                'weibo' => $weibo,
                'score' => $score
            ];
        }
        
        // 按得分排序
        usort($scoredWeibos, function($a, $b) {
            return $b['score'] <=> $a['score'];
        });
        
        // 提取排序后的微博
        return array_slice(array_column($scoredWeibos, 'weibo'), 0, 20);
    }
    
    /**
     * 计算个性化得分
     */
    private function calculatePersonalizedScore($userId, $weibo, $userProfile) {
        $score = 0;
        
        // 1. 时间因子 (权重: 30%)
        $timeScore = $this->calculateTimeScore($weibo['created_at']);
        $score += $timeScore * 0.3;
        
        // 2. 作者亲密度 (权重: 25%)
        $authorScore = $this->calculateAuthorScore($userId, $weibo['user_id']);
        $score += $authorScore * 0.25;
        
        // 3. 内容热度 (权重: 20%)
        $popularityScore = $this->calculatePopularityScore($weibo);
        $score += $popularityScore * 0.2;
        
        // 4. 内容相关性 (权重: 15%)
        $relevanceScore = $this->calculateRelevanceScore($weibo, $userProfile);
        $score += $relevanceScore * 0.15;
        
        // 5. 互动可能性 (权重: 10%)
        $interactionScore = $this->calculateInteractionScore($userId, $weibo);
        $score += $interactionScore * 0.1;
        
        return $score;
    }
    
    /**
     * 时间得分计算
     */
    private function calculateTimeScore($createdAt) {
        $hoursAgo = (time() - strtotime($createdAt)) / 3600;
        
        // 24小时内的微博得分较高，之后指数衰减
        if ($hoursAgo <= 1) {
            return 1.0;
        } elseif ($hoursAgo <= 6) {
            return 0.8;
        } elseif ($hoursAgo <= 24) {
            return 0.6;
        } else {
            return max(0.1, 0.6 * exp(-($hoursAgo - 24) / 48));
        }
    }
    
    /**
     * 作者亲密度得分
     */
    private function calculateAuthorScore($userId, $authorId) {
        // 从缓存获取互动数据
        $interactionKey = "interaction:{$userId}:{$authorId}";
        $interaction = $this->redis->hgetall($interactionKey);
        
        if (empty($interaction)) {
            // 从数据库加载互动数据
            $interaction = $this->loadUserInteraction($userId, $authorId);
            $this->redis->hmset($interactionKey, $interaction);
            $this->redis->expire($interactionKey, 3600);
        }
        
        $score = 0;
        $score += min(($interaction['like_count'] ?? 0) * 0.1, 0.3);
        $score += min(($interaction['comment_count'] ?? 0) * 0.2, 0.4);
        $score += min(($interaction['repost_count'] ?? 0) * 0.3, 0.5);
        
        return min($score, 1.0);
    }
    
    /**
     * 内容热度得分
     */
    private function calculatePopularityScore($weibo) {
        $likeCount = $weibo['like_count'] ?? 0;
        $commentCount = $weibo['comment_count'] ?? 0;
        $repostCount = $weibo['repost_count'] ?? 0;
        
        // 加权计算热度
        $popularity = $likeCount * 0.1 + $commentCount * 0.3 + $repostCount * 0.6;
        
        // 对数归一化
        return min(log(1 + $popularity) / log(1000), 1.0);
    }
    
    /**
     * 补充Timeline数据
     */
    private function enrichTimelineData($timeline) {
        if (empty($timeline)) {
            return [];
        }
        
        $enriched = [];
        foreach ($timeline as $weibo) {
            // 处理图片URL
            $weibo['images'] = $this->processImageUrls($weibo['images'] ?? []);
            
            // 处理时间显示
            $weibo['time_display'] = $this->formatTimeDisplay($weibo['created_at']);
            
            // 处理内容（@用户、#话题#、链接等）
            $weibo['content_formatted'] = $this->formatContent($weibo['content']);
            
            // 获取互动状态
            $weibo['user_interaction'] = $this->getUserInteractionStatus($weibo['id']);
            
            $enriched[] = $weibo;
        }
        
        return $enriched;
    }
    
    /**
     * 降级方案 - 获取推荐Timeline
     */
    private function getFallbackTimeline($userId, $pageSize) {
        // 返回热门微博作为降级方案
        $sql = "
            SELECT w.*, u.nickname, u.avatar, u.verified
            FROM weibos w
            INNER JOIN users u ON w.user_id = u.id
            WHERE w.status = 'published'
              AND w.created_at >= DATE_SUB(NOW(), INTERVAL 1 DAY)
            ORDER BY (w.like_count + w.comment_count * 2 + w.repost_count * 3) DESC
            LIMIT {$pageSize}
        ";
        
        return $this->mysql->query($sql);
    }
}
```

##### 2. 缓存策略优化
```php
<?php
class TimelineCacheManager {
    private $redis;
    private $localCache;
    
    public function __construct() {
        $this->redis = new RedisCluster(['127.0.0.1:7000']);
        $this->localCache = new APCuCache();
    }
    
    /**
     * 多级缓存获取
     */
    public function getTimeline($userId, $page = 1) {
        $cacheKey = "timeline:{$userId}:page:{$page}";
        
        // L1: 本地缓存 (30秒)
        $timeline = $this->localCache->get($cacheKey);
        if ($timeline !== false) {
            return $timeline;
        }
        
        // L2: Redis缓存 (5分钟)
        $timeline = $this->redis->get($cacheKey);
        if ($timeline !== false) {
            $timeline = json_decode($timeline, true);
            $this->localCache->set($cacheKey, $timeline, 30);
            return $timeline;
        }
        
        return null;
    }
    
    /**
     * 设置多级缓存
     */
    public function setTimeline($userId, $page, $timeline) {
        $cacheKey = "timeline:{$userId}:page:{$page}";
        
        // 设置本地缓存
        $this->localCache->set($cacheKey, $timeline, 30);
        
        // 设置Redis缓存
        $this->redis->setex($cacheKey, 300, json_encode($timeline));
    }
    
    /**
     * 缓存预热
     */
    public function warmupCache($activeUsers) {
        foreach ($activeUsers as $userId) {
            // 异步预热活跃用户的首页Timeline
            $this->asyncWarmup($userId);
        }
    }
    
    /**
     * 缓存失效策略
     */
    public function invalidateUserTimeline($userId) {
        $pattern = "timeline:{$userId}:page:*";
        $keys = $this->redis->keys($pattern);
        
        if (!empty($keys)) {
            $this->redis->del($keys);
        }
        
        // 清理本地缓存
        for ($page = 1; $page <= 5; $page++) {
            $this->localCache->delete("timeline:{$userId}:page:{$page}");
        }
    }
}
```

##### 3. 数据库优化
```php
<?php
class TimelineDataAccess {
    private $masterDb;
    private $slaveDbs;
    
    public function __construct() {
        $this->masterDb = new PDO('mysql:host=master;dbname=weibo', $user, $pass);
        $this->slaveDbs = [
            new PDO('mysql:host=slave1;dbname=weibo', $user, $pass),
            new PDO('mysql:host=slave2;dbname=weibo', $user, $pass),
        ];
    }
    
    /**
     * 读写分离查询
     */
    public function getTimelineWeibos($followingUsers, $page, $pageSize) {
        $offset = ($page - 1) * $pageSize;
        
        // 使用从库查询
        $slaveDb = $this->slaveDbs[array_rand($this->slaveDbs)];
        
        $userIds = implode(',', array_map('intval', $followingUsers));
        
        $sql = "
            SELECT 
                w.id, w.user_id, w.content, w.images, w.like_count, 
                w.comment_count, w.repost_count, w.created_at,
                u.nickname, u.avatar, u.verified
            FROM weibos w
            INNER JOIN users u ON w.user_id = u.id
            WHERE w.user_id IN ({$userIds})
              AND w.status = 'published'
              AND w.created_at >= DATE_SUB(NOW(), INTERVAL 7 DAY)
            ORDER BY w.created_at DESC
            LIMIT {$pageSize} OFFSET {$offset}
        ";
        
        $stmt = $slaveDb->prepare($sql);
        $stmt->execute();
        
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }
    
    /**
     * 批量获取用户互动数据
     */
    public function batchGetUserInteractions($userId, $authorIds) {
        if (empty($authorIds)) {
            return [];
        }
        
        $slaveDb = $this->slaveDbs[array_rand($this->slaveDbs)];
        $authorIdList = implode(',', array_map('intval', $authorIds));
        
        $sql = "
            SELECT author_id, 
                   SUM(CASE WHEN action_type = 'like' THEN 1 ELSE 0 END) as like_count,
                   SUM(CASE WHEN action_type = 'comment' THEN 1 ELSE 0 END) as comment_count,
                   SUM(CASE WHEN action_type = 'repost' THEN 1 ELSE 0 END) as repost_count
            FROM user_interactions 
            WHERE user_id = ? AND author_id IN ({$authorIdList})
              AND created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)
            GROUP BY author_id
        ";
        
        $stmt = $slaveDb->prepare($sql);
        $stmt->execute([$userId]);
        
        $result = [];
        while ($row = $stmt->fetch(PDO::FETCH_ASSOC)) {
            $result[$row['author_id']] = $row;
        }
        
        return $result;
    }
}
```

#### 四、数据库设计
```sql
-- 微博表 (按时间分表)
CREATE TABLE weibos_202401 (
    id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    content TEXT NOT NULL,
    images JSON,
    like_count INT DEFAULT 0,
    comment_count INT DEFAULT 0,
    repost_count INT DEFAULT 0,
    status ENUM('draft', 'published', 'deleted') DEFAULT 'published',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_user_created (user_id, created_at DESC),
    INDEX idx_created_status (created_at DESC, status),
    INDEX idx_hot_weibos (like_count, comment_count, repost_count)
);

-- 用户关注关系表
CREATE TABLE user_follows (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    follower_id BIGINT NOT NULL COMMENT '粉丝ID',
    following_id BIGINT NOT NULL COMMENT '被关注者ID',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE KEY uk_follow_relation (follower_id, following_id),
    INDEX idx_follower (follower_id),
    INDEX idx_following (following_id)
);

-- 用户互动记录表
CREATE TABLE user_interactions (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    weibo_id BIGINT NOT NULL,
    author_id BIGINT NOT NULL,
    action_type ENUM('like', 'comment', 'repost') NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_user_author (user_id, author_id),
    INDEX idx_user_action (user_id, action_type, created_at),
    INDEX idx_weibo_action (weibo_id, action_type)
);

-- 用户画像表
CREATE TABLE user_profiles (
    user_id BIGINT PRIMARY KEY,
    interests JSON COMMENT '兴趣标签',
    active_hours JSON COMMENT '活跃时段',
    interaction_preferences JSON COMMENT '互动偏好',
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_updated (last_updated)
);
```

#### 五、性能优化策略
##### 1. 索引优化
```sql
-- 复合索引优化Timeline查询
CREATE INDEX idx_timeline_query ON weibos (user_id, status, created_at DESC);

-- 覆盖索引减少回表
CREATE INDEX idx_timeline_cover ON weibos (user_id, created_at DESC) 
INCLUDE (id, content, like_count, comment_count, repost_count);

-- 分区索引
ALTER TABLE weibos_202401 
PARTITION BY RANGE (UNIX_TIMESTAMP(created_at)) (
    PARTITION p20240101 VALUES LESS THAN (UNIX_TIMESTAMP('2024-01-02')),
    PARTITION p20240102 VALUES LESS THAN (UNIX_TIMESTAMP('2024-01-03')),
    -- 按天分区
);
```

##### 2. 异步处理
```php
<?php
// 异步Timeline更新
class AsyncTimelineUpdater {
    private $kafka;
    private $redis;
    
    public function __construct() {
        $this->kafka = new KafkaProducer();
        $this->redis = new Redis();
    }
    
    /**
     * 异步更新用户Timeline
     */
    public function updateUserTimeline($userId, $newWeiboId) {
        // 发送到消息队列异步处理
        $this->kafka->send('timeline.update', [
            'user_id' => $userId,
            'weibo_id' => $newWeiboId,
            'action' => 'add',
            'timestamp' => time()
        ]);
    }
    
    /**
     * 批量更新粉丝Timeline
     */
    public function batchUpdateFollowersTimeline($authorId, $weiboId, $followers) {
        $batchSize = 1000;
        $batches = array_chunk($followers, $batchSize);
        
        foreach ($batches as $batch) {
            $this->kafka->send('timeline.batch_update', [
                'author_id' => $authorId,
                'weibo_id' => $weiboId,
                'followers' => $batch,
                'timestamp' => time()
            ]);
        }
    }
}
```

#### 六、监控与告警
```php
<?php
class TimelineMonitor {
    private $prometheus;
    private $alertManager;
    
    public function __construct() {
        $this->prometheus = new PrometheusClient();
        $this->alertManager = new AlertManager();
    }
    
    /**
     * 监控Timeline性能指标
     */
    public function monitorPerformance() {
        // 1. 响应时间监控
        $avgResponseTime = $this->getAverageResponseTime();
        $this->prometheus->histogram('timeline_response_time', $avgResponseTime);
        
        if ($avgResponseTime > 500) {
            $this->alertManager->send('SLOW_TIMELINE', "Timeline响应时间: {$avgResponseTime}ms");
        }
        
        // 2. 缓存命中率监控
        $cacheHitRate = $this->getCacheHitRate();
        $this->prometheus->gauge('timeline_cache_hit_rate', $cacheHitRate);
        
        if ($cacheHitRate < 0.85) {
            $this->alertManager->send('LOW_CACHE_HIT', "缓存命中率: {$cacheHitRate}");
        }
        
        // 3. 数据库查询监控
        $dbQueryTime = $this->getDatabaseQueryTime();
        $this->prometheus->histogram('timeline_db_query_time', $dbQueryTime);
        
        if ($dbQueryTime > 100) {
            $this->alertManager->send('SLOW_DB_QUERY', "数据库查询时间: {$dbQueryTime}ms");
        }
    }
}
```

**总结**：微博首页Timeline系统需要平衡实时性、个性化和性能。通过多级缓存、读写分离、异步处理和智能排序算法，可以在保证用户体验的同时支撑大规模并发访问。关键是要根据用户行为特征进行个性化优化，并建立完善的监控体系确保系统稳定运行。
</details>

### 3. 抢红包算法（微信/支付宝）

<details>
<summary>答案与解析</summary> 

#### 一、需求分析
##### 1. 功能需求
- **红包发放**：支持固定金额和随机金额红包
- **抢红包**：多人同时抢红包，保证公平性
- **金额分配**：随机分配但保证每人都有份
- **实时通知**：抢红包结果实时推送
- **防作弊**：防止重复抢取、恶意攻击

##### 2. 非功能需求
- **高并发**：支持百万级并发抢红包
- **一致性**：保证金额分配的准确性
- **低延迟**：抢红包响应时间 < 100ms
- **高可用**：99.99% 可用性
- **安全性**：防止金额篡改和重复领取

#### 二、系统架构设计
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   客户端        │    │   CDN/网关      │    │   负载均衡       │
│  - 微信/支付宝  │───▶│  - 静态资源     │───▶│   - Nginx       │
│  - H5页面       │    │  - API网关      │    │   - LVS         │
│  - 小程序       │    │  - 限流熔断     │    │   - 一致性哈希   │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                                        │
                                                        ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   业务服务层    │    │   缓存层        │    │   数据存储层    │
│ - 红包服务      │───▶│ - Redis集群     │───▶│ - MySQL主从     │
│ - 用户服务      │    │ - 本地缓存      │    │ - 分库分表      │
│ - 支付服务      │    │ - 预分配缓存    │    │ - 事务日志      │
│ - 通知服务      │    │ - 防重缓存      │    │ - 备份恢复      │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │   消息队列      │
                       │  - Kafka        │
                       │  - 异步通知     │
                       │  - 事件驱动     │
                       └─────────────────┘
```

#### 三、核心算法实现
##### 1. 红包金额分配算法
```php
<?php
class RedPacketAlgorithm {
    
    /**
     * 二倍均值法 - 微信红包算法
     * 保证每个红包金额在 0.01 到 (剩余平均值 * 2) 之间
     */
    public function wechatAlgorithm($totalAmount, $totalCount) {
        if ($totalCount <= 0 || $totalAmount <= 0) {
            throw new InvalidArgumentException('Invalid parameters');
        }
        
        $amounts = [];
        $remainAmount = $totalAmount;
        $remainCount = $totalCount;
        
        for ($i = 0; $i < $totalCount - 1; $i++) {
            // 计算当前可分配的最大金额（剩余平均值的2倍）
            $maxAmount = ($remainAmount / $remainCount) * 2;
            
            // 随机生成金额，最小0.01元
            $currentAmount = mt_rand(1, intval($maxAmount * 100)) / 100;
            $currentAmount = max(0.01, $currentAmount);
            
            $amounts[] = $currentAmount;
            $remainAmount -= $currentAmount;
            $remainCount--;
        }
        
        // 最后一个红包获得剩余金额
        $amounts[] = round($remainAmount, 2);
        
        return $amounts;
    }
    
    /**
     * 线段切割法 - 支付宝红包算法
     * 在线段上随机切割点，保证分配更均匀
     */
    public function alipayAlgorithm($totalAmount, $totalCount) {
        if ($totalCount <= 0 || $totalAmount <= 0) {
            throw new InvalidArgumentException('Invalid parameters');
        }
        
        // 转换为分，避免浮点数精度问题
        $totalCents = intval($totalAmount * 100);
        $minCents = 1; // 最小1分
        
        // 预留每人最少金额
        $reservedCents = $totalCount * $minCents;
        $availableCents = $totalCents - $reservedCents;
        
        if ($availableCents < 0) {
            throw new InvalidArgumentException('Total amount too small');
        }
        
        // 生成随机切割点
        $cutPoints = [];
        for ($i = 0; $i < $totalCount - 1; $i++) {
            $cutPoints[] = mt_rand(0, $availableCents);
        }
        
        // 排序切割点
        sort($cutPoints);
        
        // 计算每段长度
        $amounts = [];
        $lastPoint = 0;
        
        foreach ($cutPoints as $point) {
            $segmentCents = $point - $lastPoint + $minCents;
            $amounts[] = $segmentCents / 100;
            $lastPoint = $point;
        }
        
        // 最后一段
        $lastSegmentCents = $availableCents - $lastPoint + $minCents;
        $amounts[] = $lastSegmentCents / 100;
        
        return $amounts;
    }
    
    /**
     * 正态分布算法 - 更公平的分配
     */
    public function normalDistributionAlgorithm($totalAmount, $totalCount) {
        if ($totalCount <= 0 || $totalAmount <= 0) {
            throw new InvalidArgumentException('Invalid parameters');
        }
        
        $average = $totalAmount / $totalCount;
        $stdDev = $average * 0.3; // 标准差为平均值的30%
        
        $amounts = [];
        $totalGenerated = 0;
        
        for ($i = 0; $i < $totalCount - 1; $i++) {
            // 生成正态分布随机数
            $amount = $this->generateNormalRandom($average, $stdDev);
            
            // 确保金额在合理范围内
            $amount = max(0.01, min($amount, $average * 3));
            
            $amounts[] = round($amount, 2);
            $totalGenerated += $amount;
        }
        
        // 最后一个红包调整总金额
        $lastAmount = $totalAmount - $totalGenerated;
        $amounts[] = max(0.01, round($lastAmount, 2));
        
        return $amounts;
    }
    
    /**
     * 生成正态分布随机数（Box-Muller变换）
     */
    private function generateNormalRandom($mean, $stdDev) {
        static $hasSpare = false;
        static $spare;
        
        if ($hasSpare) {
            $hasSpare = false;
            return $spare * $stdDev + $mean;
        }
        
        $hasSpare = true;
        $u = mt_rand() / mt_getrandmax();
        $v = mt_rand() / mt_getrandmax();
        
        $mag = $stdDev * sqrt(-2.0 * log($u));
        $spare = $mag * cos(2.0 * M_PI * $v);
        
        return $mag * sin(2.0 * M_PI * $v) + $mean;
    }
}
```

##### 2. 红包服务核心实现
```php
<?php
class RedPacketService {
    private $redis;
    private $mysql;
    private $algorithm;
    private $lockService;
    
    public function __construct() {
        $this->redis = new RedisCluster(['127.0.0.1:7000', '127.0.0.1:7001']);
        $this->mysql = new MySQLCluster();
        $this->algorithm = new RedPacketAlgorithm();
        $this->lockService = new DistributedLockService($this->redis);
    }
    
    /**
     * 创建红包
     */
    public function createRedPacket($userId, $totalAmount, $totalCount, $message = '') {
        try {
            // 1. 参数验证
            $this->validateCreateParams($totalAmount, $totalCount);
            
            // 2. 检查用户余额
            if (!$this->checkUserBalance($userId, $totalAmount)) {
                throw new InsufficientBalanceException('Insufficient balance');
            }
            
            // 3. 生成红包ID
            $redPacketId = $this->generateRedPacketId();
            
            // 4. 预分配红包金额
            $amounts = $this->algorithm->wechatAlgorithm($totalAmount, $totalCount);
            
            // 5. 开启事务
            $this->mysql->beginTransaction();
            
            try {
                // 6. 扣除用户余额
                $this->deductUserBalance($userId, $totalAmount);
                
                // 7. 创建红包记录
                $this->insertRedPacketRecord($redPacketId, $userId, $totalAmount, $totalCount, $message);
                
                // 8. 预存红包金额到Redis
                $this->preAllocateAmounts($redPacketId, $amounts);
                
                // 9. 设置红包状态
                $this->setRedPacketStatus($redPacketId, 'active', 3600); // 1小时有效期
                
                $this->mysql->commit();
                
                return [
                    'red_packet_id' => $redPacketId,
                    'total_amount' => $totalAmount,
                    'total_count' => $totalCount,
                    'amounts' => $amounts
                ];
                
            } catch (Exception $e) {
                $this->mysql->rollback();
                throw $e;
            }
            
        } catch (Exception $e) {
            error_log("Create red packet failed: " . $e->getMessage());
            throw $e;
        }
    }
    
    /**
     * 抢红包
     */
    public function grabRedPacket($redPacketId, $userId) {
        $lockKey = "grab_lock:{$redPacketId}";
        $userLockKey = "user_grab:{$redPacketId}:{$userId}";
        
        try {
            // 1. 防重复抢取检查
            if ($this->redis->exists($userLockKey)) {
                throw new DuplicateGrabException('Already grabbed');
            }
            
            // 2. 设置用户防重标记
            $this->redis->setex($userLockKey, 3600, 1);
            
            // 3. 获取分布式锁
            $lock = $this->lockService->acquire($lockKey, 5000); // 5秒超时
            
            if (!$lock) {
                throw new LockTimeoutException('Grab timeout');
            }
            
            try {
                // 4. 检查红包状态
                $redPacketInfo = $this->getRedPacketInfo($redPacketId);
                
                if (!$redPacketInfo || $redPacketInfo['status'] !== 'active') {
                    throw new RedPacketExpiredException('Red packet expired or not exists');
                }
                
                // 5. 检查是否已被抢完
                $remainCount = $this->getRemainCount($redPacketId);
                if ($remainCount <= 0) {
                    throw new RedPacketEmptyException('Red packet is empty');
                }
                
                // 6. 从预分配队列中获取金额
                $amount = $this->popAmount($redPacketId);
                
                if ($amount === false) {
                    throw new RedPacketEmptyException('No amount available');
                }
                
                // 7. 记录抢红包结果
                $grabId = $this->recordGrabResult($redPacketId, $userId, $amount);
                
                // 8. 增加用户余额
                $this->addUserBalance($userId, $amount);
                
                // 9. 更新红包统计
                $this->updateRedPacketStats($redPacketId, $amount);
                
                // 10. 检查是否抢完
                $newRemainCount = $this->getRemainCount($redPacketId);
                if ($newRemainCount <= 0) {
                    $this->setRedPacketStatus($redPacketId, 'finished');
                }
                
                // 11. 异步发送通知
                $this->sendGrabNotification($redPacketId, $userId, $amount);
                
                return [
                    'grab_id' => $grabId,
                    'amount' => $amount,
                    'remain_count' => max(0, $newRemainCount)
                ];
                
            } finally {
                $this->lockService->release($lock);
            }
            
        } catch (Exception $e) {
            // 抢红包失败，清除用户防重标记
            $this->redis->del($userLockKey);
            
            error_log("Grab red packet failed: " . $e->getMessage());
            throw $e;
        }
    }
    
    /**
     * 预分配红包金额到Redis
     */
    private function preAllocateAmounts($redPacketId, $amounts) {
        $queueKey = "amounts:{$redPacketId}";
        
        // 使用Redis List存储预分配金额
        foreach ($amounts as $amount) {
            $this->redis->lpush($queueKey, $amount);
        }
        
        // 设置过期时间
        $this->redis->expire($queueKey, 3600);
        
        // 设置剩余数量
        $countKey = "count:{$redPacketId}";
        $this->redis->set($countKey, count($amounts));
        $this->redis->expire($countKey, 3600);
    }
    
    /**
     * 从队列中弹出金额
     */
    private function popAmount($redPacketId) {
        $queueKey = "amounts:{$redPacketId}";
        $countKey = "count:{$redPacketId}";
        
        // 使用Lua脚本保证原子性
        $luaScript = '
            local queueKey = KEYS[1]
            local countKey = KEYS[2]
            
            local count = redis.call("GET", countKey)
            if not count or tonumber(count) <= 0 then
                return false
            end
            
            local amount = redis.call("RPOP", queueKey)
            if amount then
                redis.call("DECR", countKey)
                return amount
            else
                return false
            end
        ';
        
        return $this->redis->eval($luaScript, [$queueKey, $countKey], 2);
    }
    
    /**
     * 获取剩余红包数量
     */
    private function getRemainCount($redPacketId) {
        $countKey = "count:{$redPacketId}";
        $count = $this->redis->get($countKey);
        return $count !== false ? intval($count) : 0;
    }
    
    /**
     * 记录抢红包结果
     */
    private function recordGrabResult($redPacketId, $userId, $amount) {
        $grabId = $this->generateGrabId();
        
        $sql = "
            INSERT INTO red_packet_grabs 
            (grab_id, red_packet_id, user_id, amount, created_at) 
            VALUES (?, ?, ?, ?, NOW())
        ";
        
        $this->mysql->execute($sql, [$grabId, $redPacketId, $userId, $amount]);
        
        return $grabId;
    }
    
    /**
     * 生成红包ID
     */
    private function generateRedPacketId() {
        // 使用雪花算法或UUID生成唯一ID
        return 'RP' . date('YmdHis') . mt_rand(100000, 999999);
    }
    
    /**
     * 生成抢红包ID
     */
    private function generateGrabId() {
        return 'GR' . date('YmdHis') . mt_rand(100000, 999999);
    }
}
```

##### 3. 分布式锁实现
```php
<?php
class DistributedLockService {
    private $redis;
    
    public function __construct($redis) {
        $this->redis = $redis;
    }
    
    /**
     * 获取分布式锁
     */
    public function acquire($lockKey, $timeout = 5000, $expiry = 10000) {
        $identifier = uniqid();
        $lockKey = "lock:{$lockKey}";
        
        $end = microtime(true) * 1000 + $timeout;
        
        while (microtime(true) * 1000 < $end) {
            // 使用SET命令的NX和PX选项实现原子操作
            $result = $this->redis->set($lockKey, $identifier, ['NX', 'PX' => $expiry]);
            
            if ($result === true) {
                return [
                    'key' => $lockKey,
                    'identifier' => $identifier,
                    'expiry' => $expiry
                ];
            }
            
            // 短暂等待后重试
            usleep(1000); // 1ms
        }
        
        return false;
    }
    
    /**
     * 释放分布式锁
     */
    public function release($lock) {
        if (!$lock || !isset($lock['key'], $lock['identifier'])) {
            return false;
        }
        
        // 使用Lua脚本保证原子性
        $luaScript = '
            if redis.call("GET", KEYS[1]) == ARGV[1] then
                return redis.call("DEL", KEYS[1])
            else
                return 0
            end
        ';
        
        return $this->redis->eval($luaScript, [$lock['key']], [$lock['identifier']]) === 1;
    }
}
```

#### 四、数据库设计
```sql
-- 红包主表
CREATE TABLE red_packets (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    red_packet_id VARCHAR(32) UNIQUE NOT NULL,
    creator_id BIGINT NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL,
    total_count INT NOT NULL,
    grabbed_amount DECIMAL(10,2) DEFAULT 0,
    grabbed_count INT DEFAULT 0,
    message VARCHAR(200) DEFAULT '',
    status ENUM('active', 'finished', 'expired') DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expired_at TIMESTAMP NULL,
    
    INDEX idx_creator (creator_id, created_at),
    INDEX idx_status (status, created_at),
    INDEX idx_red_packet_id (red_packet_id)
);

-- 抢红包记录表
CREATE TABLE red_packet_grabs (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    grab_id VARCHAR(32) UNIQUE NOT NULL,
    red_packet_id VARCHAR(32) NOT NULL,
    user_id BIGINT NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_red_packet (red_packet_id, created_at),
    INDEX idx_user (user_id, created_at),
    INDEX idx_grab_id (grab_id),
    
    FOREIGN KEY (red_packet_id) REFERENCES red_packets(red_packet_id)
);

-- 用户余额表
CREATE TABLE user_balances (
    user_id BIGINT PRIMARY KEY,
    balance DECIMAL(15,2) DEFAULT 0,
    frozen_balance DECIMAL(15,2) DEFAULT 0,
    version INT DEFAULT 0, -- 乐观锁版本号
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_updated (updated_at)
);

-- 余额变动记录表
CREATE TABLE balance_logs (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    balance_before DECIMAL(15,2) NOT NULL,
    balance_after DECIMAL(15,2) NOT NULL,
    type ENUM('red_packet_create', 'red_packet_grab', 'recharge', 'withdraw') NOT NULL,
    ref_id VARCHAR(32) DEFAULT '',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_user_type (user_id, type, created_at),
    INDEX idx_ref (ref_id)
);
```

#### 五、性能优化策略
##### 1. 缓存预热
```php
<?php
class RedPacketCacheWarmer {
    private $redis;
    
    public function __construct($redis) {
        $this->redis = $redis;
    }
    
    /**
     * 预热活跃红包数据
     */
    public function warmupActiveRedPackets() {
        // 获取最近1小时的活跃红包
        $sql = "
            SELECT red_packet_id, total_amount, total_count, grabbed_count
            FROM red_packets 
            WHERE status = 'active' 
              AND created_at >= DATE_SUB(NOW(), INTERVAL 1 HOUR)
            ORDER BY created_at DESC
            LIMIT 1000
        ";
        
        $redPackets = $this->mysql->query($sql);
        
        foreach ($redPackets as $redPacket) {
            $this->warmupRedPacketCache($redPacket);
        }
    }
    
    /**
     * 预热单个红包缓存
     */
    private function warmupRedPacketCache($redPacket) {
        $redPacketId = $redPacket['red_packet_id'];
        
        // 缓存红包基本信息
        $infoKey = "info:{$redPacketId}";
        $this->redis->hmset($infoKey, $redPacket);
        $this->redis->expire($infoKey, 3600);
        
        // 缓存剩余数量
        $remainCount = $redPacket['total_count'] - $redPacket['grabbed_count'];
        $countKey = "count:{$redPacketId}";
        $this->redis->set($countKey, $remainCount);
        $this->redis->expire($countKey, 3600);
    }
}
```

##### 2. 监控告警
```php
<?php
class RedPacketMonitor {
    private $prometheus;
    private $alertManager;
    
    public function __construct() {
        $this->prometheus = new PrometheusClient();
        $this->alertManager = new AlertManager();
    }
    
    /**
     * 监控红包系统指标
     */
    public function monitorMetrics() {
        // 1. 抢红包成功率
        $successRate = $this->getGrabSuccessRate();
        $this->prometheus->gauge('red_packet_success_rate', $successRate);
        
        if ($successRate < 0.95) {
            $this->alertManager->send('LOW_SUCCESS_RATE', "抢红包成功率: {$successRate}");
        }
        
        // 2. 平均响应时间
        $avgResponseTime = $this->getAverageResponseTime();
        $this->prometheus->histogram('red_packet_response_time', $avgResponseTime);
        
        if ($avgResponseTime > 100) {
            $this->alertManager->send('SLOW_RESPONSE', "响应时间: {$avgResponseTime}ms");
        }
        
        // 3. 并发抢红包数
        $concurrentGrabs = $this->getConcurrentGrabCount();
        $this->prometheus->gauge('red_packet_concurrent_grabs', $concurrentGrabs);
        
        if ($concurrentGrabs > 10000) {
            $this->alertManager->send('HIGH_CONCURRENCY', "并发数: {$concurrentGrabs}");
        }
        
        // 4. 锁等待时间
        $lockWaitTime = $this->getAverageLockWaitTime();
        $this->prometheus->histogram('red_packet_lock_wait_time', $lockWaitTime);
        
        if ($lockWaitTime > 50) {
            $this->alertManager->send('LOCK_CONTENTION', "锁等待时间: {$lockWaitTime}ms");
        }
    }
}
```

**总结**：抢红包系统的核心在于算法公平性、高并发处理和数据一致性。通过预分配算法、分布式锁、缓存优化和完善的监控体系，可以构建一个稳定可靠的抢红包系统。关键是要在性能和一致性之间找到平衡，确保用户体验和资金安全。
</details>

### 4. 设计一个短链系统（百度）

<details>
<summary>答案与解析</summary> 

#### 一、需求分析
##### 1. 功能需求
- **短链生成**：将长URL转换为短链接
- **短链还原**：通过短链跳转到原始URL
- **自定义短链**：支持用户自定义短链后缀
- **统计分析**：点击量、来源、时间等统计
- **有效期管理**：支持设置短链有效期
- **批量操作**：批量生成和管理短链

##### 2. 非功能需求
- **高性能**：支持10万+ QPS的访问
- **高可用**：99.9% 可用性
- **低延迟**：跳转响应时间 < 100ms
- **可扩展**：支持千亿级短链存储
- **安全性**：防止恶意链接和滥用

#### 二、系统架构设计
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   客户端        │    │   CDN           │    │   负载均衡       │
│  - 浏览器       │───▶│  - 静态资源     │───▶│   - Nginx       │
│  - 移动App      │    │  - 缓存页面     │    │   - LVS         │
│  - API调用      │    │  - 地理分布     │    │   - 一致性哈希   │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                                        │
                                                        ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   网关层        │    │   业务服务层    │    │   数据存储层    │
│  - API Gateway  │───▶│ - 短链服务      │───▶│ - MySQL集群     │
│  - 限流熔断     │    │ - 统计服务      │    │ - Redis集群     │
│  - 鉴权认证     │    │ - 用户服务      │    │ - MongoDB       │
│  - 路由转发     │    │ - 安全服务      │    │ - ClickHouse    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │   消息队列      │
                       │  - Kafka        │
                       │  - 统计数据     │
                       │  - 异步处理     │
                       └─────────────────┘
```

#### 三、核心算法实现
##### 1. 短链生成算法
```php
<?php
class ShortUrlGenerator {
    
    // Base62字符集
    private const BASE62_CHARS = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
    
    private $redis;
    private $mysql;
    
    public function __construct() {
        $this->redis = new RedisCluster(['127.0.0.1:7000', '127.0.0.1:7001']);
        $this->mysql = new MySQLCluster();
    }
    
    /**
     * 基于自增ID的Base62编码
     */
    public function generateByAutoIncrement($longUrl, $userId = null) {
        try {
            // 1. 检查URL是否已存在
            $existingShortUrl = $this->findExistingShortUrl($longUrl, $userId);
            if ($existingShortUrl) {
                return $existingShortUrl;
            }
            
            // 2. 获取全局自增ID
            $id = $this->getNextId();
            
            // 3. Base62编码
            $shortCode = $this->base62Encode($id);
            
            // 4. 存储映射关系
            $shortUrl = $this->saveShortUrl($shortCode, $longUrl, $userId);
            
            return $shortUrl;
            
        } catch (Exception $e) {
            error_log("Generate short url failed: " . $e->getMessage());
            throw $e;
        }
    }
    
    /**
     * 基于哈希的短链生成
     */
    public function generateByHash($longUrl, $userId = null) {
        try {
            // 1. 检查URL是否已存在
            $existingShortUrl = $this->findExistingShortUrl($longUrl, $userId);
            if ($existingShortUrl) {
                return $existingShortUrl;
            }
            
            // 2. 生成哈希值
            $hash = $this->generateHash($longUrl, $userId);
            
            // 3. 取前6位作为短码
            $shortCode = substr($hash, 0, 6);
            
            // 4. 处理冲突
            $attempts = 0;
            while ($this->shortCodeExists($shortCode) && $attempts < 10) {
                $hash = $this->generateHash($longUrl . $attempts, $userId);
                $shortCode = substr($hash, 0, 6);
                $attempts++;
            }
            
            if ($attempts >= 10) {
                throw new Exception('Failed to generate unique short code');
            }
            
            // 5. 存储映射关系
            $shortUrl = $this->saveShortUrl($shortCode, $longUrl, $userId);
            
            return $shortUrl;
            
        } catch (Exception $e) {
            error_log("Generate short url by hash failed: " . $e->getMessage());
            throw $e;
        }
    }
    
    /**
     * Base62编码
     */
    private function base62Encode($number) {
        if ($number == 0) {
            return self::BASE62_CHARS[0];
        }
        
        $result = '';
        $base = strlen(self::BASE62_CHARS);
        
        while ($number > 0) {
            $remainder = $number % $base;
            $result = self::BASE62_CHARS[$remainder] . $result;
            $number = intval($number / $base);
        }
        
        return $result;
    }
    
    /**
     * 生成哈希值
     */
    private function generateHash($input, $userId = null) {
        $salt = $userId ? "user_{$userId}_" : 'global_';
        return hash('sha256', $salt . $input . microtime(true));
    }
    
    /**
     * 获取全局自增ID
     */
    private function getNextId() {
        // 使用Redis的INCR命令获取全局自增ID
        $key = 'short_url:global_id';
        return $this->redis->incr($key);
    }
    
    /**
     * 检查短码是否存在
     */
    private function shortCodeExists($shortCode) {
        $cacheKey = "short_url:exists:{$shortCode}";
        
        // 先检查缓存
        $exists = $this->redis->get($cacheKey);
        if ($exists !== false) {
            return $exists === '1';
        }
        
        // 检查数据库
        $sql = "SELECT 1 FROM short_urls WHERE short_code = ? LIMIT 1";
        $result = $this->mysql->query($sql, [$shortCode]);
        
        $exists = !empty($result);
        
        // 缓存结果
        $this->redis->setex($cacheKey, 300, $exists ? '1' : '0');
        
        return $exists;
    }
    
    /**
     * 查找已存在的短链
     */
    private function findExistingShortUrl($longUrl, $userId) {
        $urlHash = md5($longUrl);
        $cacheKey = "short_url:mapping:{$urlHash}";
        
        // 先检查缓存
        $shortCode = $this->redis->get($cacheKey);
        if ($shortCode !== false) {
            return $this->buildShortUrl($shortCode);
        }
        
        // 检查数据库
        $sql = "SELECT short_code FROM short_urls WHERE url_hash = ? AND user_id = ? AND status = 'active' LIMIT 1";
        $result = $this->mysql->query($sql, [$urlHash, $userId]);
        
        if (!empty($result)) {
            $shortCode = $result[0]['short_code'];
            
            // 缓存结果
            $this->redis->setex($cacheKey, 3600, $shortCode);
            
            return $this->buildShortUrl($shortCode);
        }
        
        return null;
    }
    
    /**
     * 保存短链映射
     */
    private function saveShortUrl($shortCode, $longUrl, $userId) {
        $urlHash = md5($longUrl);
        $shortUrlId = $this->generateShortUrlId();
        
        // 保存到数据库
        $sql = "
            INSERT INTO short_urls 
            (id, short_code, long_url, url_hash, user_id, created_at, status) 
            VALUES (?, ?, ?, ?, ?, NOW(), 'active')
        ";
        
        $this->mysql->execute($sql, [
            $shortUrlId, $shortCode, $longUrl, $urlHash, $userId
        ]);
        
        // 缓存映射关系
        $this->cacheShortUrlMapping($shortCode, $longUrl, $urlHash);
        
        return $this->buildShortUrl($shortCode);
    }
    
    /**
     * 缓存短链映射
     */
    private function cacheShortUrlMapping($shortCode, $longUrl, $urlHash) {
        // 缓存短码到长链的映射
        $shortToLongKey = "short_url:s2l:{$shortCode}";
        $this->redis->setex($shortToLongKey, 3600, $longUrl);
        
        // 缓存长链到短码的映射
        $longToShortKey = "short_url:mapping:{$urlHash}";
        $this->redis->setex($longToShortKey, 3600, $shortCode);
        
        // 缓存短码存在性
        $existsKey = "short_url:exists:{$shortCode}";
        $this->redis->setex($existsKey, 300, '1');
    }
    
    /**
     * 构建完整短链
     */
    private function buildShortUrl($shortCode) {
        $domain = config('short_url.domain', 'https://t.co');
        return $domain . '/' . $shortCode;
    }
    
    /**
     * 生成短链ID
     */
    private function generateShortUrlId() {
        return 'SU' . date('YmdHis') . mt_rand(100000, 999999);
    }
}
```

##### 2. 短链服务核心实现
```php
<?php
class ShortUrlService {
    private $redis;
    private $mysql;
    private $generator;
    
    public function __construct() {
        $this->redis = new RedisCluster(['127.0.0.1:7000', '127.0.0.1:7001']);
        $this->mysql = new MySQLCluster();
        $this->generator = new ShortUrlGenerator();
    }
    
    /**
     * 创建短链
     */
    public function createShortUrl($longUrl, $options = []) {
        try {
            // 1. 参数验证
            $this->validateUrl($longUrl);
            
            // 2. 安全检查
            if (!$this->isUrlSafe($longUrl)) {
                throw new UnsafeUrlException('URL contains unsafe content');
            }
            
            // 3. 提取参数
            $userId = $options['user_id'] ?? null;
            $customCode = $options['custom_code'] ?? null;
            $expireAt = $options['expire_at'] ?? null;
            $algorithm = $options['algorithm'] ?? 'auto_increment';
            
            // 4. 处理自定义短码
            if ($customCode) {
                return $this->createCustomShortUrl($longUrl, $customCode, $userId, $expireAt);
            }
            
            // 5. 根据算法生成短链
            switch ($algorithm) {
                case 'hash':
                    $shortUrl = $this->generator->generateByHash($longUrl, $userId);
                    break;
                default:
                    $shortUrl = $this->generator->generateByAutoIncrement($longUrl, $userId);
            }
            
            // 6. 设置过期时间
            if ($expireAt) {
                $this->setExpiration($this->extractShortCode($shortUrl), $expireAt);
            }
            
            return [
                'short_url' => $shortUrl,
                'long_url' => $longUrl,
                'created_at' => date('Y-m-d H:i:s'),
                'expire_at' => $expireAt
            ];
            
        } catch (Exception $e) {
            error_log("Create short url failed: " . $e->getMessage());
            throw $e;
        }
    }
    
    /**
     * 短链跳转
     */
    public function redirect($shortCode, $request = []) {
        try {
            // 1. 获取长链
            $longUrl = $this->getLongUrl($shortCode);
            
            if (!$longUrl) {
                throw new ShortUrlNotFoundException('Short URL not found');
            }
            
            // 2. 检查是否过期
            if ($this->isExpired($shortCode)) {
                throw new ShortUrlExpiredException('Short URL has expired');
            }
            
            // 3. 记录访问统计（异步）
            $this->recordAccess($shortCode, $request);
            
            // 4. 返回长链
            return $longUrl;
            
        } catch (Exception $e) {
            error_log("Redirect failed: " . $e->getMessage());
            throw $e;
        }
    }
    
    /**
     * 获取长链
     */
    private function getLongUrl($shortCode) {
        $cacheKey = "short_url:s2l:{$shortCode}";
        
        // 先检查缓存
        $longUrl = $this->redis->get($cacheKey);
        if ($longUrl !== false) {
            return $longUrl;
        }
        
        // 查询数据库
        $sql = "SELECT long_url FROM short_urls WHERE short_code = ? AND status = 'active' LIMIT 1";
        $result = $this->mysql->query($sql, [$shortCode]);
        
        if (empty($result)) {
            return null;
        }
        
        $longUrl = $result[0]['long_url'];
        
        // 缓存结果
        $this->redis->setex($cacheKey, 3600, $longUrl);
        
        return $longUrl;
    }
    
    /**
     * 记录访问统计
     */
    private function recordAccess($shortCode, $request) {
        // 异步记录，不影响跳转性能
        $accessData = [
            'short_code' => $shortCode,
            'ip' => $request['ip'] ?? '',
            'user_agent' => $request['user_agent'] ?? '',
            'referer' => $request['referer'] ?? '',
            'timestamp' => time()
        ];
        
        // 发送到消息队列
        $this->sendToQueue('url_access', $accessData);
        
        // 实时计数器
        $this->incrementAccessCount($shortCode);
    }
    
    /**
     * 增加访问计数
     */
    private function incrementAccessCount($shortCode) {
        // Redis计数器
        $countKey = "short_url:count:{$shortCode}";
        $this->redis->incr($countKey);
        
        // 每日计数器
        $dailyKey = "short_url:daily:" . date('Y-m-d') . ":{$shortCode}";
        $this->redis->incr($dailyKey);
        $this->redis->expire($dailyKey, 86400 * 7); // 保留7天
    }
    
    /**
     * 验证URL安全性
     */
    private function isUrlSafe($url) {
        // 检查恶意域名黑名单
        $blacklist = $this->getBlacklistDomains();
        $domain = parse_url($url, PHP_URL_HOST);
        
        if (in_array($domain, $blacklist)) {
            return false;
        }
        
        // 检查URL模式
        $dangerousPatterns = [
            '/javascript:/i',
            '/data:/i',
            '/vbscript:/i'
        ];
        
        foreach ($dangerousPatterns as $pattern) {
            if (preg_match($pattern, $url)) {
                return false;
            }
        }
        
        return true;
    }
}
```

#### 四、数据库设计
```sql
-- 短链主表
CREATE TABLE short_urls (
    id VARCHAR(32) PRIMARY KEY,
    short_code VARCHAR(10) UNIQUE NOT NULL,
    long_url TEXT NOT NULL,
    url_hash VARCHAR(32) NOT NULL,
    user_id BIGINT,
    title VARCHAR(200) DEFAULT '',
    description TEXT DEFAULT '',
    status ENUM('active', 'inactive', 'deleted') DEFAULT 'active',
    access_count BIGINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    expire_at TIMESTAMP NULL,
    
    INDEX idx_short_code (short_code),
    INDEX idx_url_hash (url_hash),
    INDEX idx_user (user_id, created_at),
    INDEX idx_status (status, created_at),
    INDEX idx_expire (expire_at)
);

-- 访问统计表（ClickHouse）
CREATE TABLE url_analytics (
    short_code String,
    access_time DateTime,
    ip String,
    country String,
    city String,
    browser String,
    os String,
    device String,
    referer String,
    referer_domain String
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(access_time)
ORDER BY (short_code, access_time)
TTL access_time + INTERVAL 1 YEAR;

-- 黑名单域名表
CREATE TABLE blacklist_domains (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    domain VARCHAR(255) NOT NULL,
    reason VARCHAR(500) DEFAULT '',
    status ENUM('active', 'inactive') DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE KEY uk_domain (domain),
    INDEX idx_status (status)
);
```

#### 五、性能优化策略
##### 1. 缓存策略
```php
<?php
class ShortUrlCacheManager {
    private $redis;
    private $localCache;
    
    public function __construct() {
        $this->redis = new RedisCluster(['127.0.0.1:7000']);
        $this->localCache = new APCuCache();
    }
    
    /**
     * 多级缓存获取长链
     */
    public function getLongUrl($shortCode) {
        // L1: 本地缓存 (1分钟)
        $longUrl = $this->localCache->get("s2l:{$shortCode}");
        if ($longUrl !== false) {
            return $longUrl;
        }
        
        // L2: Redis缓存 (1小时)
        $longUrl = $this->redis->get("short_url:s2l:{$shortCode}");
        if ($longUrl !== false) {
            $this->localCache->set("s2l:{$shortCode}", $longUrl, 60);
            return $longUrl;
        }
        
        return null;
    }
    
    /**
     * 预热热门短链
     */
    public function warmupHotUrls() {
        // 获取访问量前1000的短链
        $sql = "
            SELECT short_code, long_url 
            FROM short_urls 
            WHERE status = 'active' 
              AND access_count > 100
            ORDER BY access_count DESC 
            LIMIT 1000
        ";
        
        $hotUrls = $this->mysql->query($sql);
        
        foreach ($hotUrls as $url) {
            $this->redis->setex(
                "short_url:s2l:{$url['short_code']}", 
                3600, 
                $url['long_url']
            );
        }
    }
}
```

##### 2. 分库分表策略
```php
<?php
class ShortUrlSharding {
    private $shardCount = 1024;
    
    /**
     * 根据短码计算分片
     */
    public function getShardByShortCode($shortCode) {
        $hash = crc32($shortCode);
        return abs($hash) % $this->shardCount;
    }
    
    /**
     * 根据用户ID计算分片
     */
    public function getShardByUserId($userId) {
        return $userId % $this->shardCount;
    }
    
    /**
     * 获取分片数据库连接
     */
    public function getShardConnection($shardId) {
        $dbIndex = intval($shardId / 128); // 每个DB包含128个分片
        $tableIndex = $shardId % 128;
        
        return [
            'db' => "short_url_db_{$dbIndex}",
            'table' => "short_urls_{$tableIndex}"
        ];
    }
}
```

**总结**：短链系统的核心在于高效的编码算法、快速的查询性能和完善的统计分析。通过合理的缓存策略、数据库优化和分布式架构，可以构建一个高性能、高可用的短链服务。关键是要在存储效率、查询性能和功能完整性之间找到平衡。
</details>

### 5. 设计一个分布式缓存系统（阿里/腾讯/字节）

<details>
<summary>答案与解析</summary> 

#### 一、需求分析
##### 1. 功能需求
- **数据存储**：支持key-value存储，多种数据类型（String、Hash、List、Set、ZSet）
- **分布式架构**：支持集群模式，数据分片和副本
- **高可用性**：主从复制、故障转移、数据持久化
- **缓存策略**：LRU、LFU、TTL等过期策略
- **一致性保证**：最终一致性、强一致性选项
- **监控运维**：性能监控、日志记录、运维工具

##### 2. 非功能需求
- **高性能**：支持百万级QPS，毫秒级响应
- **高可用**：99.99% 可用性，故障自动恢复
- **可扩展**：支持水平扩展，动态增减节点
- **数据安全**：访问控制、数据加密、备份恢复

#### 二、系统架构设计
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   客户端层      │    │   代理层        │    │   集群管理层    │
│  - 应用程序     │───▶│  - 负载均衡     │───▶│  - 配置中心     │
│  - SDK客户端    │    │  - 路由转发     │    │  - 服务发现     │
│  - 连接池       │    │  - 故障检测     │    │  - 健康检查     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │                        │
                                ▼                        ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   缓存节点层    │    │   存储引擎层    │    │   持久化层      │
│  - Master节点   │───▶│  - 内存管理     │───▶│  - RDB快照      │
│  - Slave节点    │    │  - 数据结构     │    │  - AOF日志      │
│  - 哨兵节点     │    │  - 过期策略     │    │  - 增量备份     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │   监控告警层    │
                       │  - 性能监控     │
                       │  - 日志收集     │
                       │  - 告警通知     │
                       └─────────────────┘
```

#### 三、核心组件实现
##### 1. 分布式缓存核心引擎
```php
<?php
class DistributedCacheEngine {
    private $nodes = [];
    private $hashRing;
    private $replicationFactor = 3;
    private $consistentHash;
    
    public function __construct($config) {
        $this->initializeNodes($config['nodes']);
        $this->consistentHash = new ConsistentHash();
        $this->setupHashRing();
    }
    
    /**
     * 设置缓存数据
     */
    public function set($key, $value, $ttl = 0) {
        try {
            // 1. 计算key的哈希值
            $hash = $this->hashKey($key);
            
            // 2. 找到负责的节点
            $primaryNode = $this->consistentHash->getNode($hash);
            $replicaNodes = $this->getReplicaNodes($primaryNode, $this->replicationFactor - 1);
            
            // 3. 构建缓存项
            $cacheItem = [
                'key' => $key,
                'value' => $this->serialize($value),
                'ttl' => $ttl > 0 ? time() + $ttl : 0,
                'version' => $this->generateVersion(),
                'created_at' => microtime(true)
            ];
            
            // 4. 写入主节点
            $success = $this->writeToNode($primaryNode, $cacheItem);
            if (!$success) {
                throw new CacheWriteException("Failed to write to primary node: {$primaryNode}");
            }
            
            // 5. 异步复制到副本节点
            $this->replicateAsync($replicaNodes, $cacheItem);
            
            // 6. 记录操作日志
            $this->logOperation('SET', $key, $primaryNode, $replicaNodes);
            
            return true;
            
        } catch (Exception $e) {
            error_log("Cache set failed: " . $e->getMessage());
            throw $e;
        }
    }
    
    /**
     * 获取缓存数据
     */
    public function get($key) {
        try {
            // 1. 计算key的哈希值
            $hash = $this->hashKey($key);
            
            // 2. 找到负责的节点
            $primaryNode = $this->consistentHash->getNode($hash);
            $replicaNodes = $this->getReplicaNodes($primaryNode, $this->replicationFactor - 1);
            
            // 3. 尝试从主节点读取
            $cacheItem = $this->readFromNode($primaryNode, $key);
            
            // 4. 主节点失败时从副本节点读取
            if ($cacheItem === null) {
                foreach ($replicaNodes as $replicaNode) {
                    $cacheItem = $this->readFromNode($replicaNode, $key);
                    if ($cacheItem !== null) {
                        // 异步修复主节点数据
                        $this->repairAsync($primaryNode, $cacheItem);
                        break;
                    }
                }
            }
            
            // 5. 检查数据是否过期
            if ($cacheItem && $this->isExpired($cacheItem)) {
                $this->delete($key);
                return null;
            }
            
            // 6. 反序列化并返回
            return $cacheItem ? $this->unserialize($cacheItem['value']) : null;
            
        } catch (Exception $e) {
            error_log("Cache get failed: " . $e->getMessage());
            return null;
        }
    }
    
    /**
     * 删除缓存数据
     */
    public function delete($key) {
        try {
            $hash = $this->hashKey($key);
            $primaryNode = $this->consistentHash->getNode($hash);
            $replicaNodes = $this->getReplicaNodes($primaryNode, $this->replicationFactor - 1);
            
            // 删除主节点数据
            $this->deleteFromNode($primaryNode, $key);
            
            // 删除副本节点数据
            foreach ($replicaNodes as $replicaNode) {
                $this->deleteFromNode($replicaNode, $key);
            }
            
            return true;
            
        } catch (Exception $e) {
            error_log("Cache delete failed: " . $e->getMessage());
            return false;
        }
    }
    
    /**
     * 批量操作
     */
    public function mget($keys) {
        $results = [];
        $nodeKeys = [];
        
        // 按节点分组keys
        foreach ($keys as $key) {
            $hash = $this->hashKey($key);
            $node = $this->consistentHash->getNode($hash);
            $nodeKeys[$node][] = $key;
        }
        
        // 并行从各节点获取数据
        $promises = [];
        foreach ($nodeKeys as $node => $nodeKeyList) {
            $promises[$node] = $this->batchReadFromNodeAsync($node, $nodeKeyList);
        }
        
        // 收集结果
        foreach ($promises as $node => $promise) {
            $nodeResults = $promise->wait();
            $results = array_merge($results, $nodeResults);
        }
        
        return $results;
    }
    
    /**
     * 一致性哈希计算
     */
    private function hashKey($key) {
        return crc32($key);
    }
    
    /**
     * 获取副本节点
     */
    private function getReplicaNodes($primaryNode, $count) {
        $replicas = [];
        $allNodes = array_keys($this->nodes);
        $primaryIndex = array_search($primaryNode, $allNodes);
        
        for ($i = 1; $i <= $count; $i++) {
            $replicaIndex = ($primaryIndex + $i) % count($allNodes);
            $replicas[] = $allNodes[$replicaIndex];
        }
        
        return $replicas;
    }
    
    /**
     * 写入节点
     */
    private function writeToNode($node, $cacheItem) {
        try {
            $connection = $this->getNodeConnection($node);
            
            // 使用Redis协议写入
            $key = $cacheItem['key'];
            $value = json_encode($cacheItem);
            $ttl = $cacheItem['ttl'];
            
            if ($ttl > 0) {
                return $connection->setex($key, $ttl - time(), $value);
            } else {
                return $connection->set($key, $value);
            }
            
        } catch (Exception $e) {
            error_log("Write to node {$node} failed: " . $e->getMessage());
            return false;
        }
    }
    
    /**
     * 从节点读取
     */
    private function readFromNode($node, $key) {
        try {
            $connection = $this->getNodeConnection($node);
            $value = $connection->get($key);
            
            return $value ? json_decode($value, true) : null;
            
        } catch (Exception $e) {
            error_log("Read from node {$node} failed: " . $e->getMessage());
            return null;
        }
    }
    
    /**
     * 异步复制
     */
    private function replicateAsync($replicaNodes, $cacheItem) {
        foreach ($replicaNodes as $replicaNode) {
            // 使用消息队列异步复制
            $this->sendToQueue('cache_replication', [
                'node' => $replicaNode,
                'operation' => 'SET',
                'data' => $cacheItem
            ]);
        }
    }
    
    /**
     * 检查是否过期
     */
    private function isExpired($cacheItem) {
        return $cacheItem['ttl'] > 0 && time() > $cacheItem['ttl'];
    }
    
    /**
     * 序列化数据
     */
    private function serialize($value) {
        return serialize($value);
    }
    
    /**
     * 反序列化数据
     */
    private function unserialize($value) {
        return unserialize($value);
    }
    
    /**
     * 生成版本号
     */
    private function generateVersion() {
        return microtime(true) . '_' . mt_rand(1000, 9999);
    }
}
```

##### 2. 一致性哈希实现
```php
<?php
class ConsistentHash {
    private $ring = [];
    private $nodes = [];
    private $virtualNodes = 150; // 每个物理节点的虚拟节点数
    
    /**
     * 添加节点
     */
    public function addNode($node) {
        $this->nodes[] = $node;
        
        // 为每个物理节点创建虚拟节点
        for ($i = 0; $i < $this->virtualNodes; $i++) {
            $virtualNode = $node . ':' . $i;
            $hash = $this->hash($virtualNode);
            $this->ring[$hash] = $node;
        }
        
        // 排序哈希环
        ksort($this->ring);
    }
    
    /**
     * 移除节点
     */
    public function removeNode($node) {
        // 移除所有虚拟节点
        for ($i = 0; $i < $this->virtualNodes; $i++) {
            $virtualNode = $node . ':' . $i;
            $hash = $this->hash($virtualNode);
            unset($this->ring[$hash]);
        }
        
        // 从节点列表中移除
        $this->nodes = array_filter($this->nodes, function($n) use ($node) {
            return $n !== $node;
        });
    }
    
    /**
     * 获取负责的节点
     */
    public function getNode($key) {
        if (empty($this->ring)) {
            return null;
        }
        
        $hash = is_numeric($key) ? $key : $this->hash($key);
        
        // 在哈希环中查找第一个大于等于key哈希值的节点
        foreach ($this->ring as $ringHash => $node) {
            if ($ringHash >= $hash) {
                return $node;
            }
        }
        
        // 如果没找到，返回第一个节点（环形结构）
        return reset($this->ring);
    }
    
    /**
     * 哈希函数
     */
    private function hash($key) {
        return crc32($key);
    }
    
    /**
     * 获取所有节点
     */
    public function getNodes() {
        return $this->nodes;
    }
    
    /**
     * 获取哈希环状态
     */
    public function getRingStatus() {
        $status = [];
        foreach ($this->ring as $hash => $node) {
            $status[] = [
                'hash' => $hash,
                'node' => $node
            ];
        }
        return $status;
    }
}
```

##### 3. 故障检测和自动恢复
```php
<?php
class CacheFailureDetector {
    private $nodes = [];
    private $healthStatus = [];
    private $checkInterval = 5; // 5秒检查一次
    private $failureThreshold = 3; // 连续失败3次认为节点故障
    
    public function __construct($nodes) {
        $this->nodes = $nodes;
        $this->initHealthStatus();
    }
    
    /**
     * 启动健康检查
     */
    public function startHealthCheck() {
        while (true) {
            foreach ($this->nodes as $node) {
                $this->checkNodeHealth($node);
            }
            
            sleep($this->checkInterval);
        }
    }
    
    /**
     * 检查节点健康状态
     */
    private function checkNodeHealth($node) {
        try {
            $connection = $this->getNodeConnection($node);
            $startTime = microtime(true);
            
            // 发送PING命令
            $result = $connection->ping();
            $responseTime = (microtime(true) - $startTime) * 1000;
            
            if ($result === 'PONG') {
                $this->markNodeHealthy($node, $responseTime);
            } else {
                $this->markNodeUnhealthy($node, 'Invalid ping response');
            }
            
        } catch (Exception $e) {
            $this->markNodeUnhealthy($node, $e->getMessage());
        }
    }
    
    /**
     * 标记节点健康
     */
    private function markNodeHealthy($node, $responseTime) {
        $this->healthStatus[$node] = [
            'status' => 'healthy',
            'last_check' => time(),
            'response_time' => $responseTime,
            'failure_count' => 0,
            'last_error' => null
        ];
        
        // 如果节点之前是故障状态，触发恢复事件
        if (isset($this->healthStatus[$node]['previous_status']) && 
            $this->healthStatus[$node]['previous_status'] === 'failed') {
            $this->onNodeRecovered($node);
        }
    }
    
    /**
     * 标记节点不健康
     */
    private function markNodeUnhealthy($node, $error) {
        if (!isset($this->healthStatus[$node])) {
            $this->healthStatus[$node] = ['failure_count' => 0];
        }
        
        $this->healthStatus[$node]['failure_count']++;
        $this->healthStatus[$node]['last_error'] = $error;
        $this->healthStatus[$node]['last_check'] = time();
        
        // 达到故障阈值时标记为故障
        if ($this->healthStatus[$node]['failure_count'] >= $this->failureThreshold) {
            $this->healthStatus[$node]['status'] = 'failed';
            $this->onNodeFailed($node);
        }
    }
    
    /**
     * 节点故障处理
     */
    private function onNodeFailed($node) {
        error_log("Node {$node} marked as failed");
        
        // 从一致性哈希环中移除节点
        $this->removeNodeFromHashRing($node);
        
        // 触发数据迁移
        $this->triggerDataMigration($node);
        
        // 发送告警
        $this->sendAlert('NODE_FAILED', "Cache node {$node} has failed");
    }
    
    /**
     * 节点恢复处理
     */
    private function onNodeRecovered($node) {
        error_log("Node {$node} recovered");
        
        // 重新加入一致性哈希环
        $this->addNodeToHashRing($node);
        
        // 发送恢复通知
        $this->sendAlert('NODE_RECOVERED', "Cache node {$node} has recovered");
    }
    
    /**
     * 获取健康状态
     */
    public function getHealthStatus() {
        return $this->healthStatus;
    }
    
    /**
     * 获取健康的节点列表
     */
    public function getHealthyNodes() {
        $healthyNodes = [];
        foreach ($this->healthStatus as $node => $status) {
            if ($status['status'] === 'healthy') {
                $healthyNodes[] = $node;
            }
        }
        return $healthyNodes;
    }
}
```

#### 四、数据持久化策略
```php
<?php
class CachePersistenceManager {
    private $persistenceType;
    private $config;
    
    public function __construct($config) {
        $this->config = $config;
        $this->persistenceType = $config['persistence_type'] ?? 'hybrid';
    }
    
    /**
     * RDB快照持久化
     */
    public function createSnapshot($node) {
        try {
            $connection = $this->getNodeConnection($node);
            $snapshotFile = $this->getSnapshotPath($node);
            
            // 创建快照
            $connection->bgsave();
            
            // 等待快照完成
            while ($connection->lastsave() === $connection->lastsave()) {
                usleep(100000); // 100ms
            }
            
            // 压缩快照文件
            $this->compressSnapshot($snapshotFile);
            
            // 上传到备份存储
            $this->uploadToBackupStorage($snapshotFile, $node);
            
            return true;
            
        } catch (Exception $e) {
            error_log("Create snapshot failed for node {$node}: " . $e->getMessage());
            return false;
        }
    }
    
    /**
     * AOF日志持久化
     */
    public function enableAOF($node) {
        try {
            $connection = $this->getNodeConnection($node);
            
            // 启用AOF
            $connection->config('SET', 'appendonly', 'yes');
            $connection->config('SET', 'appendfsync', 'everysec');
            
            // 设置AOF重写策略
            $connection->config('SET', 'auto-aof-rewrite-percentage', '100');
            $connection->config('SET', 'auto-aof-rewrite-min-size', '64mb');
            
            return true;
            
        } catch (Exception $e) {
            error_log("Enable AOF failed for node {$node}: " . $e->getMessage());
            return false;
        }
    }
    
    /**
     * 混合持久化
     */
    public function enableHybridPersistence($node) {
        try {
            // 启用RDB+AOF混合模式
            $connection = $this->getNodeConnection($node);
            $connection->config('SET', 'aof-use-rdb-preamble', 'yes');
            
            // 定期创建快照
            $this->scheduleSnapshot($node);
            
            // 启用AOF
            $this->enableAOF($node);
            
            return true;
            
        } catch (Exception $e) {
            error_log("Enable hybrid persistence failed for node {$node}: " . $e->getMessage());
            return false;
        }
    }
    
    /**
     * 数据恢复
     */
    public function restoreFromBackup($node, $backupFile) {
        try {
            // 停止节点
            $this->stopNode($node);
            
            // 下载备份文件
            $localBackupFile = $this->downloadFromBackupStorage($backupFile);
            
            // 解压备份文件
            $this->decompressBackup($localBackupFile);
            
            // 替换数据文件
            $this->replaceDataFile($node, $localBackupFile);
            
            // 启动节点
            $this->startNode($node);
            
            return true;
            
        } catch (Exception $e) {
            error_log("Restore from backup failed for node {$node}: " . $e->getMessage());
            return false;
        }
    }
}
```

#### 五、性能监控和优化
```php
<?php
class CachePerformanceMonitor {
    private $metrics = [];
    private $prometheus;
    
    public function __construct() {
        $this->prometheus = new PrometheusClient();
        $this->initMetrics();
    }
    
    /**
     * 收集性能指标
     */
    public function collectMetrics($node) {
        try {
            $connection = $this->getNodeConnection($node);
            $info = $connection->info();
            
            // 内存使用情况
            $this->recordMemoryMetrics($node, $info);
            
            // 连接数统计
            $this->recordConnectionMetrics($node, $info);
            
            // 命令执行统计
            $this->recordCommandMetrics($node, $info);
            
            // 键空间统计
            $this->recordKeyspaceMetrics($node, $info);
            
            // 网络IO统计
            $this->recordNetworkMetrics($node, $info);
            
        } catch (Exception $e) {
            error_log("Collect metrics failed for node {$node}: " . $e->getMessage());
        }
    }
    
    /**
     * 记录内存指标
     */
    private function recordMemoryMetrics($node, $info) {
        $usedMemory = $info['used_memory'];
        $maxMemory = $info['maxmemory'];
        $memoryUsageRatio = $maxMemory > 0 ? $usedMemory / $maxMemory : 0;
        
        $this->prometheus->gauge('cache_memory_used_bytes', $usedMemory, ['node' => $node]);
        $this->prometheus->gauge('cache_memory_usage_ratio', $memoryUsageRatio, ['node' => $node]);
        
        // 内存碎片率
        $fragRatio = $info['mem_fragmentation_ratio'] ?? 1.0;
        $this->prometheus->gauge('cache_memory_fragmentation_ratio', $fragRatio, ['node' => $node]);
    }
    
    /**
     * 记录连接指标
     */
    private function recordConnectionMetrics($node, $info) {
        $connectedClients = $info['connected_clients'];
        $blockedClients = $info['blocked_clients'] ?? 0;
        
        $this->prometheus->gauge('cache_connected_clients', $connectedClients, ['node' => $node]);
        $this->prometheus->gauge('cache_blocked_clients', $blockedClients, ['node' => $node]);
    }
    
    /**
     * 记录命令指标
     */
    private function recordCommandMetrics($node, $info) {
        $totalCommands = $info['total_commands_processed'];
        $opsPerSec = $info['instantaneous_ops_per_sec'];
        
        $this->prometheus->counter('cache_commands_total', $totalCommands, ['node' => $node]);
        $this->prometheus->gauge('cache_ops_per_second', $opsPerSec, ['node' => $node]);
        
        // 命令延迟统计
        $this->recordLatencyMetrics($node);
    }
    
    /**
     * 记录延迟指标
     */
    private function recordLatencyMetrics($node) {
        $connection = $this->getNodeConnection($node);
        
        // 测试GET命令延迟
        $startTime = microtime(true);
        $connection->get('__latency_test__');
        $getLatency = (microtime(true) - $startTime) * 1000;
        
        // 测试SET命令延迟
        $startTime = microtime(true);
        $connection->set('__latency_test__', 'test', 1);
        $setLatency = (microtime(true) - $startTime) * 1000;
        
        $this->prometheus->histogram('cache_get_latency_ms', $getLatency, ['node' => $node]);
        $this->prometheus->histogram('cache_set_latency_ms', $setLatency, ['node' => $node]);
    }
    
    /**
     * 性能优化建议
     */
    public function getOptimizationSuggestions($node) {
        $suggestions = [];
        $metrics = $this->getNodeMetrics($node);
        
        // 内存使用率过高
        if ($metrics['memory_usage_ratio'] > 0.8) {
            $suggestions[] = [
                'type' => 'memory',
                'level' => 'warning',
                'message' => '内存使用率超过80%，建议增加内存或启用数据淘汰策略'
            ];
        }
        
        // 内存碎片率过高
        if ($metrics['memory_fragmentation_ratio'] > 1.5) {
            $suggestions[] = [
                'type' => 'memory',
                'level' => 'info',
                'message' => '内存碎片率较高，建议执行内存整理'
            ];
        }
        
        // 连接数过多
        if ($metrics['connected_clients'] > 1000) {
            $suggestions[] = [
                'type' => 'connection',
                'level' => 'warning',
                'message' => '连接数过多，建议使用连接池或增加节点'
            ];
        }
        
        // 延迟过高
        if ($metrics['avg_latency'] > 10) {
            $suggestions[] = [
                'type' => 'performance',
                'level' => 'critical',
                'message' => '平均延迟超过10ms，需要检查网络和负载情况'
            ];
        }
        
        return $suggestions;
    }
}
```

**总结**：分布式缓存系统的核心在于数据分片、副本管理、故障恢复和性能优化。通过一致性哈希、多副本机制、健康检查和完善的监控体系，可以构建一个高性能、高可用的分布式缓存服务。关键是要在一致性、可用性和分区容错性之间找到合适的平衡点。
</details>

### 6. 设计一个消息队列系统（阿里/腾讯/字节）

<details>
<summary>答案与解析</summary>

#### 一、需求分析
##### 1. 功能需求
- **消息发布订阅**：支持点对点和发布订阅模式
- **消息持久化**：消息可靠存储，防止丢失
- **消息顺序**：支持全局顺序和分区顺序
- **消息过滤**：支持基于内容的消息过滤
- **死信队列**：处理无法消费的消息
- **消息重试**：支持消息重试机制
- **集群管理**：支持多节点集群部署

##### 2. 非功能需求
- **高吞吐量**：支持百万级消息/秒
- **低延迟**：毫秒级消息传递延迟
- **高可用性**：99.99% 可用性保证
- **水平扩展**：支持动态扩容缩容
- **数据一致性**：至少一次、最多一次、恰好一次语义

#### 二、系统架构设计
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   生产者层      │    │   代理层        │    │   消费者层      │
│  - 应用程序     │───▶│  - 负载均衡     │───▶│  - 消费者组     │
│  - SDK客户端    │    │  - 路由策略     │    │  - 消息处理器   │
│  - 批量发送     │    │  - 消息缓冲     │    │  - 确认机制     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │                        │
                                ▼                        ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Broker集群    │    │   存储引擎      │    │   协调服务      │
│  - Leader节点   │───▶│  - 消息日志     │───▶│  - ZooKeeper    │
│  - Follower节点 │    │  - 索引管理     │    │  - 元数据管理   │
│  - 分区管理     │    │  - 压缩清理     │    │  - 选举协调     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │   监控管理      │
                       │  - 性能监控     │
                       │  - 日志收集     │
                       │  - 告警通知     │
                       └─────────────────┘
```

#### 三、核心组件实现
##### 1. 消息队列核心引擎
```php
<?php
class MessageQueueEngine {
    private $brokers = [];
    private $topics = [];
    private $partitions = [];
    private $replicationFactor = 3;
    private $storage;
    
    public function __construct($config) {
        $this->initializeBrokers($config['brokers']);
        $this->storage = new MessageStorage($config['storage']);
        $this->coordinator = new ClusterCoordinator($config['zookeeper']);
    }
    
    /**
     * 发送消息
     */
    public function sendMessage($topic, $message, $options = []) {
        try {
            // 1. 验证topic是否存在
            if (!$this->topicExists($topic)) {
                $this->createTopic($topic, $options);
            }
            
            // 2. 选择分区
            $partition = $this->selectPartition($topic, $message, $options);
            
            // 3. 构建消息对象
            $messageObj = $this->buildMessage($message, $options);
            
            // 4. 获取分区的Leader Broker
            $leaderBroker = $this->getPartitionLeader($topic, $partition);
            
            // 5. 写入消息到Leader
            $offset = $this->writeToLeader($leaderBroker, $topic, $partition, $messageObj);
            
            // 6. 等待副本同步（根据acks配置）
            if ($options['acks'] === 'all') {
                $this->waitForReplication($topic, $partition, $offset);
            }
            
            // 7. 返回消息元数据
            return [
                'topic' => $topic,
                'partition' => $partition,
                'offset' => $offset,
                'timestamp' => $messageObj['timestamp']
            ];
            
        } catch (Exception $e) {
            error_log("Send message failed: " . $e->getMessage());
            throw new MessageSendException($e->getMessage());
        }
    }
    
    /**
     * 消费消息
     */
    public function consumeMessages($consumerGroup, $topics, $options = []) {
        try {
            // 1. 注册消费者组
            $consumerId = $this->registerConsumer($consumerGroup, $options);
            
            // 2. 分配分区
            $assignedPartitions = $this->assignPartitions($consumerGroup, $topics);
            
            // 3. 获取消费位置
            $offsets = $this->getConsumerOffsets($consumerGroup, $assignedPartitions);
            
            $messages = [];
            
            // 4. 从各分区拉取消息
            foreach ($assignedPartitions as $topicPartition) {
                $topic = $topicPartition['topic'];
                $partition = $topicPartition['partition'];
                $offset = $offsets[$topic][$partition] ?? 0;
                
                // 获取分区的Leader Broker
                $leaderBroker = $this->getPartitionLeader($topic, $partition);
                
                // 拉取消息
                $partitionMessages = $this->fetchFromLeader(
                    $leaderBroker, 
                    $topic, 
                    $partition, 
                    $offset, 
                    $options['max_messages'] ?? 100
                );
                
                $messages = array_merge($messages, $partitionMessages);
            }
            
            return $messages;
            
        } catch (Exception $e) {
            error_log("Consume messages failed: " . $e->getMessage());
            throw new MessageConsumeException($e->getMessage());
        }
    }
    
    /**
     * 提交消费位置
     */
    public function commitOffset($consumerGroup, $topic, $partition, $offset) {
        try {
            // 更新消费者组的offset
            $this->coordinator->updateConsumerOffset($consumerGroup, $topic, $partition, $offset);
            
            // 持久化到存储
            $this->storage->saveConsumerOffset($consumerGroup, $topic, $partition, $offset);
            
            return true;
            
        } catch (Exception $e) {
            error_log("Commit offset failed: " . $e->getMessage());
            return false;
        }
    }
    
    /**
     * 创建主题
     */
    public function createTopic($topic, $options = []) {
        try {
            $partitionCount = $options['partitions'] ?? 3;
            $replicationFactor = $options['replication_factor'] ?? $this->replicationFactor;
            
            // 1. 在协调服务中注册topic
            $this->coordinator->createTopic($topic, $partitionCount, $replicationFactor);
            
            // 2. 分配分区到Broker
            $partitionAssignments = $this->assignPartitionsToBrokers(
                $topic, 
                $partitionCount, 
                $replicationFactor
            );
            
            // 3. 在各Broker上创建分区
            foreach ($partitionAssignments as $partition => $brokers) {
                foreach ($brokers as $brokerId) {
                    $this->createPartitionOnBroker($brokerId, $topic, $partition);
                }
            }
            
            // 4. 选举分区Leader
            $this->electPartitionLeaders($topic, $partitionAssignments);
            
            $this->topics[$topic] = [
                'partitions' => $partitionCount,
                'replication_factor' => $replicationFactor,
                'created_at' => time()
            ];
            
            return true;
            
        } catch (Exception $e) {
            error_log("Create topic failed: " . $e->getMessage());
            throw new TopicCreationException($e->getMessage());
        }
    }
    
    /**
     * 选择分区
     */
    private function selectPartition($topic, $message, $options) {
        $partitionCount = $this->getTopicPartitionCount($topic);
        
        // 如果指定了分区
        if (isset($options['partition'])) {
            return $options['partition'] % $partitionCount;
        }
        
        // 如果指定了key，使用key的哈希值
        if (isset($options['key'])) {
            return crc32($options['key']) % $partitionCount;
        }
        
        // 轮询分区
        static $roundRobinCounter = [];
        if (!isset($roundRobinCounter[$topic])) {
            $roundRobinCounter[$topic] = 0;
        }
        
        return $roundRobinCounter[$topic]++ % $partitionCount;
    }
    
    /**
     * 构建消息对象
     */
    private function buildMessage($message, $options) {
        return [
            'id' => $this->generateMessageId(),
            'key' => $options['key'] ?? null,
            'value' => $this->serializeMessage($message),
            'headers' => $options['headers'] ?? [],
            'timestamp' => microtime(true),
            'producer_id' => $options['producer_id'] ?? 'unknown',
            'compression' => $options['compression'] ?? 'none'
        ];
    }
    
    /**
     * 写入Leader Broker
     */
    private function writeToLeader($brokerId, $topic, $partition, $message) {
        try {
            $broker = $this->getBroker($brokerId);
            
            // 获取下一个offset
            $offset = $this->getNextOffset($topic, $partition);
            
            // 构建日志记录
            $logRecord = [
                'offset' => $offset,
                'timestamp' => $message['timestamp'],
                'key' => $message['key'],
                'value' => $message['value'],
                'headers' => $message['headers']
            ];
            
            // 写入消息日志
            $broker->appendToLog($topic, $partition, $logRecord);
            
            // 更新分区的高水位标记
            $this->updateHighWatermark($topic, $partition, $offset);
            
            return $offset;
            
        } catch (Exception $e) {
            error_log("Write to leader failed: " . $e->getMessage());
            throw $e;
        }
    }
    
    /**
     * 从Leader拉取消息
     */
    private function fetchFromLeader($brokerId, $topic, $partition, $offset, $maxMessages) {
        try {
            $broker = $this->getBroker($brokerId);
            
            // 从消息日志读取
            $logRecords = $broker->readFromLog($topic, $partition, $offset, $maxMessages);
            
            $messages = [];
            foreach ($logRecords as $record) {
                $messages[] = [
                    'topic' => $topic,
                    'partition' => $partition,
                    'offset' => $record['offset'],
                    'key' => $record['key'],
                    'value' => $this->deserializeMessage($record['value']),
                    'headers' => $record['headers'],
                    'timestamp' => $record['timestamp']
                ];
            }
            
            return $messages;
            
        } catch (Exception $e) {
            error_log("Fetch from leader failed: " . $e->getMessage());
            return [];
        }
    }
    
    /**
     * 等待副本同步
     */
    private function waitForReplication($topic, $partition, $offset) {
        $replicas = $this->getPartitionReplicas($topic, $partition);
        $timeout = 5000; // 5秒超时
        $startTime = microtime(true);
        
        while ((microtime(true) - $startTime) * 1000 < $timeout) {
            $syncedReplicas = 0;
            
            foreach ($replicas as $replicaBrokerId) {
                $replicaOffset = $this->getReplicaOffset($replicaBrokerId, $topic, $partition);
                if ($replicaOffset >= $offset) {
                    $syncedReplicas++;
                }
            }
            
            // 大多数副本已同步
            if ($syncedReplicas >= ceil(count($replicas) / 2)) {
                return true;
            }
            
            usleep(10000); // 10ms
        }
        
        throw new ReplicationTimeoutException("Replication timeout for topic {$topic} partition {$partition}");
    }
}
```

##### 2. 消息存储引擎
```php
<?php
class MessageStorage {
    private $dataDir;
    private $segmentSize;
    private $indexInterval;
    private $compressionType;
    
    public function __construct($config) {
        $this->dataDir = $config['data_dir'];
        $this->segmentSize = $config['segment_size'] ?? 1024 * 1024 * 1024; // 1GB
        $this->indexInterval = $config['index_interval'] ?? 4096;
        $this->compressionType = $config['compression'] ?? 'snappy';
    }
    
    /**
     * 追加消息到日志
     */
    public function appendMessage($topic, $partition, $message) {
        try {
            $logDir = $this->getLogDir($topic, $partition);
            $this->ensureLogDirExists($logDir);
            
            // 获取当前活跃段
            $activeSegment = $this->getActiveSegment($topic, $partition);
            
            // 检查是否需要滚动到新段
            if ($this->shouldRollSegment($activeSegment)) {
                $activeSegment = $this->rollToNewSegment($topic, $partition);
            }
            
            // 序列化消息
            $serializedMessage = $this->serializeMessage($message);
            
            // 压缩消息（如果启用）
            if ($this->compressionType !== 'none') {
                $serializedMessage = $this->compressMessage($serializedMessage);
            }
            
            // 写入日志文件
            $position = $this->writeToSegment($activeSegment, $serializedMessage);
            
            // 更新索引
            $this->updateIndex($activeSegment, $message['offset'], $position);
            
            // 刷盘（根据配置）
            if ($this->shouldFlush()) {
                $this->flushSegment($activeSegment);
            }
            
            return $message['offset'];
            
        } catch (Exception $e) {
            error_log("Append message failed: " . $e->getMessage());
            throw new StorageException($e->getMessage());
        }
    }
    
    /**
     * 读取消息
     */
    public function readMessages($topic, $partition, $startOffset, $maxBytes = null) {
        try {
            $logDir = $this->getLogDir($topic, $partition);
            
            // 查找包含起始offset的段
            $segment = $this->findSegmentForOffset($topic, $partition, $startOffset);
            
            if (!$segment) {
                return [];
            }
            
            // 从索引查找消息位置
            $position = $this->findPositionFromIndex($segment, $startOffset);
            
            // 读取消息数据
            $messages = [];
            $bytesRead = 0;
            $currentOffset = $startOffset;
            
            while (true) {
                // 读取消息长度
                $lengthData = $this->readFromSegment($segment, $position, 4);
                if (strlen($lengthData) < 4) {
                    break;
                }
                
                $messageLength = unpack('N', $lengthData)[1];
                
                // 检查是否超过最大字节数
                if ($maxBytes && $bytesRead + $messageLength > $maxBytes) {
                    break;
                }
                
                // 读取完整消息
                $messageData = $this->readFromSegment($segment, $position + 4, $messageLength);
                
                // 解压缩消息
                if ($this->compressionType !== 'none') {
                    $messageData = $this->decompressMessage($messageData);
                }
                
                // 反序列化消息
                $message = $this->deserializeMessage($messageData);
                $message['offset'] = $currentOffset;
                
                $messages[] = $message;
                
                $position += 4 + $messageLength;
                $bytesRead += 4 + $messageLength;
                $currentOffset++;
                
                // 检查是否到达段末尾
                if ($position >= $this->getSegmentSize($segment)) {
                    // 切换到下一个段
                    $nextSegment = $this->getNextSegment($segment);
                    if (!$nextSegment) {
                        break;
                    }
                    $segment = $nextSegment;
                    $position = 0;
                }
            }
            
            return $messages;
            
        } catch (Exception $e) {
            error_log("Read messages failed: " . $e->getMessage());
            return [];
        }
    }
    
    /**
     * 清理过期日志
     */
    public function cleanupExpiredLogs($retentionMs = null) {
        try {
            $retentionMs = $retentionMs ?? (7 * 24 * 60 * 60 * 1000); // 默认7天
            $cutoffTime = microtime(true) * 1000 - $retentionMs;
            
            $topicDirs = glob($this->dataDir . '/*', GLOB_ONLYDIR);
            
            foreach ($topicDirs as $topicDir) {
                $partitionDirs = glob($topicDir . '/*', GLOB_ONLYDIR);
                
                foreach ($partitionDirs as $partitionDir) {
                    $this->cleanupPartitionLogs($partitionDir, $cutoffTime);
                }
            }
            
        } catch (Exception $e) {
            error_log("Cleanup expired logs failed: " . $e->getMessage());
        }
    }
    
    /**
     * 清理分区日志
     */
    private function cleanupPartitionLogs($partitionDir, $cutoffTime) {
        $segments = $this->getSegments($partitionDir);
        
        foreach ($segments as $segment) {
            $segmentTime = $this->getSegmentTimestamp($segment);
            
            // 删除过期段
            if ($segmentTime < $cutoffTime && !$this->isActiveSegment($segment)) {
                $this->deleteSegment($segment);
                error_log("Deleted expired segment: {$segment}");
            }
        }
    }
    
    /**
     * 压缩日志
     */
    public function compactLogs($topic, $partition) {
        try {
            $logDir = $this->getLogDir($topic, $partition);
            $segments = $this->getSegments($logDir);
            
            // 构建key到最新offset的映射
            $keyToLatestOffset = [];
            
            // 从最新的段开始扫描
            foreach (array_reverse($segments) as $segment) {
                $messages = $this->readSegmentMessages($segment);
                
                foreach (array_reverse($messages) as $message) {
                    $key = $message['key'];
                    if ($key !== null && !isset($keyToLatestOffset[$key])) {
                        $keyToLatestOffset[$key] = $message['offset'];
                    }
                }
            }
            
            // 创建压缩后的段
            $compactedSegment = $this->createCompactedSegment($topic, $partition);
            
            foreach ($segments as $segment) {
                if ($this->isActiveSegment($segment)) {
                    continue; // 跳过活跃段
                }
                
                $messages = $this->readSegmentMessages($segment);
                
                foreach ($messages as $message) {
                    $key = $message['key'];
                    $offset = $message['offset'];
                    
                    // 只保留最新版本的消息
                    if ($key === null || $keyToLatestOffset[$key] === $offset) {
                        $this->appendToCompactedSegment($compactedSegment, $message);
                    }
                }
                
                // 删除原始段
                $this->deleteSegment($segment);
            }
            
            // 重命名压缩段
            $this->finalizeCompactedSegment($compactedSegment);
            
        } catch (Exception $e) {
            error_log("Compact logs failed: " . $e->getMessage());
            throw new CompactionException($e->getMessage());
        }
    }
}
```

##### 3. 消费者组管理
```php
<?php
class ConsumerGroupManager {
    private $coordinator;
    private $storage;
    private $consumerGroups = [];
    private $rebalanceStrategy;
    
    public function __construct($coordinator, $storage) {
        $this->coordinator = $coordinator;
        $this->storage = $storage;
        $this->rebalanceStrategy = new RangeAssignmentStrategy();
    }
    
    /**
     * 注册消费者
     */
    public function registerConsumer($groupId, $consumerId, $topics, $metadata = []) {
        try {
            // 1. 检查消费者组是否存在
            if (!$this->groupExists($groupId)) {
                $this->createConsumerGroup($groupId);
            }
            
            // 2. 添加消费者到组
            $consumer = [
                'id' => $consumerId,
                'topics' => $topics,
                'metadata' => $metadata,
                'last_heartbeat' => time(),
                'session_timeout' => $metadata['session_timeout'] ?? 30000,
                'assignments' => []
            ];
            
            $this->consumerGroups[$groupId]['consumers'][$consumerId] = $consumer;
            
            // 3. 触发重平衡
            $this->triggerRebalance($groupId);
            
            // 4. 在协调服务中注册
            $this->coordinator->registerConsumer($groupId, $consumerId, $consumer);
            
            return $consumerId;
            
        } catch (Exception $e) {
            error_log("Register consumer failed: " . $e->getMessage());
            throw new ConsumerRegistrationException($e->getMessage());
        }
    }
    
    /**
     * 消费者心跳
     */
    public function heartbeat($groupId, $consumerId) {
        try {
            if (!isset($this->consumerGroups[$groupId]['consumers'][$consumerId])) {
                throw new ConsumerNotFoundException("Consumer {$consumerId} not found in group {$groupId}");
            }
            
            // 更新心跳时间
            $this->consumerGroups[$groupId]['consumers'][$consumerId]['last_heartbeat'] = time();
            
            // 检查是否需要重平衡
            if ($this->consumerGroups[$groupId]['rebalance_needed']) {
                return ['rebalance_needed' => true];
            }
            
            return ['rebalance_needed' => false];
            
        } catch (Exception $e) {
            error_log("Heartbeat failed: " . $e->getMessage());
            throw $e;
        }
    }
    
    /**
     * 执行重平衡
     */
    public function rebalance($groupId) {
        try {
            $group = $this->consumerGroups[$groupId];
            
            // 1. 获取所有活跃消费者
            $activeConsumers = $this->getActiveConsumers($groupId);
            
            // 2. 获取所有订阅的主题分区
            $allPartitions = $this->getAllSubscribedPartitions($groupId);
            
            // 3. 执行分区分配
            $assignments = $this->rebalanceStrategy->assign($activeConsumers, $allPartitions);
            
            // 4. 更新消费者分配
            foreach ($assignments as $consumerId => $partitions) {
                $this->consumerGroups[$groupId]['consumers'][$consumerId]['assignments'] = $partitions;
            }
            
            // 5. 通知所有消费者新的分配
            $this->notifyConsumersOfAssignment($groupId, $assignments);
            
            // 6. 标记重平衡完成
            $this->consumerGroups[$groupId]['rebalance_needed'] = false;
            $this->consumerGroups[$groupId]['generation']++;
            
            error_log("Rebalance completed for group {$groupId}, generation {$this->consumerGroups[$groupId]['generation']}");
            
        } catch (Exception $e) {
            error_log("Rebalance failed: " . $e->getMessage());
            throw new RebalanceException($e->getMessage());
        }
    }
    
    /**
     * 提交偏移量
     */
    public function commitOffsets($groupId, $consumerId, $offsets) {
        try {
            // 1. 验证消费者
            if (!isset($this->consumerGroups[$groupId]['consumers'][$consumerId])) {
                throw new ConsumerNotFoundException("Consumer {$consumerId} not found");
            }
            
            // 2. 验证分区分配
            $assignments = $this->consumerGroups[$groupId]['consumers'][$consumerId]['assignments'];
            
            foreach ($offsets as $topicPartition => $offset) {
                list($topic, $partition) = explode('-', $topicPartition);
                
                if (!in_array(['topic' => $topic, 'partition' => $partition], $assignments)) {
                    throw new InvalidOffsetCommitException("Consumer not assigned to partition {$topicPartition}");
                }
            }
            
            // 3. 持久化偏移量
            foreach ($offsets as $topicPartition => $offset) {
                list($topic, $partition) = explode('-', $topicPartition);
                $this->storage->saveConsumerOffset($groupId, $topic, $partition, $offset);
            }
            
            // 4. 更新内存中的偏移量
            $this->consumerGroups[$groupId]['offsets'] = array_merge(
                $this->consumerGroups[$groupId]['offsets'] ?? [],
                $offsets
            );
            
            return true;
            
        } catch (Exception $e) {
            error_log("Commit offsets failed: " . $e->getMessage());
            throw $e;
        }
    }
    
    /**
     * 获取消费者偏移量
     */
    public function getConsumerOffsets($groupId, $topicPartitions) {
        try {
            $offsets = [];
            
            foreach ($topicPartitions as $tp) {
                $topic = $tp['topic'];
                $partition = $tp['partition'];
                
                // 从存储中读取偏移量
                $offset = $this->storage->getConsumerOffset($groupId, $topic, $partition);
                
                // 如果没有存储的偏移量，使用默认策略
                if ($offset === null) {
                    $offset = $this->getDefaultOffset($topic, $partition);
                }
                
                $offsets["{$topic}-{$partition}"] = $offset;
            }
            
            return $offsets;
            
        } catch (Exception $e) {
            error_log("Get consumer offsets failed: " . $e->getMessage());
            return [];
        }
    }
    
    /**
     * 检查消费者健康状态
     */
    public function checkConsumerHealth() {
        $currentTime = time();
        
        foreach ($this->consumerGroups as $groupId => $group) {
            $deadConsumers = [];
            
            foreach ($group['consumers'] as $consumerId => $consumer) {
                $timeSinceHeartbeat = $currentTime - $consumer['last_heartbeat'];
                
                // 检查是否超时
                if ($timeSinceHeartbeat > $consumer['session_timeout'] / 1000) {
                    $deadConsumers[] = $consumerId;
                }
            }
            
            // 移除死亡的消费者
            if (!empty($deadConsumers)) {
                foreach ($deadConsumers as $consumerId) {
                    unset($this->consumerGroups[$groupId]['consumers'][$consumerId]);
                    error_log("Removed dead consumer {$consumerId} from group {$groupId}");
                }
                
                // 触发重平衡
                $this->triggerRebalance($groupId);
            }
        }
    }
}
```

#### 四、性能优化策略
```php
<?php
class MessageQueueOptimizer {
    private $metrics;
    private $config;
    
    public function __construct($config) {
        $this->config = $config;
        $this->metrics = new PerformanceMetrics();
    }
    
    /**
     * 批量处理优化
     */
    public function optimizeBatchProcessing($producer) {
        // 启用批量发送
        $producer->setBatchSize($this->config['batch_size'] ?? 16384); // 16KB
        $producer->setLingerMs($this->config['linger_ms'] ?? 5); // 5ms
        
        // 启用压缩
        $producer->setCompression($this->config['compression'] ?? 'snappy');
        
        // 设置缓冲区大小
        $producer->setBufferMemory($this->config['buffer_memory'] ?? 33554432); // 32MB
    }
    
    /**
     * 分区策略优化
     */
    public function optimizePartitioning($topic, $messagePattern) {
        $currentPartitions = $this->getTopicPartitionCount($topic);
        $throughput = $this->metrics->getThroughput($topic);
        
        // 基于吞吐量调整分区数
        $optimalPartitions = $this->calculateOptimalPartitions($throughput);
        
        if ($optimalPartitions > $currentPartitions) {
            $this->recommendPartitionIncrease($topic, $optimalPartitions);
        }
        
        // 优化分区键策略
        $this->optimizePartitionKey($topic, $messagePattern);
    }
    
    /**
     * 消费者优化
     */
    public function optimizeConsumer($consumer, $consumerGroup) {
        // 调整拉取大小
        $consumer->setFetchMinBytes($this->config['fetch_min_bytes'] ?? 1);
        $consumer->setFetchMaxBytes($this->config['fetch_max_bytes'] ?? 52428800); // 50MB
        $consumer->setFetchMaxWaitMs($this->config['fetch_max_wait_ms'] ?? 500);
        
        // 优化消费者数量
        $lag = $this->metrics->getConsumerLag($consumerGroup);
        if ($lag > $this->config['lag_threshold'] ?? 10000) {
            $this->recommendConsumerScaling($consumerGroup);
        }
    }
    
    /**
     * 存储优化
     */
    public function optimizeStorage() {
        // 日志压缩
        $this->enableLogCompaction();
        
        // 段大小优化
        $this->optimizeSegmentSize();
        
        // 索引间隔优化
        $this->optimizeIndexInterval();
        
        // 刷盘策略优化
        $this->optimizeFlushPolicy();
    }
}
```

**总结**：消息队列系统的核心在于高吞吐量、低延迟和可靠性保证。通过分区机制、副本同步、消费者组管理和完善的存储引擎，可以构建一个高性能、高可用的消息队列服务。关键是要在吞吐量、延迟和一致性之间找到合适的平衡点。
</details>

### 7. 设计一个搜索引擎系统（阿里/腾讯/字节）

<details>
<summary>答案与解析</summary>

#### 一、需求分析
##### 1. 功能需求
- **文档索引**：支持海量文档的快速索引和更新
- **全文搜索**：支持关键词、短语、模糊匹配搜索
- **高级搜索**：支持布尔查询、范围查询、聚合查询
- **相关性排序**：基于TF-IDF、BM25等算法的相关性评分
- **实时搜索**：支持实时索引更新和搜索
- **多语言支持**：支持中文分词、多语言检索
- **搜索建议**：自动补全、拼写纠错、搜索推荐

##### 2. 非功能需求
- **高性能**：毫秒级搜索响应时间
- **高并发**：支持万级QPS搜索请求
- **高可用性**：99.9% 可用性保证
- **水平扩展**：支持集群扩容和分片
- **数据一致性**：最终一致性保证

#### 二、系统架构设计
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   数据采集层    │    │   索引构建层    │    │   搜索服务层    │
│  - 爬虫系统     │───▶│  - 文档解析     │───▶│  - 查询解析     │
│  - 数据接口     │    │  - 分词处理     │    │  - 搜索执行     │
│  - 实时流       │    │  - 索引构建     │    │  - 结果排序     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │                        │
                                ▼                        ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   存储引擎      │    │   索引存储      │    │   缓存层        │
│  - 原始文档     │───▶│  - 倒排索引     │───▶│  - 查询缓存     │
│  - 元数据       │    │  - 正排索引     │    │  - 结果缓存     │
│  - 配置信息     │    │  - 分片管理     │    │  - 热点数据     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │   协调管理      │
                       │  - 集群管理     │
                       │  - 分片路由     │
                       │  - 负载均衡     │
                       └─────────────────┘
```

#### 三、核心组件实现
##### 1. 搜索引擎核心
```php
<?php
class SearchEngine {
    private $indexManager;
    private $queryProcessor;
    private $rankingEngine;
    private $cacheManager;
    private $clusterManager;
    
    public function __construct($config) {
        $this->indexManager = new IndexManager($config['index']);
        $this->queryProcessor = new QueryProcessor($config['query']);
        $this->rankingEngine = new RankingEngine($config['ranking']);
        $this->cacheManager = new CacheManager($config['cache']);
        $this->clusterManager = new ClusterManager($config['cluster']);
    }
    
    /**
     * 索引文档
     */
    public function indexDocument($document, $options = []) {
        try {
            // 1. 文档预处理
            $processedDoc = $this->preprocessDocument($document);
            
            // 2. 文档分析和分词
            $analyzedDoc = $this->analyzeDocument($processedDoc, $options);
            
            // 3. 构建索引项
            $indexEntries = $this->buildIndexEntries($analyzedDoc);
            
            // 4. 选择目标分片
            $shardId = $this->selectShard($document['id'], $options);
            
            // 5. 写入倒排索引
            $this->indexManager->addToInvertedIndex($shardId, $indexEntries);
            
            // 6. 写入正排索引（存储原始文档）
            $this->indexManager->addToForwardIndex($shardId, $document['id'], $processedDoc);
            
            // 7. 更新文档元数据
            $this->updateDocumentMetadata($document['id'], $analyzedDoc, $shardId);
            
            // 8. 刷新索引（根据配置）
            if ($options['refresh'] ?? false) {
                $this->indexManager->refreshShard($shardId);
            }
            
            return [
                'document_id' => $document['id'],
                'shard_id' => $shardId,
                'indexed_at' => microtime(true),
                'terms_count' => count($indexEntries)
            ];
            
        } catch (Exception $e) {
            error_log("Index document failed: " . $e->getMessage());
            throw new IndexingException($e->getMessage());
        }
    }
    
    /**
     * 搜索文档
     */
    public function search($query, $options = []) {
        try {
            $startTime = microtime(true);
            
            // 1. 查询缓存检查
            $cacheKey = $this->generateCacheKey($query, $options);
            $cachedResult = $this->cacheManager->get($cacheKey);
            if ($cachedResult && !($options['bypass_cache'] ?? false)) {
                return $this->addSearchMetadata($cachedResult, $startTime, true);
            }
            
            // 2. 查询解析和预处理
            $parsedQuery = $this->queryProcessor->parse($query, $options);
            
            // 3. 查询计划生成
            $queryPlan = $this->generateQueryPlan($parsedQuery, $options);
            
            // 4. 分片查询执行
            $shardResults = $this->executeShardQueries($queryPlan);
            
            // 5. 结果合并和排序
            $mergedResults = $this->mergeShardResults($shardResults, $options);
            
            // 6. 相关性评分
            $scoredResults = $this->rankingEngine->scoreResults($mergedResults, $parsedQuery);
            
            // 7. 结果分页和格式化
            $finalResults = $this->formatResults($scoredResults, $options);
            
            // 8. 缓存结果
            $this->cacheManager->set($cacheKey, $finalResults, $options['cache_ttl'] ?? 300);
            
            return $this->addSearchMetadata($finalResults, $startTime, false);
            
        } catch (Exception $e) {
            error_log("Search failed: " . $e->getMessage());
            throw new SearchException($e->getMessage());
        }
    }
    
    /**
     * 聚合查询
     */
    public function aggregate($query, $aggregations, $options = []) {
        try {
            // 1. 解析聚合查询
            $parsedAggregations = $this->parseAggregations($aggregations);
            
            // 2. 执行基础查询
            $baseResults = $this->search($query, array_merge($options, ['return_raw' => true]));
            
            // 3. 执行聚合计算
            $aggregationResults = [];
            
            foreach ($parsedAggregations as $aggName => $aggConfig) {
                switch ($aggConfig['type']) {
                    case 'terms':
                        $aggregationResults[$aggName] = $this->termsAggregation(
                            $baseResults['documents'], 
                            $aggConfig
                        );
                        break;
                        
                    case 'range':
                        $aggregationResults[$aggName] = $this->rangeAggregation(
                            $baseResults['documents'], 
                            $aggConfig
                        );
                        break;
                        
                    case 'date_histogram':
                        $aggregationResults[$aggName] = $this->dateHistogramAggregation(
                            $baseResults['documents'], 
                            $aggConfig
                        );
                        break;
                        
                    case 'stats':
                        $aggregationResults[$aggName] = $this->statsAggregation(
                            $baseResults['documents'], 
                            $aggConfig
                        );
                        break;
                }
            }
            
            return [
                'query_results' => $baseResults,
                'aggregations' => $aggregationResults,
                'total_hits' => $baseResults['total_hits']
            ];
            
        } catch (Exception $e) {
            error_log("Aggregation failed: " . $e->getMessage());
            throw new AggregationException($e->getMessage());
        }
    }
    
    /**
     * 文档预处理
     */
    private function preprocessDocument($document) {
        // 1. 字段提取和清理
        $processedDoc = [
            'id' => $document['id'],
            'title' => $this->cleanText($document['title'] ?? ''),
            'content' => $this->cleanText($document['content'] ?? ''),
            'url' => $document['url'] ?? '',
            'timestamp' => $document['timestamp'] ?? time(),
            'metadata' => $document['metadata'] ?? []
        ];
        
        // 2. HTML标签清理
        if (isset($document['html_content'])) {
            $processedDoc['content'] = $this->stripHtmlTags($document['html_content']);
        }
        
        // 3. 字符编码标准化
        foreach (['title', 'content'] as $field) {
            if (isset($processedDoc[$field])) {
                $processedDoc[$field] = mb_convert_encoding(
                    $processedDoc[$field], 
                    'UTF-8', 
                    'auto'
                );
            }
        }
        
        return $processedDoc;
    }
    
    /**
     * 文档分析和分词
     */
    private function analyzeDocument($document, $options) {
        $analyzer = $options['analyzer'] ?? 'standard';
        
        $analyzedDoc = $document;
        
        // 分析标题
        if (!empty($document['title'])) {
            $analyzedDoc['title_terms'] = $this->analyzeText($document['title'], $analyzer);
        }
        
        // 分析内容
        if (!empty($document['content'])) {
            $analyzedDoc['content_terms'] = $this->analyzeText($document['content'], $analyzer);
        }
        
        // 计算词频
        $analyzedDoc['term_frequencies'] = $this->calculateTermFrequencies(
            array_merge(
                $analyzedDoc['title_terms'] ?? [],
                $analyzedDoc['content_terms'] ?? []
            )
        );
        
        return $analyzedDoc;
    }
    
    /**
     * 文本分析
     */
    private function analyzeText($text, $analyzer) {
        switch ($analyzer) {
            case 'standard':
                return $this->standardAnalyzer($text);
            case 'chinese':
                return $this->chineseAnalyzer($text);
            case 'keyword':
                return [$text];
            default:
                return $this->standardAnalyzer($text);
        }
    }
    
    /**
     * 标准分析器
     */
    private function standardAnalyzer($text) {
        // 1. 转小写
        $text = mb_strtolower($text, 'UTF-8');
        
        // 2. 移除标点符号
        $text = preg_replace('/[^\p{L}\p{N}\s]/u', ' ', $text);
        
        // 3. 分词
        $terms = preg_split('/\s+/', $text, -1, PREG_SPLIT_NO_EMPTY);
        
        // 4. 过滤停用词
        $terms = $this->removeStopWords($terms);
        
        // 5. 词干提取
        $terms = $this->stemWords($terms);
        
        return $terms;
    }
    
    /**
     * 中文分析器
     */
    private function chineseAnalyzer($text) {
        // 使用中文分词库（如jieba-php）
        $segmenter = new ChineseSegmenter();
        $terms = $segmenter->segment($text);
        
        // 过滤停用词
        $terms = $this->removeChineseStopWords($terms);
        
        return $terms;
    }
    
    /**
     * 构建索引项
     */
    private function buildIndexEntries($document) {
        $entries = [];
        $docId = $document['id'];
        
        // 处理标题词项（权重更高）
        foreach ($document['title_terms'] ?? [] as $term) {
            $entries[] = [
                'term' => $term,
                'document_id' => $docId,
                'field' => 'title',
                'position' => count($entries),
                'frequency' => $document['term_frequencies'][$term] ?? 1,
                'boost' => 2.0 // 标题权重提升
            ];
        }
        
        // 处理内容词项
        foreach ($document['content_terms'] ?? [] as $position => $term) {
            $entries[] = [
                'term' => $term,
                'document_id' => $docId,
                'field' => 'content',
                'position' => $position,
                'frequency' => $document['term_frequencies'][$term] ?? 1,
                'boost' => 1.0
            ];
        }
        
        return $entries;
    }
    
    /**
     * 执行分片查询
     */
    private function executeShardQueries($queryPlan) {
        $shardResults = [];
        $targetShards = $queryPlan['target_shards'] ?? $this->clusterManager->getAllShards();
        
        // 并行执行分片查询
        $promises = [];
        foreach ($targetShards as $shardId) {
            $promises[$shardId] = $this->executeShardQuery($shardId, $queryPlan);
        }
        
        // 等待所有分片结果
        foreach ($promises as $shardId => $promise) {
            try {
                $shardResults[$shardId] = $promise; // 简化的同步执行
            } catch (Exception $e) {
                error_log("Shard {$shardId} query failed: " . $e->getMessage());
                // 继续处理其他分片
            }
        }
        
        return $shardResults;
    }
    
    /**
     * 执行单个分片查询
     */
    private function executeShardQuery($shardId, $queryPlan) {
        $shard = $this->clusterManager->getShard($shardId);
        
        // 根据查询类型执行不同的查询逻辑
        switch ($queryPlan['query_type']) {
            case 'term':
                return $this->executeTermQuery($shard, $queryPlan);
            case 'phrase':
                return $this->executePhraseQuery($shard, $queryPlan);
            case 'boolean':
                return $this->executeBooleanQuery($shard, $queryPlan);
            case 'wildcard':
                return $this->executeWildcardQuery($shard, $queryPlan);
            case 'range':
                return $this->executeRangeQuery($shard, $queryPlan);
            default:
                throw new UnsupportedQueryException("Unsupported query type: {$queryPlan['query_type']}");
        }
    }
    
    /**
     * 词项查询
     */
    private function executeTermQuery($shard, $queryPlan) {
        $term = $queryPlan['term'];
        $field = $queryPlan['field'] ?? 'content';
        
        // 从倒排索引获取文档列表
        $postingList = $shard->getPostingList($term, $field);
        
        $results = [];
        foreach ($postingList as $posting) {
            $docId = $posting['document_id'];
            $document = $shard->getDocument($docId);
            
            if ($document) {
                $results[] = [
                    'document' => $document,
                    'score' => $this->calculateTermScore($posting, $queryPlan),
                    'highlights' => $this->generateHighlights($document, [$term])
                ];
            }
        }
        
        return $results;
    }
    
    /**
     * 短语查询
     */
    private function executePhraseQuery($shard, $queryPlan) {
        $terms = $queryPlan['terms'];
        $slop = $queryPlan['slop'] ?? 0; // 允许的词间距离
        
        // 获取所有词项的倒排列表
        $postingLists = [];
        foreach ($terms as $term) {
            $postingLists[$term] = $shard->getPostingList($term, $queryPlan['field'] ?? 'content');
        }
        
        // 查找短语匹配
        $results = [];
        $candidateDocs = $this->findCandidateDocuments($postingLists);
        
        foreach ($candidateDocs as $docId) {
            $positions = $this->getTermPositions($postingLists, $docId);
            
            if ($this->isPhraseMatch($positions, $terms, $slop)) {
                $document = $shard->getDocument($docId);
                if ($document) {
                    $results[] = [
                        'document' => $document,
                        'score' => $this->calculatePhraseScore($positions, $queryPlan),
                        'highlights' => $this->generateHighlights($document, $terms)
                    ];
                }
            }
        }
        
        return $results;
    }
    
    /**
     * 布尔查询
     */
    private function executeBooleanQuery($shard, $queryPlan) {
        $mustQueries = $queryPlan['must'] ?? [];
        $shouldQueries = $queryPlan['should'] ?? [];
        $mustNotQueries = $queryPlan['must_not'] ?? [];
        
        $results = [];
        
        // 执行MUST查询（必须匹配）
        $mustResults = [];
        foreach ($mustQueries as $subQuery) {
            $subResults = $this->executeShardQuery($shard->getId(), $subQuery);
            $mustResults[] = $subResults;
        }
        
        // 取交集
        $candidateDocs = $this->intersectResults($mustResults);
        
        // 执行SHOULD查询（可选匹配）
        $shouldResults = [];
        foreach ($shouldQueries as $subQuery) {
            $subResults = $this->executeShardQuery($shard->getId(), $subQuery);
            $shouldResults = array_merge($shouldResults, $subResults);
        }
        
        // 执行MUST_NOT查询（必须不匹配）
        $mustNotDocs = [];
        foreach ($mustNotQueries as $subQuery) {
            $subResults = $this->executeShardQuery($shard->getId(), $subQuery);
            foreach ($subResults as $result) {
                $mustNotDocs[] = $result['document']['id'];
            }
        }
        
        // 合并结果并过滤
        foreach ($candidateDocs as $docId => $score) {
            if (!in_array($docId, $mustNotDocs)) {
                $document = $shard->getDocument($docId);
                if ($document) {
                    $results[] = [
                        'document' => $document,
                        'score' => $score,
                        'highlights' => $this->generateBooleanHighlights($document, $queryPlan)
                    ];
                }
            }
        }
        
        return $results;
    }
}
```

##### 2. 索引管理器
```php
<?php
class IndexManager {
    private $storage;
    private $shards = [];
    private $config;
    
    public function __construct($config) {
        $this->config = $config;
        $this->storage = new IndexStorage($config['storage']);
        $this->initializeShards();
    }
    
    /**
     * 添加到倒排索引
     */
    public function addToInvertedIndex($shardId, $indexEntries) {
        try {
            $shard = $this->getShard($shardId);
            
            foreach ($indexEntries as $entry) {
                $term = $entry['term'];
                $docId = $entry['document_id'];
                $field = $entry['field'];
                
                // 获取或创建词项的倒排列表
                $postingList = $shard->getPostingList($term, $field) ?? [];
                
                // 添加新的文档记录
                $posting = [
                    'document_id' => $docId,
                    'term_frequency' => $entry['frequency'],
                    'positions' => [$entry['position']],
                    'boost' => $entry['boost'],
                    'field_length' => $this->getFieldLength($docId, $field)
                ];
                
                // 如果文档已存在，更新位置信息
                $existingIndex = $this->findDocumentInPostingList($postingList, $docId);
                if ($existingIndex !== false) {
                    $postingList[$existingIndex]['positions'][] = $entry['position'];
                    $postingList[$existingIndex]['term_frequency']++;
                } else {
                    $postingList[] = $posting;
                }
                
                // 按文档ID排序
                usort($postingList, function($a, $b) {
                    return $a['document_id'] <=> $b['document_id'];
                });
                
                // 保存更新后的倒排列表
                $shard->savePostingList($term, $field, $postingList);
                
                // 更新词项统计信息
                $this->updateTermStatistics($shardId, $term, $field, count($postingList));
            }
            
        } catch (Exception $e) {
            error_log("Add to inverted index failed: " . $e->getMessage());
            throw new IndexingException($e->getMessage());
        }
    }
    
    /**
     * 添加到正排索引
     */
    public function addToForwardIndex($shardId, $docId, $document) {
        try {
            $shard = $this->getShard($shardId);
            
            // 存储完整文档
            $shard->saveDocument($docId, $document);
            
            // 更新文档统计信息
            $this->updateDocumentStatistics($shardId, $docId, $document);
            
        } catch (Exception $e) {
            error_log("Add to forward index failed: " . $e->getMessage());
            throw new IndexingException($e->getMessage());
        }
    }
    
    /**
     * 删除文档
     */
    public function deleteDocument($docId) {
        try {
            // 查找文档所在的分片
            $shardId = $this->findDocumentShard($docId);
            if (!$shardId) {
                throw new DocumentNotFoundException("Document {$docId} not found");
            }
            
            $shard = $this->getShard($shardId);
            
            // 获取文档信息
            $document = $shard->getDocument($docId);
            if (!$document) {
                throw new DocumentNotFoundException("Document {$docId} not found in shard {$shardId}");
            }
            
            // 从倒排索引中移除
            $this->removeFromInvertedIndex($shard, $docId, $document);
            
            // 从正排索引中移除
            $shard->deleteDocument($docId);
            
            // 更新统计信息
            $this->updateDeleteStatistics($shardId, $docId);
            
            return true;
            
        } catch (Exception $e) {
            error_log("Delete document failed: " . $e->getMessage());
            throw new DeletionException($e->getMessage());
        }
    }
    
    /**
     * 更新文档
     */
    public function updateDocument($docId, $newDocument, $options = []) {
        try {
            // 先删除旧文档
            $this->deleteDocument($docId);
            
            // 重新索引新文档
            $newDocument['id'] = $docId;
            return $this->indexDocument($newDocument, $options);
            
        } catch (Exception $e) {
            error_log("Update document failed: " . $e->getMessage());
            throw new UpdateException($e->getMessage());
        }
    }
    
    /**
     * 优化索引
     */
    public function optimizeIndex($shardId = null) {
        try {
            $targetShards = $shardId ? [$shardId] : array_keys($this->shards);
            
            foreach ($targetShards as $id) {
                $shard = $this->getShard($id);
                
                // 合并段文件
                $this->mergeSegments($shard);
                
                // 清理删除的文档
                $this->expungeDeletes($shard);
                
                // 重建索引统计信息
                $this->rebuildStatistics($shard);
                
                error_log("Optimized shard {$id}");
            }
            
        } catch (Exception $e) {
            error_log("Optimize index failed: " . $e->getMessage());
            throw new OptimizationException($e->getMessage());
        }
    }
    
    /**
     * 刷新分片
     */
    public function refreshShard($shardId) {
        try {
            $shard = $this->getShard($shardId);
            
            // 提交内存中的更改到磁盘
            $shard->commit();
            
            // 重新打开搜索器
            $shard->reopenSearcher();
            
            // 清理缓存
            $this->clearShardCache($shardId);
            
        } catch (Exception $e) {
            error_log("Refresh shard failed: " . $e->getMessage());
            throw new RefreshException($e->getMessage());
        }
    }
}
```

##### 3. 相关性评分引擎
```php
<?php
class RankingEngine {
    private $config;
    private $statistics;
    
    public function __construct($config) {
        $this->config = $config;
        $this->statistics = new IndexStatistics();
    }
    
    /**
     * 对搜索结果评分
     */
    public function scoreResults($results, $query) {
        $scoringAlgorithm = $this->config['algorithm'] ?? 'bm25';
        
        foreach ($results as &$result) {
            switch ($scoringAlgorithm) {
                case 'tfidf':
                    $result['score'] = $this->calculateTfIdfScore($result, $query);
                    break;
                case 'bm25':
                    $result['score'] = $this->calculateBm25Score($result, $query);
                    break;
                case 'custom':
                    $result['score'] = $this->calculateCustomScore($result, $query);
                    break;
                default:
                    $result['score'] = $this->calculateBm25Score($result, $query);
            }
        }
        
        // 按分数降序排序
        usort($results, function($a, $b) {
            return $b['score'] <=> $a['score'];
        });
        
        return $results;
    }
    
    /**
     * BM25评分算法
     */
    private function calculateBm25Score($result, $query) {
        $k1 = $this->config['bm25_k1'] ?? 1.2;
        $b = $this->config['bm25_b'] ?? 0.75;
        
        $document = $result['document'];
        $docId = $document['id'];
        $score = 0;
        
        foreach ($query['terms'] as $term) {
            // 词频 (TF)
            $tf = $this->getTermFrequency($docId, $term);
            
            // 文档频率 (DF)
            $df = $this->statistics->getDocumentFrequency($term);
            
            // 总文档数
            $totalDocs = $this->statistics->getTotalDocuments();
            
            // 逆文档频率 (IDF)
            $idf = log(($totalDocs - $df + 0.5) / ($df + 0.5));
            
            // 文档长度
            $docLength = $this->getDocumentLength($docId);
            $avgDocLength = $this->statistics->getAverageDocumentLength();
            
            // BM25公式
            $numerator = $tf * ($k1 + 1);
            $denominator = $tf + $k1 * (1 - $b + $b * ($docLength / $avgDocLength));
            
            $termScore = $idf * ($numerator / $denominator);
            
            // 字段权重
            $fieldBoost = $this->getFieldBoost($term, $document);
            $score += $termScore * $fieldBoost;
        }
        
        return $score;
    }
    
    /**
     * TF-IDF评分算法
     */
    private function calculateTfIdfScore($result, $query) {
        $document = $result['document'];
        $docId = $document['id'];
        $score = 0;
        
        foreach ($query['terms'] as $term) {
            // 词频 (TF)
            $tf = $this->getTermFrequency($docId, $term);
            $normalizedTf = $tf > 0 ? (1 + log($tf)) : 0;
            
            // 逆文档频率 (IDF)
            $df = $this->statistics->getDocumentFrequency($term);
            $totalDocs = $this->statistics->getTotalDocuments();
            $idf = log($totalDocs / ($df + 1));
            
            // TF-IDF分数
            $termScore = $normalizedTf * $idf;
            
            // 字段权重
            $fieldBoost = $this->getFieldBoost($term, $document);
            $score += $termScore * $fieldBoost;
        }
        
        // 文档长度归一化
        $docLength = $this->getDocumentLength($docId);
        $normalizedScore = $score / sqrt($docLength);
        
        return $normalizedScore;
    }
    
    /**
     * 自定义评分算法
     */
    private function calculateCustomScore($result, $query) {
        $document = $result['document'];
        $baseScore = $this->calculateBm25Score($result, $query);
        
        // 时间衰减因子
        $timeDecay = $this->calculateTimeDecay($document['timestamp'] ?? time());
        
        // 点击率权重
        $clickBoost = $this->getClickBoost($document['id']);
        
        // 权威性权重
        $authorityBoost = $this->getAuthorityBoost($document);
        
        // 个性化权重
        $personalizationBoost = $this->getPersonalizationBoost($document, $query);
        
        $finalScore = $baseScore * $timeDecay * $clickBoost * $authorityBoost * $personalizationBoost;
        
        return $finalScore;
    }
}
```

#### 四、性能优化策略
```php
<?php
class SearchOptimizer {
    private $cacheManager;
    private $indexManager;
    private $metrics;
    
    public function __construct($cacheManager, $indexManager) {
        $this->cacheManager = $cacheManager;
        $this->indexManager = $indexManager;
        $this->metrics = new SearchMetrics();
    }
    
    /**
     * 查询优化
     */
    public function optimizeQuery($query, $options = []) {
        // 1. 查询重写
        $rewrittenQuery = $this->rewriteQuery($query);
        
        // 2. 词项扩展
        $expandedQuery = $this->expandQuery($rewrittenQuery);
        
        // 3. 停用词过滤
        $filteredQuery = $this->filterStopWords($expandedQuery);
        
        // 4. 查询简化
        $simplifiedQuery = $this->simplifyQuery($filteredQuery);
        
        return $simplifiedQuery;
    }
    
    /**
     * 缓存优化
     */
    public function optimizeCache() {
        // 1. 预热热门查询
        $this->warmupPopularQueries();
        
        // 2. 清理过期缓存
        $this->cleanupExpiredCache();
        
        // 3. 优化缓存策略
        $this->optimizeCacheStrategy();
    }
    
    /**
     * 索引优化
     */
    public function optimizeIndex() {
        // 1. 段合并
        $this->mergeSegments();
        
        // 2. 删除清理
        $this->expungeDeletes();
        
        // 3. 索引压缩
        $this->compressIndex();
    }
}
```

**总结**：搜索引擎系统的核心在于高效的索引结构、精准的相关性评分和快速的查询处理。通过倒排索引、分片机制、多级缓存和智能排序算法，可以构建一个高性能、高可用的搜索服务。关键是要在索引大小、查询速度和结果质量之间找到最佳平衡点。
</details>

### 8. 设计一个直播系统（阿里/腾讯/字节）

<details>
<summary>答案与解析</summary>

#### 一、需求分析
##### 1. 功能需求
- **直播推流**：支持主播端音视频采集、编码、推流
- **直播拉流**：支持观众端音视频拉流、解码、播放
- **多码率支持**：支持多种分辨率和码率的自适应播放
- **实时互动**：支持弹幕、礼物、连麦等实时互动功能
- **录制回放**：支持直播录制和点播回放
- **美颜滤镜**：支持实时美颜、滤镜、特效处理
- **直播管理**：支持直播间创建、管理、权限控制
- **数据统计**：支持观看人数、流量、收益等数据统计

##### 2. 非功能需求
- **低延迟**：端到端延迟 < 3秒
- **高并发**：支持百万级并发观看
- **高可用性**：99.9% 可用性保证
- **全球分发**：支持全球CDN加速
- **弹性扩容**：支持自动扩缩容
- **安全防护**：支持防盗链、DRM保护

#### 二、系统架构设计
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   主播端        │    │   流媒体服务    │    │   观众端        │
│  - 音视频采集   │───▶│  - 推流接收     │───▶│  - 拉流播放     │
│  - 编码压缩     │    │  - 转码处理     │    │  - 解码渲染     │
│  - 推流上传     │    │  - 分发转发     │    │  - 互动功能     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │
                                ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   CDN边缘节点   │    │   核心处理层    │    │   存储层        │
│  - 就近接入     │◀───│  - 负载均衡     │───▶│  - 视频存储     │
│  - 缓存加速     │    │  - 转码集群     │    │  - 配置存储     │
│  - 智能调度     │    │  - 录制服务     │    │  - 日志存储     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │   业务服务层    │
                       │  - 直播管理     │
                       │  - 用户管理     │
                       │  - 互动服务     │
                       │  - 统计分析     │
                       └─────────────────┘
```

#### 三、核心组件实现
##### 1. 直播流媒体服务
```php
<?php
class LiveStreamingService {
    private $rtmpServer;
    private $transcodingService;
    private $cdnManager;
    private $recordingService;
    private $interactionService;
    
    public function __construct($config) {
        $this->rtmpServer = new RTMPServer($config['rtmp']);
        $this->transcodingService = new TranscodingService($config['transcoding']);
        $this->cdnManager = new CDNManager($config['cdn']);
        $this->recordingService = new RecordingService($config['recording']);
        $this->interactionService = new InteractionService($config['interaction']);
    }
    
    /**
     * 开始直播
     */
    public function startLive($streamKey, $options = []) {
        try {
            // 1. 验证推流权限
            $streamInfo = $this->validateStreamKey($streamKey);
            if (!$streamInfo) {
                throw new UnauthorizedException("Invalid stream key: {$streamKey}");
            }
            
            // 2. 创建直播会话
            $liveSession = $this->createLiveSession($streamInfo, $options);
            
            // 3. 启动RTMP接收服务
            $rtmpEndpoint = $this->rtmpServer->createEndpoint($streamKey, [
                'callback' => [$this, 'handleIncomingStream'],
                'session_id' => $liveSession['id']
            ]);
            
            // 4. 准备转码任务
            $transcodingTasks = $this->prepareTranscodingTasks($liveSession, $options);
            
            // 5. 配置CDN分发
            $cdnEndpoints = $this->configureCDNDistribution($liveSession);
            
            // 6. 启动录制服务（如果需要）
            if ($options['enable_recording'] ?? true) {
                $this->recordingService->startRecording($liveSession['id'], $options);
            }
            
            // 7. 初始化互动服务
            $this->interactionService->initializeLiveRoom($liveSession['id']);
            
            // 8. 更新直播状态
            $this->updateLiveStatus($liveSession['id'], 'live');
            
            return [
                'session_id' => $liveSession['id'],
                'rtmp_url' => $rtmpEndpoint['url'],
                'play_urls' => $cdnEndpoints,
                'transcoding_tasks' => $transcodingTasks,
                'started_at' => microtime(true)
            ];
            
        } catch (Exception $e) {
            error_log("Start live failed: " . $e->getMessage());
            throw new LiveStreamException($e->getMessage());
        }
    }
    
    /**
     * 处理推流数据
     */
    public function handleIncomingStream($streamKey, $streamData) {
        try {
            $sessionId = $this->getSessionByStreamKey($streamKey);
            if (!$sessionId) {
                throw new SessionNotFoundException("Session not found for stream key: {$streamKey}");
            }
            
            // 1. 解析音视频流
            $parsedStream = $this->parseStreamData($streamData);
            
            // 2. 质量检测
            $qualityMetrics = $this->analyzeStreamQuality($parsedStream);
            if ($qualityMetrics['score'] < 0.7) {
                $this->notifyQualityIssue($sessionId, $qualityMetrics);
            }
            
            // 3. 分发到转码服务
            $this->distributeToTranscoding($sessionId, $parsedStream);
            
            // 4. 更新流统计信息
            $this->updateStreamMetrics($sessionId, [
                'bitrate' => $parsedStream['bitrate'],
                'fps' => $parsedStream['fps'],
                'resolution' => $parsedStream['resolution'],
                'timestamp' => microtime(true)
            ]);
            
            // 5. 检测异常情况
            $this->detectStreamAnomalies($sessionId, $parsedStream);
            
        } catch (Exception $e) {
            error_log("Handle incoming stream failed: " . $e->getMessage());
            $this->handleStreamError($streamKey, $e);
        }
    }
    
    /**
     * 停止直播
     */
    public function stopLive($sessionId, $reason = 'manual') {
        try {
            $session = $this->getLiveSession($sessionId);
            if (!$session) {
                throw new SessionNotFoundException("Live session not found: {$sessionId}");
            }
            
            // 1. 停止推流接收
            $this->rtmpServer->closeEndpoint($session['stream_key']);
            
            // 2. 停止转码任务
            $this->transcodingService->stopAllTasks($sessionId);
            
            // 3. 停止CDN分发
            $this->cdnManager->stopDistribution($sessionId);
            
            // 4. 完成录制
            if ($session['recording_enabled']) {
                $recordingResult = $this->recordingService->finishRecording($sessionId);
            }
            
            // 5. 关闭互动服务
            $this->interactionService->closeLiveRoom($sessionId);
            
            // 6. 生成直播报告
            $liveReport = $this->generateLiveReport($sessionId);
            
            // 7. 更新直播状态
            $this->updateLiveStatus($sessionId, 'ended', [
                'end_reason' => $reason,
                'ended_at' => microtime(true),
                'report' => $liveReport
            ]);
            
            return [
                'session_id' => $sessionId,
                'duration' => $liveReport['duration'],
                'viewers_count' => $liveReport['peak_viewers'],
                'recording_url' => $recordingResult['url'] ?? null,
                'report' => $liveReport
            ];
            
        } catch (Exception $e) {
            error_log("Stop live failed: " . $e->getMessage());
            throw new LiveStreamException($e->getMessage());
        }
    }
    
    /**
     * 获取播放地址
     */
    public function getPlayUrls($sessionId, $options = []) {
        try {
            $session = $this->getLiveSession($sessionId);
            if (!$session || $session['status'] !== 'live') {
                throw new SessionNotFoundException("Live session not available: {$sessionId}");
            }
            
            // 1. 获取用户地理位置
            $userLocation = $this->getUserLocation($options['client_ip'] ?? '');
            
            // 2. 选择最优CDN节点
            $optimalNodes = $this->cdnManager->selectOptimalNodes($userLocation, $sessionId);
            
            // 3. 生成多码率播放地址
            $playUrls = [];
            foreach ($session['transcoding_outputs'] as $quality => $output) {
                $playUrls[$quality] = [];
                
                foreach ($optimalNodes as $node) {
                    $playUrls[$quality][] = [
                        'protocol' => $output['protocol'],
                        'url' => $this->generatePlayUrl($node, $sessionId, $quality),
                        'bitrate' => $output['bitrate'],
                        'resolution' => $output['resolution'],
                        'node_location' => $node['location']
                    ];
                }
            }
            
            // 4. 添加备用地址
            $backupUrls = $this->generateBackupUrls($sessionId, $userLocation);
            
            return [
                'session_id' => $sessionId,
                'play_urls' => $playUrls,
                'backup_urls' => $backupUrls,
                'recommended_quality' => $this->getRecommendedQuality($options),
                'expires_at' => time() + 3600 // 1小时后过期
            ];
            
        } catch (Exception $e) {
            error_log("Get play URLs failed: " . $e->getMessage());
            throw new PlayUrlException($e->getMessage());
        }
    }
    
    /**
     * 处理观众连接
     */
    public function handleViewerConnection($sessionId, $viewerId, $options = []) {
        try {
            // 1. 验证观看权限
            $this->validateViewerPermission($sessionId, $viewerId);
            
            // 2. 记录观众信息
            $viewerInfo = [
                'viewer_id' => $viewerId,
                'session_id' => $sessionId,
                'join_time' => microtime(true),
                'client_info' => $options['client_info'] ?? [],
                'location' => $this->getUserLocation($options['client_ip'] ?? '')
            ];
            
            $this->recordViewerJoin($viewerInfo);
            
            // 3. 更新在线人数
            $this->updateViewerCount($sessionId, 1);
            
            // 4. 发送欢迎消息
            $this->interactionService->sendWelcomeMessage($sessionId, $viewerId);
            
            // 5. 推送直播状态
            $liveStatus = $this->getLiveStatus($sessionId);
            
            return [
                'viewer_id' => $viewerId,
                'session_id' => $sessionId,
                'live_status' => $liveStatus,
                'current_viewers' => $liveStatus['viewer_count'],
                'interaction_token' => $this->generateInteractionToken($sessionId, $viewerId)
            ];
            
        } catch (Exception $e) {
            error_log("Handle viewer connection failed: " . $e->getMessage());
            throw new ViewerConnectionException($e->getMessage());
        }
    }
    
    /**
     * 处理观众断开
     */
    public function handleViewerDisconnection($sessionId, $viewerId) {
        try {
            // 1. 记录观看时长
            $viewDuration = $this->calculateViewDuration($sessionId, $viewerId);
            
            // 2. 更新在线人数
            $this->updateViewerCount($sessionId, -1);
            
            // 3. 清理互动状态
            $this->interactionService->cleanupViewerState($sessionId, $viewerId);
            
            // 4. 记录观看统计
            $this->recordViewerLeave([
                'viewer_id' => $viewerId,
                'session_id' => $sessionId,
                'view_duration' => $viewDuration,
                'leave_time' => microtime(true)
            ]);
            
        } catch (Exception $e) {
            error_log("Handle viewer disconnection failed: " . $e->getMessage());
        }
    }
}
```

##### 2. 转码服务
```php
<?php
class TranscodingService {
    private $ffmpegCluster;
    private $taskQueue;
    private $outputStorage;
    private $config;
    
    public function __construct($config) {
        $this->config = $config;
        $this->ffmpegCluster = new FFmpegCluster($config['cluster']);
        $this->taskQueue = new TaskQueue($config['queue']);
        $this->outputStorage = new OutputStorage($config['storage']);
    }
    
    /**
     * 创建转码任务
     */
    public function createTranscodingTasks($sessionId, $inputStream, $profiles) {
        try {
            $tasks = [];
            
            foreach ($profiles as $profile) {
                $task = [
                    'id' => $this->generateTaskId(),
                    'session_id' => $sessionId,
                    'input_stream' => $inputStream,
                    'output_profile' => $profile,
                    'status' => 'pending',
                    'created_at' => microtime(true)
                ];
                
                // 生成FFmpeg命令
                $task['ffmpeg_command'] = $this->generateFFmpegCommand($inputStream, $profile);
                
                // 分配转码节点
                $task['assigned_node'] = $this->assignTranscodingNode($profile);
                
                $tasks[] = $task;
                
                // 提交到任务队列
                $this->taskQueue->enqueue($task);
            }
            
            return $tasks;
            
        } catch (Exception $e) {
            error_log("Create transcoding tasks failed: " . $e->getMessage());
            throw new TranscodingException($e->getMessage());
        }
    }
    
    /**
     * 生成FFmpeg命令
     */
    private function generateFFmpegCommand($inputStream, $profile) {
        $command = [
            'ffmpeg',
            '-i', $inputStream['url'],
            '-c:v', $profile['video_codec'] ?? 'libx264',
            '-preset', $profile['preset'] ?? 'fast',
            '-b:v', $profile['video_bitrate'],
            '-maxrate', $profile['max_bitrate'] ?? $profile['video_bitrate'],
            '-bufsize', $profile['buffer_size'] ?? ($profile['video_bitrate'] * 2),
            '-vf', $this->buildVideoFilters($profile),
            '-c:a', $profile['audio_codec'] ?? 'aac',
            '-b:a', $profile['audio_bitrate'] ?? '128k',
            '-ar', $profile['audio_sample_rate'] ?? '44100',
            '-f', $profile['output_format'] ?? 'flv'
        ];
        
        // 添加GOP设置
        if (isset($profile['gop_size'])) {
            $command = array_merge($command, ['-g', $profile['gop_size']]);
        }
        
        // 添加关键帧间隔
        if (isset($profile['keyint'])) {
            $command = array_merge($command, ['-keyint_min', $profile['keyint']]);
        }
        
        // 添加输出地址
        $command[] = $this->generateOutputUrl($profile);
        
        return implode(' ', $command);
    }
    
    /**
     * 构建视频滤镜
     */
    private function buildVideoFilters($profile) {
        $filters = [];
        
        // 分辨率缩放
        if (isset($profile['width']) && isset($profile['height'])) {
            $filters[] = "scale={$profile['width']}:{$profile['height']}";
        }
        
        // 帧率控制
        if (isset($profile['fps'])) {
            $filters[] = "fps={$profile['fps']}";
        }
        
        // 去隔行
        if ($profile['deinterlace'] ?? false) {
            $filters[] = 'yadif';
        }
        
        // 美颜滤镜
        if ($profile['beauty_filter'] ?? false) {
            $filters[] = $this->buildBeautyFilter($profile['beauty_settings'] ?? []);
        }
        
        return implode(',', $filters);
    }
    
    /**
     * 执行转码任务
     */
    public function executeTranscodingTask($task) {
        try {
            $node = $this->ffmpegCluster->getNode($task['assigned_node']);
            
            // 1. 更新任务状态
            $this->updateTaskStatus($task['id'], 'running');
            
            // 2. 执行FFmpeg命令
            $process = $node->executeCommand($task['ffmpeg_command'], [
                'timeout' => $this->config['task_timeout'] ?? 3600,
                'callback' => [$this, 'handleTranscodingProgress']
            ]);
            
            // 3. 监控转码进度
            while ($process->isRunning()) {
                $progress = $this->parseFFmpegProgress($process->getOutput());
                $this->updateTaskProgress($task['id'], $progress);
                
                sleep(1);
            }
            
            // 4. 检查转码结果
            if ($process->getExitCode() === 0) {
                $this->handleTranscodingSuccess($task);
            } else {
                $this->handleTranscodingFailure($task, $process->getErrorOutput());
            }
            
        } catch (Exception $e) {
            error_log("Execute transcoding task failed: " . $e->getMessage());
            $this->handleTranscodingFailure($task, $e->getMessage());
        }
    }
    
    /**
     * 处理转码成功
     */
    private function handleTranscodingSuccess($task) {
        try {
            // 1. 验证输出文件
            $outputInfo = $this->validateOutputFile($task);
            
            // 2. 上传到存储
            $storageUrl = $this->outputStorage->upload($task['output_file'], [
                'session_id' => $task['session_id'],
                'profile' => $task['output_profile']['name']
            ]);
            
            // 3. 更新CDN
            $this->updateCDNEndpoint($task['session_id'], $task['output_profile'], $storageUrl);
            
            // 4. 更新任务状态
            $this->updateTaskStatus($task['id'], 'completed', [
                'output_url' => $storageUrl,
                'output_info' => $outputInfo,
                'completed_at' => microtime(true)
            ]);
            
            // 5. 通知直播服务
            $this->notifyLiveService($task['session_id'], 'transcoding_ready', [
                'profile' => $task['output_profile'],
                'url' => $storageUrl
            ]);
            
        } catch (Exception $e) {
            error_log("Handle transcoding success failed: " . $e->getMessage());
            $this->handleTranscodingFailure($task, $e->getMessage());
        }
    }
}
```

##### 3. 互动服务
```php
<?php
class InteractionService {
    private $websocketServer;
    private $messageQueue;
    private $giftService;
    private $moderationService;
    
    public function __construct($config) {
        $this->websocketServer = new WebSocketServer($config['websocket']);
        $this->messageQueue = new MessageQueue($config['message_queue']);
        $this->giftService = new GiftService($config['gift']);
        $this->moderationService = new ModerationService($config['moderation']);
    }
    
    /**
     * 发送弹幕
     */
    public function sendDanmaku($sessionId, $viewerId, $message, $options = []) {
        try {
            // 1. 内容审核
            $moderationResult = $this->moderationService->checkContent($message);
            if (!$moderationResult['approved']) {
                throw new ContentModerationException($moderationResult['reason']);
            }
            
            // 2. 频率限制
            $this->checkRateLimit($viewerId, 'danmaku');
            
            // 3. 构建弹幕消息
            $danmaku = [
                'id' => $this->generateMessageId(),
                'session_id' => $sessionId,
                'viewer_id' => $viewerId,
                'type' => 'danmaku',
                'content' => $message,
                'timestamp' => microtime(true),
                'style' => $options['style'] ?? []
            ];
            
            // 4. 广播弹幕
            $this->broadcastToLiveRoom($sessionId, $danmaku);
            
            // 5. 记录弹幕
            $this->recordInteraction($danmaku);
            
            return $danmaku;
            
        } catch (Exception $e) {
            error_log("Send danmaku failed: " . $e->getMessage());
            throw new InteractionException($e->getMessage());
        }
    }
    
    /**
     * 发送礼物
     */
    public function sendGift($sessionId, $viewerId, $giftId, $quantity = 1) {
        try {
            // 1. 验证礼物信息
            $giftInfo = $this->giftService->getGiftInfo($giftId);
            if (!$giftInfo) {
                throw new InvalidGiftException("Invalid gift ID: {$giftId}");
            }
            
            // 2. 检查用户余额
            $totalCost = $giftInfo['price'] * $quantity;
            $this->giftService->checkUserBalance($viewerId, $totalCost);
            
            // 3. 扣除费用
            $this->giftService->deductBalance($viewerId, $totalCost);
            
            // 4. 构建礼物消息
            $gift = [
                'id' => $this->generateMessageId(),
                'session_id' => $sessionId,
                'viewer_id' => $viewerId,
                'type' => 'gift',
                'gift_id' => $giftId,
                'gift_name' => $giftInfo['name'],
                'quantity' => $quantity,
                'total_value' => $totalCost,
                'timestamp' => microtime(true),
                'animation' => $giftInfo['animation'] ?? null
            ];
            
            // 5. 广播礼物
            $this->broadcastToLiveRoom($sessionId, $gift);
            
            // 6. 更新主播收益
            $this->updateStreamerRevenue($sessionId, $totalCost);
            
            // 7. 记录礼物
            $this->recordInteraction($gift);
            
            return $gift;
            
        } catch (Exception $e) {
            error_log("Send gift failed: " . $e->getMessage());
            throw new InteractionException($e->getMessage());
        }
    }
}
```

#### 四、性能优化策略
```php
<?php
class LiveStreamOptimizer {
    private $config;
    private $metrics;
    
    public function __construct($config) {
        $this->config = $config;
        $this->metrics = new LiveMetrics();
    }
    
    /**
     * 自适应码率优化
     */
    public function optimizeBitrate($sessionId, $networkConditions) {
        // 1. 分析网络状况
        $bandwidth = $networkConditions['bandwidth'];
        $latency = $networkConditions['latency'];
        $packetLoss = $networkConditions['packet_loss'];
        
        // 2. 计算推荐码率
        $recommendedBitrate = $this->calculateOptimalBitrate($bandwidth, $latency, $packetLoss);
        
        // 3. 调整转码参数
        $this->adjustTranscodingParams($sessionId, $recommendedBitrate);
        
        return $recommendedBitrate;
    }
    
    /**
     * CDN节点优化
     */
    public function optimizeCDNRouting($userLocation, $sessionId) {
        // 1. 获取可用节点
        $availableNodes = $this->getAvailableCDNNodes();
        
        // 2. 计算节点评分
        $nodeScores = [];
        foreach ($availableNodes as $node) {
            $score = $this->calculateNodeScore($node, $userLocation, $sessionId);
            $nodeScores[$node['id']] = $score;
        }
        
        // 3. 选择最优节点
        arsort($nodeScores);
        $optimalNodes = array_slice(array_keys($nodeScores), 0, 3);
        
        return $optimalNodes;
    }
}
```

**总结**：直播系统的核心在于低延迟的音视频传输、高并发的观众支持和丰富的互动功能。通过RTMP推流、多码率转码、CDN分发和WebSocket互动，可以构建一个高性能、高可用的直播平台。关键是要在延迟、画质和成本之间找到最佳平衡点。
</details>

### 9. 设计一个分布式文件存储系统（阿里/腾讯/字节）

<details>
<summary>答案与解析</summary>

#### 一、需求分析
##### 1. 功能需求
- **文件上传**：支持大文件分块上传、断点续传
- **文件下载**：支持多线程下载、范围下载
- **文件管理**：支持文件增删改查、目录管理
- **版本控制**：支持文件版本管理、历史版本恢复
- **权限控制**：支持细粒度的访问权限控制
- **文件同步**：支持多副本同步、一致性保证
- **元数据管理**：支持文件元信息存储和检索
- **数据压缩**：支持文件压缩和解压缩

##### 2. 非功能需求
- **高可用性**：99.99% 可用性保证
- **高扩展性**：支持PB级数据存储
- **高性能**：支持高并发读写操作
- **数据一致性**：保证数据的强一致性
- **容错能力**：支持节点故障自动恢复
- **安全性**：支持数据加密和访问控制

#### 二、系统架构设计
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   客户端        │    │   网关层        │    │   元数据服务    │
│  - 文件操作API  │───▶│  - 负载均衡     │───▶│  - 文件元信息   │
│  - 分块上传     │    │  - 请求路由     │    │  - 目录结构     │
│  - 断点续传     │    │  - 权限验证     │    │  - 版本管理     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │                        │
                                ▼                        ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   存储节点集群  │    │   数据节点      │    │   副本管理器    │
│  - 数据分片     │◀───│  - 文件存储     │───▶│  - 副本策略     │
│  - 负载均衡     │    │  - 数据校验     │    │  - 一致性控制   │
│  - 故障检测     │    │  - 压缩解压     │    │  - 故障恢复     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │   监控告警      │
                       │  - 性能监控     │
                       │  - 健康检查     │
                       │  - 容量预警     │
                       │  - 故障告警     │
                       └─────────────────┘
```

#### 三、核心组件实现
##### 1. 分布式文件存储核心服务
```php
<?php
class DistributedFileSystem {
    private $metadataService;
    private $storageCluster;
    private $replicationManager;
    private $chunkManager;
    private $compressionService;
    
    public function __construct($config) {
        $this->metadataService = new MetadataService($config['metadata']);
        $this->storageCluster = new StorageCluster($config['storage']);
        $this->replicationManager = new ReplicationManager($config['replication']);
        $this->chunkManager = new ChunkManager($config['chunk']);
        $this->compressionService = new CompressionService($config['compression']);
    }
    
    /**
     * 上传文件
     */
    public function uploadFile($filePath, $fileData, $options = []) {
        try {
            // 1. 验证文件信息
            $fileInfo = $this->validateFileInfo($filePath, $fileData, $options);
            
            // 2. 检查存储空间
            $this->checkStorageCapacity($fileInfo['size']);
            
            // 3. 生成文件ID
            $fileId = $this->generateFileId($filePath, $fileInfo);
            
            // 4. 文件分块
            $chunks = $this->chunkManager->splitFile($fileData, [
                'chunk_size' => $options['chunk_size'] ?? (4 * 1024 * 1024), // 4MB
                'file_id' => $fileId
            ]);
            
            // 5. 压缩处理（如果启用）
            if ($options['enable_compression'] ?? true) {
                $chunks = $this->compressChunks($chunks);
            }
            
            // 6. 选择存储节点
            $storageNodes = $this->selectStorageNodes($chunks, $options);
            
            // 7. 并行上传分块
            $uploadResults = $this->uploadChunksParallel($chunks, $storageNodes);
            
            // 8. 创建副本
            $replicationResults = $this->createReplicas($chunks, $uploadResults, $options);
            
            // 9. 保存元数据
            $metadata = [
                'file_id' => $fileId,
                'file_path' => $filePath,
                'file_size' => $fileInfo['size'],
                'chunk_count' => count($chunks),
                'chunks' => $uploadResults,
                'replicas' => $replicationResults,
                'checksum' => $fileInfo['checksum'],
                'compression' => $options['enable_compression'] ?? true,
                'created_at' => microtime(true),
                'version' => 1
            ];
            
            $this->metadataService->saveFileMetadata($fileId, $metadata);
            
            // 10. 更新目录结构
            $this->updateDirectoryStructure($filePath, $fileId);
            
            return [
                'file_id' => $fileId,
                'file_path' => $filePath,
                'file_size' => $fileInfo['size'],
                'chunk_count' => count($chunks),
                'upload_time' => microtime(true) - $fileInfo['start_time'],
                'checksum' => $fileInfo['checksum']
            ];
            
        } catch (Exception $e) {
            error_log("Upload file failed: " . $e->getMessage());
            $this->cleanupFailedUpload($fileId ?? null, $chunks ?? []);
            throw new FileUploadException($e->getMessage());
        }
    }
    
    /**
     * 下载文件
     */
    public function downloadFile($filePath, $options = []) {
        try {
            // 1. 获取文件元数据
            $metadata = $this->metadataService->getFileMetadata($filePath);
            if (!$metadata) {
                throw new FileNotFoundException("File not found: {$filePath}");
            }
            
            // 2. 验证访问权限
            $this->validateAccessPermission($filePath, 'read', $options);
            
            // 3. 检查文件完整性
            $this->verifyFileIntegrity($metadata);
            
            // 4. 选择最优存储节点
            $optimalNodes = $this->selectOptimalNodesForRead($metadata, $options);
            
            // 5. 并行下载分块
            $chunks = $this->downloadChunksParallel($metadata['chunks'], $optimalNodes, $options);
            
            // 6. 验证分块完整性
            $this->verifyChunksIntegrity($chunks, $metadata);
            
            // 7. 解压缩处理
            if ($metadata['compression']) {
                $chunks = $this->decompressChunks($chunks);
            }
            
            // 8. 合并分块
            $fileData = $this->chunkManager->mergeChunks($chunks, $metadata);
            
            // 9. 验证文件完整性
            $actualChecksum = hash('sha256', $fileData);
            if ($actualChecksum !== $metadata['checksum']) {
                throw new FileIntegrityException("File checksum mismatch: {$filePath}");
            }
            
            // 10. 更新访问统计
            $this->updateAccessStatistics($metadata['file_id'], 'download');
            
            return [
                'file_data' => $fileData,
                'file_size' => $metadata['file_size'],
                'file_path' => $filePath,
                'version' => $metadata['version'],
                'download_time' => microtime(true)
            ];
            
        } catch (Exception $e) {
            error_log("Download file failed: " . $e->getMessage());
            throw new FileDownloadException($e->getMessage());
        }
    }
    
    /**
     * 删除文件
     */
    public function deleteFile($filePath, $options = []) {
        try {
            // 1. 获取文件元数据
            $metadata = $this->metadataService->getFileMetadata($filePath);
            if (!$metadata) {
                throw new FileNotFoundException("File not found: {$filePath}");
            }
            
            // 2. 验证删除权限
            $this->validateAccessPermission($filePath, 'delete', $options);
            
            // 3. 软删除处理
            if ($options['soft_delete'] ?? true) {
                return $this->softDeleteFile($metadata, $options);
            }
            
            // 4. 删除所有分块和副本
            $deletionResults = $this->deleteChunksAndReplicas($metadata);
            
            // 5. 删除元数据
            $this->metadataService->deleteFileMetadata($metadata['file_id']);
            
            // 6. 更新目录结构
            $this->updateDirectoryStructure($filePath, null, 'delete');
            
            // 7. 释放存储空间
            $this->releaseStorageSpace($metadata['file_size']);
            
            return [
                'file_id' => $metadata['file_id'],
                'file_path' => $filePath,
                'deleted_size' => $metadata['file_size'],
                'deletion_results' => $deletionResults,
                'deleted_at' => microtime(true)
            ];
            
        } catch (Exception $e) {
            error_log("Delete file failed: " . $e->getMessage());
            throw new FileDeletionException($e->getMessage());
        }
    }
    
    /**
     * 列出目录内容
     */
    public function listDirectory($directoryPath, $options = []) {
        try {
            // 1. 验证目录路径
            $this->validateDirectoryPath($directoryPath);
            
            // 2. 验证访问权限
            $this->validateAccessPermission($directoryPath, 'list', $options);
            
            // 3. 获取目录内容
            $directoryContent = $this->metadataService->getDirectoryContent($directoryPath, $options);
            
            // 4. 过滤和排序
            $filteredContent = $this->filterAndSortContent($directoryContent, $options);
            
            // 5. 添加统计信息
            $contentWithStats = $this->addContentStatistics($filteredContent);
            
            return [
                'directory_path' => $directoryPath,
                'total_items' => count($contentWithStats),
                'items' => $contentWithStats,
                'pagination' => $this->buildPagination($options),
                'listed_at' => microtime(true)
            ];
            
        } catch (Exception $e) {
            error_log("List directory failed: " . $e->getMessage());
            throw new DirectoryListException($e->getMessage());
        }
    }
    
    /**
     * 创建目录
     */
    public function createDirectory($directoryPath, $options = []) {
        try {
            // 1. 验证目录路径
            $this->validateDirectoryPath($directoryPath);
            
            // 2. 检查目录是否已存在
            if ($this->metadataService->directoryExists($directoryPath)) {
                if (!($options['ignore_existing'] ?? false)) {
                    throw new DirectoryExistsException("Directory already exists: {$directoryPath}");
                }
                return ['directory_path' => $directoryPath, 'status' => 'already_exists'];
            }
            
            // 3. 验证创建权限
            $this->validateAccessPermission(dirname($directoryPath), 'create', $options);
            
            // 4. 递归创建父目录
            if ($options['recursive'] ?? true) {
                $this->createParentDirectories($directoryPath, $options);
            }
            
            // 5. 创建目录元数据
            $directoryMetadata = [
                'directory_path' => $directoryPath,
                'created_at' => microtime(true),
                'permissions' => $options['permissions'] ?? 0755,
                'owner' => $options['owner'] ?? 'system'
            ];
            
            $this->metadataService->createDirectory($directoryPath, $directoryMetadata);
            
            return [
                'directory_path' => $directoryPath,
                'status' => 'created',
                'created_at' => $directoryMetadata['created_at']
            ];
            
        } catch (Exception $e) {
            error_log("Create directory failed: " . $e->getMessage());
            throw new DirectoryCreationException($e->getMessage());
        }
    }
}
```

##### 2. 元数据服务
```php
<?php
class MetadataService {
    private $database;
    private $cache;
    private $config;
    
    public function __construct($config) {
        $this->config = $config;
        $this->database = new MetadataDatabase($config['database']);
        $this->cache = new MetadataCache($config['cache']);
    }
    
    /**
     * 保存文件元数据
     */
    public function saveFileMetadata($fileId, $metadata) {
        try {
            // 1. 验证元数据格式
            $this->validateMetadataFormat($metadata);
            
            // 2. 保存到数据库
            $this->database->insertFileMetadata($fileId, $metadata);
            
            // 3. 更新缓存
            $this->cache->setFileMetadata($fileId, $metadata);
            
            // 4. 建立索引
            $this->buildMetadataIndexes($fileId, $metadata);
            
            return true;
            
        } catch (Exception $e) {
            error_log("Save file metadata failed: " . $e->getMessage());
            throw new MetadataException($e->getMessage());
        }
    }
    
    /**
     * 获取文件元数据
     */
    public function getFileMetadata($filePath) {
        try {
            // 1. 从缓存获取
            $fileId = $this->getFileIdByPath($filePath);
            if ($fileId) {
                $metadata = $this->cache->getFileMetadata($fileId);
                if ($metadata) {
                    return $metadata;
                }
            }
            
            // 2. 从数据库获取
            $metadata = $this->database->getFileMetadataByPath($filePath);
            if ($metadata) {
                // 3. 更新缓存
                $this->cache->setFileMetadata($metadata['file_id'], $metadata);
                return $metadata;
            }
            
            return null;
            
        } catch (Exception $e) {
            error_log("Get file metadata failed: " . $e->getMessage());
            throw new MetadataException($e->getMessage());
        }
    }
    
    /**
     * 更新文件元数据
     */
    public function updateFileMetadata($fileId, $updates) {
        try {
            // 1. 获取当前元数据
            $currentMetadata = $this->getFileMetadataById($fileId);
            if (!$currentMetadata) {
                throw new FileNotFoundException("File metadata not found: {$fileId}");
            }
            
            // 2. 合并更新
            $updatedMetadata = array_merge($currentMetadata, $updates);
            $updatedMetadata['updated_at'] = microtime(true);
            $updatedMetadata['version'] = ($currentMetadata['version'] ?? 1) + 1;
            
            // 3. 保存版本历史
            $this->saveMetadataVersion($fileId, $currentMetadata);
            
            // 4. 更新数据库
            $this->database->updateFileMetadata($fileId, $updatedMetadata);
            
            // 5. 更新缓存
            $this->cache->setFileMetadata($fileId, $updatedMetadata);
            
            // 6. 更新索引
            $this->updateMetadataIndexes($fileId, $updatedMetadata);
            
            return $updatedMetadata;
            
        } catch (Exception $e) {
            error_log("Update file metadata failed: " . $e->getMessage());
            throw new MetadataException($e->getMessage());
        }
    }
}
```

##### 3. 副本管理器
```php
<?php
class ReplicationManager {
    private $storageCluster;
    private $consistencyChecker;
    private $config;
    
    public function __construct($config) {
        $this->config = $config;
        $this->storageCluster = new StorageCluster($config['storage']);
        $this->consistencyChecker = new ConsistencyChecker($config['consistency']);
    }
    
    /**
     * 创建副本
     */
    public function createReplicas($chunks, $primaryNodes, $options = []) {
        try {
            $replicationFactor = $options['replication_factor'] ?? $this->config['default_replication_factor'];
            $replicas = [];
            
            foreach ($chunks as $chunkId => $chunkData) {
                $primaryNode = $primaryNodes[$chunkId];
                
                // 1. 选择副本节点
                $replicaNodes = $this->selectReplicaNodes($primaryNode, $replicationFactor - 1);
                
                // 2. 创建副本
                $chunkReplicas = [];
                foreach ($replicaNodes as $replicaNode) {
                    $replicaResult = $this->createChunkReplica($chunkId, $chunkData, $replicaNode);
                    $chunkReplicas[] = $replicaResult;
                }
                
                $replicas[$chunkId] = [
                    'primary_node' => $primaryNode,
                    'replica_nodes' => $chunkReplicas,
                    'replication_factor' => $replicationFactor,
                    'created_at' => microtime(true)
                ];
            }
            
            return $replicas;
            
        } catch (Exception $e) {
            error_log("Create replicas failed: " . $e->getMessage());
            throw new ReplicationException($e->getMessage());
        }
    }
    
    /**
     * 检查副本一致性
     */
    public function checkReplicaConsistency($fileId) {
        try {
            $metadata = $this->getFileMetadata($fileId);
            $inconsistencies = [];
            
            foreach ($metadata['chunks'] as $chunkId => $chunkInfo) {
                $replicas = $metadata['replicas'][$chunkId];
                
                // 1. 获取所有副本的校验和
                $checksums = $this->getReplicaChecksums($chunkId, $replicas);
                
                // 2. 检查一致性
                $uniqueChecksums = array_unique($checksums);
                if (count($uniqueChecksums) > 1) {
                    $inconsistencies[$chunkId] = [
                        'checksums' => $checksums,
                        'replicas' => $replicas,
                        'detected_at' => microtime(true)
                    ];
                }
            }
            
            return [
                'file_id' => $fileId,
                'consistent' => empty($inconsistencies),
                'inconsistencies' => $inconsistencies,
                'checked_at' => microtime(true)
            ];
            
        } catch (Exception $e) {
            error_log("Check replica consistency failed: " . $e->getMessage());
            throw new ConsistencyException($e->getMessage());
        }
    }
    
    /**
     * 修复不一致的副本
     */
    public function repairInconsistentReplicas($fileId, $inconsistencies) {
        try {
            $repairResults = [];
            
            foreach ($inconsistencies as $chunkId => $inconsistency) {
                // 1. 确定正确的版本
                $correctVersion = $this->determineCorrectVersion($chunkId, $inconsistency);
                
                // 2. 修复不一致的副本
                $repairResult = $this->repairChunkReplicas($chunkId, $correctVersion, $inconsistency['replicas']);
                
                $repairResults[$chunkId] = $repairResult;
            }
            
            return [
                'file_id' => $fileId,
                'repair_results' => $repairResults,
                'repaired_at' => microtime(true)
            ];
            
        } catch (Exception $e) {
            error_log("Repair inconsistent replicas failed: " . $e->getMessage());
            throw new RepairException($e->getMessage());
        }
    }
}
```

#### 四、性能优化策略
```php
<?php
class FileSystemOptimizer {
    private $config;
    private $metrics;
    
    public function __construct($config) {
        $this->config = $config;
        $this->metrics = new FileSystemMetrics();
    }
    
    /**
     * 存储负载均衡
     */
    public function optimizeStorageLoad() {
        // 1. 获取节点负载信息
        $nodeLoads = $this->getNodeLoadMetrics();
        
        // 2. 识别热点节点
        $hotNodes = $this->identifyHotNodes($nodeLoads);
        
        // 3. 数据迁移策略
        $migrationPlan = $this->createMigrationPlan($hotNodes);
        
        // 4. 执行数据迁移
        $this->executeMigration($migrationPlan);
        
        return $migrationPlan;
    }
    
    /**
     * 缓存优化
     */
    public function optimizeCache() {
        // 1. 分析访问模式
        $accessPatterns = $this->analyzeAccessPatterns();
        
        // 2. 预热热点数据
        $this->preheatHotData($accessPatterns['hot_files']);
        
        // 3. 清理冷数据
        $this->cleanupColdData($accessPatterns['cold_files']);
        
        return $accessPatterns;
    }
}
```

**总结**：分布式文件存储系统的核心在于数据的可靠性、一致性和高可用性。通过文件分块、多副本机制、元数据管理和智能负载均衡，可以构建一个高性能、高可靠的分布式存储平台。关键是要在存储成本、访问性能和数据安全之间找到最佳平衡点。
</details>

### 10. 设计一个电商系统（阿里/腾讯/字节）

<details>
<summary>答案与解析</summary>

#### 一、需求分析
##### 1. 功能需求
- **用户管理**：用户注册、登录、个人信息管理、收货地址管理
- **商品管理**：商品展示、分类管理、库存管理、价格管理
- **购物车**：商品加入购物车、购物车管理、批量操作
- **订单系统**：下单流程、订单管理、订单状态跟踪
- **支付系统**：多种支付方式、支付安全、退款处理
- **物流系统**：物流跟踪、配送管理、签收确认
- **营销系统**：优惠券、促销活动、会员积分
- **客服系统**：在线客服、售后服务、评价系统

##### 2. 非功能需求
- **高并发**：支持双11等大促期间的高并发访问
- **高可用**：99.99% 系统可用性保证
- **数据一致性**：订单、库存、支付数据强一致性
- **安全性**：用户数据安全、支付安全、防刷防攻击
- **可扩展性**：支持业务快速增长和功能扩展
- **性能优化**：页面加载速度、搜索响应时间优化

#### 二、系统架构设计
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   前端应用      │    │   API网关       │    │   用户服务      │
│  - 商城首页     │───▶│  - 路由转发     │───▶│  - 用户管理     │
│  - 商品详情     │    │  - 负载均衡     │    │  - 权限认证     │
│  - 购物车       │    │  - 限流熔断     │    │  - 会员系统     │
│  - 订单中心     │    │  - 安全防护     │    │  - 积分系统     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │                        │
                                ▼                        ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   商品服务      │    │   订单服务      │    │   支付服务      │
│  - 商品管理     │◀───│  - 订单创建     │───▶│  - 支付处理     │
│  - 库存管理     │    │  - 订单状态     │    │  - 退款处理     │
│  - 分类管理     │    │  - 订单查询     │    │  - 对账系统     │
│  - 搜索服务     │    │  - 购物车       │    │  - 风控系统     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │                        │
                                ▼                        ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   物流服务      │    │   营销服务      │    │   数据存储      │
│  - 物流跟踪     │    │  - 优惠券       │    │  - MySQL集群    │
│  - 配送管理     │    │  - 促销活动     │    │  - Redis缓存    │
│  - 库房管理     │    │  - 推荐系统     │    │  - Elasticsearch│
│  - 运费计算     │    │  - 消息推送     │    │  - 消息队列     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

#### 三、基于Webman框架的核心实现
##### 1. 电商系统核心服务
```php
<?php
// config/app.php - Webman应用配置
return [
    'debug' => false,
    'default_timezone' => 'Asia/Shanghai',
    'request_class' => \support\Request::class,
    'public_path' => base_path() . DIRECTORY_SEPARATOR . 'public',
    'runtime_path' => base_path(false) . DIRECTORY_SEPARATOR . 'runtime',
    'controller_suffix' => 'Controller',
    'controller_reuse' => false,
];

// app/controller/ProductController.php - 商品控制器
<?php
namespace app\controller;

use support\Request;
use support\Response;
use app\service\ProductService;
use app\service\CacheService;
use app\middleware\AuthMiddleware;

class ProductController
{
    protected $productService;
    protected $cacheService;
    
    public function __construct()
    {
        $this->productService = new ProductService();
        $this->cacheService = new CacheService();
    }
    
    /**
     * 获取商品列表
     */
    public function list(Request $request): Response
    {
        try {
            $params = [
                'category_id' => $request->get('category_id', 0),
                'keyword' => $request->get('keyword', ''),
                'page' => max(1, (int)$request->get('page', 1)),
                'limit' => min(100, max(10, (int)$request->get('limit', 20))),
                'sort' => $request->get('sort', 'created_at'),
                'order' => $request->get('order', 'desc'),
                'price_min' => $request->get('price_min', 0),
                'price_max' => $request->get('price_max', 0),
                'brand_id' => $request->get('brand_id', 0)
            ];
            
            // 1. 构建缓存键
            $cacheKey = 'product_list:' . md5(json_encode($params));
            
            // 2. 尝试从缓存获取
            $result = $this->cacheService->get($cacheKey);
            if ($result) {
                return json(['code' => 0, 'data' => $result, 'msg' => 'success']);
            }
            
            // 3. 从数据库获取
            $result = $this->productService->getProductList($params);
            
            // 4. 设置缓存
            $this->cacheService->set($cacheKey, $result, 300); // 5分钟缓存
            
            return json(['code' => 0, 'data' => $result, 'msg' => 'success']);
            
        } catch (\Exception $e) {
            \support\Log::error('Get product list failed: ' . $e->getMessage());
            return json(['code' => -1, 'msg' => '获取商品列表失败']);
        }
    }
    
    /**
     * 获取商品详情
     */
    public function detail(Request $request): Response
    {
        try {
            $productId = (int)$request->get('id');
            if (!$productId) {
                return json(['code' => -1, 'msg' => '商品ID不能为空']);
            }
            
            // 1. 从缓存获取商品详情
            $cacheKey = "product_detail:{$productId}";
            $product = $this->cacheService->get($cacheKey);
            
            if (!$product) {
                // 2. 从数据库获取
                $product = $this->productService->getProductDetail($productId);
                if (!$product) {
                    return json(['code' => -1, 'msg' => '商品不存在']);
                }
                
                // 3. 设置缓存
                $this->cacheService->set($cacheKey, $product, 600); // 10分钟缓存
            }
            
            // 4. 增加浏览量
            $this->productService->incrementViewCount($productId);
            
            // 5. 获取相关推荐
            $recommendations = $this->productService->getRecommendations($productId, 10);
            
            return json([
                'code' => 0,
                'data' => [
                    'product' => $product,
                    'recommendations' => $recommendations
                ],
                'msg' => 'success'
            ]);
            
        } catch (\Exception $e) {
            \support\Log::error('Get product detail failed: ' . $e->getMessage());
            return json(['code' => -1, 'msg' => '获取商品详情失败']);
        }
    }
    
    /**
     * 商品搜索
     */
    public function search(Request $request): Response
    {
        try {
            $keyword = trim($request->get('keyword', ''));
            if (empty($keyword)) {
                return json(['code' => -1, 'msg' => '搜索关键词不能为空']);
            }
            
            $params = [
                'keyword' => $keyword,
                'page' => max(1, (int)$request->get('page', 1)),
                'limit' => min(50, max(10, (int)$request->get('limit', 20))),
                'category_id' => $request->get('category_id', 0),
                'price_min' => $request->get('price_min', 0),
                'price_max' => $request->get('price_max', 0)
            ];
            
            // 1. 使用Elasticsearch进行搜索
            $result = $this->productService->searchProducts($params);
            
            // 2. 记录搜索日志
            $this->productService->logSearch($keyword, $result['total']);
            
            return json(['code' => 0, 'data' => $result, 'msg' => 'success']);
            
        } catch (\Exception $e) {
            \support\Log::error('Product search failed: ' . $e->getMessage());
            return json(['code' => -1, 'msg' => '商品搜索失败']);
        }
    }
}

// app/controller/OrderController.php - 订单控制器
<?php
namespace app\controller;

use support\Request;
use support\Response;
use app\service\OrderService;
use app\service\ProductService;
use app\service\PaymentService;
use app\middleware\AuthMiddleware;

class OrderController
{
    protected $orderService;
    protected $productService;
    protected $paymentService;
    
    public function __construct()
    {
        $this->orderService = new OrderService();
        $this->productService = new ProductService();
        $this->paymentService = new PaymentService();
    }
    
    /**
     * 创建订单
     */
    public function create(Request $request): Response
    {
        try {
            // 1. 验证用户登录
            $userId = AuthMiddleware::getUserId($request);
            if (!$userId) {
                return json(['code' => -1, 'msg' => '请先登录']);
            }
            
            // 2. 获取订单数据
            $orderData = [
                'user_id' => $userId,
                'items' => $request->post('items', []),
                'address_id' => $request->post('address_id'),
                'coupon_id' => $request->post('coupon_id', 0),
                'remark' => $request->post('remark', ''),
                'payment_method' => $request->post('payment_method', 'alipay')
            ];
            
            // 3. 验证订单数据
            $validation = $this->orderService->validateOrderData($orderData);
            if (!$validation['valid']) {
                return json(['code' => -1, 'msg' => $validation['message']]);
            }
            
            // 4. 检查库存
            $stockCheck = $this->productService->checkStock($orderData['items']);
            if (!$stockCheck['available']) {
                return json(['code' => -1, 'msg' => $stockCheck['message']]);
            }
            
            // 5. 计算订单金额
            $orderAmount = $this->orderService->calculateOrderAmount($orderData);
            
            // 6. 创建订单（使用事务）
            \support\Db::beginTransaction();
            try {
                // 6.1 创建订单记录
                $order = $this->orderService->createOrder($orderData, $orderAmount);
                
                // 6.2 扣减库存
                $this->productService->deductStock($orderData['items']);
                
                // 6.3 使用优惠券
                if ($orderData['coupon_id']) {
                    $this->orderService->useCoupon($orderData['coupon_id'], $userId, $order['id']);
                }
                
                \support\Db::commit();
                
                // 7. 发送订单创建消息
                $this->orderService->sendOrderCreatedMessage($order);
                
                return json([
                    'code' => 0,
                    'data' => [
                        'order_id' => $order['id'],
                        'order_no' => $order['order_no'],
                        'total_amount' => $order['total_amount'],
                        'payment_url' => $this->paymentService->generatePaymentUrl($order)
                    ],
                    'msg' => '订单创建成功'
                ]);
                
            } catch (\Exception $e) {
                \support\Db::rollback();
                throw $e;
            }
            
        } catch (\Exception $e) {
            \support\Log::error('Create order failed: ' . $e->getMessage());
            return json(['code' => -1, 'msg' => '订单创建失败']);
        }
    }
    
    /**
     * 获取订单列表
     */
    public function list(Request $request): Response
    {
        try {
            $userId = AuthMiddleware::getUserId($request);
            if (!$userId) {
                return json(['code' => -1, 'msg' => '请先登录']);
            }
            
            $params = [
                'user_id' => $userId,
                'status' => $request->get('status', ''),
                'page' => max(1, (int)$request->get('page', 1)),
                'limit' => min(50, max(10, (int)$request->get('limit', 20)))
            ];
            
            $result = $this->orderService->getOrderList($params);
            
            return json(['code' => 0, 'data' => $result, 'msg' => 'success']);
            
        } catch (\Exception $e) {
            \support\Log::error('Get order list failed: ' . $e->getMessage());
            return json(['code' => -1, 'msg' => '获取订单列表失败']);
        }
    }
    
    /**
     * 取消订单
     */
    public function cancel(Request $request): Response
    {
        try {
            $userId = AuthMiddleware::getUserId($request);
            $orderId = (int)$request->post('order_id');
            
            if (!$orderId) {
                return json(['code' => -1, 'msg' => '订单ID不能为空']);
            }
            
            // 1. 验证订单归属
            $order = $this->orderService->getOrderById($orderId);
            if (!$order || $order['user_id'] != $userId) {
                return json(['code' => -1, 'msg' => '订单不存在']);
            }
            
            // 2. 检查订单状态
            if (!$this->orderService->canCancel($order)) {
                return json(['code' => -1, 'msg' => '订单当前状态不允许取消']);
            }
            
            // 3. 取消订单（使用事务）
            \support\Db::beginTransaction();
            try {
                // 3.1 更新订单状态
                $this->orderService->cancelOrder($orderId, $request->post('reason', ''));
                
                // 3.2 恢复库存
                $this->productService->restoreStock($order['items']);
                
                // 3.3 退还优惠券
                if ($order['coupon_id']) {
                    $this->orderService->restoreCoupon($order['coupon_id'], $userId);
                }
                
                \support\Db::commit();
                
                // 4. 发送订单取消消息
                $this->orderService->sendOrderCancelledMessage($order);
                
                return json(['code' => 0, 'msg' => '订单取消成功']);
                
            } catch (\Exception $e) {
                \support\Db::rollback();
                throw $e;
            }
            
        } catch (\Exception $e) {
            \support\Log::error('Cancel order failed: ' . $e->getMessage());
            return json(['code' => -1, 'msg' => '订单取消失败']);
        }
    }
}
```

##### 2. 购物车服务
```php
<?php
// app/controller/CartController.php - 购物车控制器
namespace app\controller;

use support\Request;
use support\Response;
use app\service\CartService;
use app\service\ProductService;
use app\middleware\AuthMiddleware;

class CartController
{
    protected $cartService;
    protected $productService;
    
    public function __construct()
    {
        $this->cartService = new CartService();
        $this->productService = new ProductService();
    }
    
    /**
     * 添加商品到购物车
     */
    public function add(Request $request): Response
    {
        try {
            $userId = AuthMiddleware::getUserId($request);
            if (!$userId) {
                return json(['code' => -1, 'msg' => '请先登录']);
            }
            
            $productId = (int)$request->post('product_id');
            $quantity = max(1, (int)$request->post('quantity', 1));
            $skuId = (int)$request->post('sku_id', 0);
            
            if (!$productId) {
                return json(['code' => -1, 'msg' => '商品ID不能为空']);
            }
            
            // 1. 验证商品是否存在且可购买
            $product = $this->productService->getProductById($productId);
            if (!$product || $product['status'] != 1) {
                return json(['code' => -1, 'msg' => '商品不存在或已下架']);
            }
            
            // 2. 检查库存
            $stock = $this->productService->getStock($productId, $skuId);
            if ($stock < $quantity) {
                return json(['code' => -1, 'msg' => '库存不足']);
            }
            
            // 3. 添加到购物车
            $result = $this->cartService->addToCart($userId, $productId, $quantity, $skuId);
            
            return json([
                'code' => 0,
                'data' => $result,
                'msg' => '添加成功'
            ]);
            
        } catch (\Exception $e) {
            \support\Log::error('Add to cart failed: ' . $e->getMessage());
            return json(['code' => -1, 'msg' => '添加到购物车失败']);
        }
    }
    
    /**
     * 获取购物车列表
     */
    public function list(Request $request): Response
    {
        try {
            $userId = AuthMiddleware::getUserId($request);
            if (!$userId) {
                return json(['code' => -1, 'msg' => '请先登录']);
            }
            
            $cartItems = $this->cartService->getCartItems($userId);
            
            // 计算总金额
            $totalAmount = $this->cartService->calculateTotalAmount($cartItems);
            
            return json([
                'code' => 0,
                'data' => [
                    'items' => $cartItems,
                    'total_amount' => $totalAmount,
                    'total_count' => count($cartItems)
                ],
                'msg' => 'success'
            ]);
            
        } catch (\Exception $e) {
            \support\Log::error('Get cart list failed: ' . $e->getMessage());
            return json(['code' => -1, 'msg' => '获取购物车失败']);
        }
    }
    
    /**
     * 更新购物车商品数量
     */
    public function update(Request $request): Response
    {
        try {
            $userId = AuthMiddleware::getUserId($request);
            $cartId = (int)$request->post('cart_id');
            $quantity = max(1, (int)$request->post('quantity', 1));
            
            if (!$cartId) {
                return json(['code' => -1, 'msg' => '购物车ID不能为空']);
            }
            
            // 1. 验证购物车项归属
            $cartItem = $this->cartService->getCartItemById($cartId);
            if (!$cartItem || $cartItem['user_id'] != $userId) {
                return json(['code' => -1, 'msg' => '购物车项不存在']);
            }
            
            // 2. 检查库存
            $stock = $this->productService->getStock($cartItem['product_id'], $cartItem['sku_id']);
            if ($stock < $quantity) {
                return json(['code' => -1, 'msg' => '库存不足']);
            }
            
            // 3. 更新数量
            $this->cartService->updateQuantity($cartId, $quantity);
            
            return json(['code' => 0, 'msg' => '更新成功']);
            
        } catch (\Exception $e) {
            \support\Log::error('Update cart failed: ' . $e->getMessage());
            return json(['code' => -1, 'msg' => '更新购物车失败']);
        }
    }
    
    /**
     * 删除购物车商品
     */
    public function delete(Request $request): Response
    {
        try {
            $userId = AuthMiddleware::getUserId($request);
            $cartIds = $request->post('cart_ids', []);
            
            if (empty($cartIds) || !is_array($cartIds)) {
                return json(['code' => -1, 'msg' => '请选择要删除的商品']);
            }
            
            // 验证购物车项归属并删除
            $deletedCount = $this->cartService->deleteCartItems($userId, $cartIds);
            
            return json([
                'code' => 0,
                'data' => ['deleted_count' => $deletedCount],
                'msg' => '删除成功'
            ]);
            
        } catch (\Exception $e) {
            \support\Log::error('Delete cart items failed: ' . $e->getMessage());
            return json(['code' => -1, 'msg' => '删除购物车商品失败']);
        }
    }
}
```

##### 3. 支付服务
```php
<?php
// app/service/PaymentService.php - 支付服务
namespace app\service;

use support\Db;
use support\Log;
use app\model\Order;
use app\model\Payment;

class PaymentService
{
    /**
     * 生成支付链接
     */
    public function generatePaymentUrl($order)
    {
        try {
            // 1. 创建支付记录
            $payment = [
                'order_id' => $order['id'],
                'order_no' => $order['order_no'],
                'user_id' => $order['user_id'],
                'amount' => $order['total_amount'],
                'payment_method' => $order['payment_method'],
                'status' => 'pending',
                'created_at' => date('Y-m-d H:i:s')
            ];
            
            $paymentId = Db::table('payments')->insertGetId($payment);
            
            // 2. 根据支付方式生成支付链接
            switch ($order['payment_method']) {
                case 'alipay':
                    return $this->generateAlipayUrl($order, $paymentId);
                case 'wechat':
                    return $this->generateWechatUrl($order, $paymentId);
                case 'unionpay':
                    return $this->generateUnionpayUrl($order, $paymentId);
                default:
                    throw new \Exception('不支持的支付方式');
            }
            
        } catch (\Exception $e) {
            Log::error('Generate payment URL failed: ' . $e->getMessage());
            throw $e;
        }
    }
    
    /**
     * 处理支付回调
     */
    public function handlePaymentCallback($paymentMethod, $callbackData)
    {
        try {
            // 1. 验证回调数据
            $verification = $this->verifyCallback($paymentMethod, $callbackData);
            if (!$verification['valid']) {
                throw new \Exception('回调数据验证失败');
            }
            
            // 2. 获取订单信息
            $orderNo = $verification['order_no'];
            $order = Db::table('orders')->where('order_no', $orderNo)->first();
            if (!$order) {
                throw new \Exception('订单不存在');
            }
            
            // 3. 检查订单状态
            if ($order['status'] != 'pending') {
                return ['success' => true, 'message' => '订单已处理'];
            }
            
            // 4. 更新支付状态（使用事务）
            Db::beginTransaction();
            try {
                // 4.1 更新支付记录
                Db::table('payments')
                    ->where('order_id', $order['id'])
                    ->update([
                        'status' => 'success',
                        'transaction_id' => $verification['transaction_id'],
                        'paid_at' => date('Y-m-d H:i:s'),
                        'callback_data' => json_encode($callbackData)
                    ]);
                
                // 4.2 更新订单状态
                Db::table('orders')
                    ->where('id', $order['id'])
                    ->update([
                        'status' => 'paid',
                        'paid_at' => date('Y-m-d H:i:s')
                    ]);
                
                Db::commit();
                
                // 5. 发送支付成功消息
                $this->sendPaymentSuccessMessage($order);
                
                return ['success' => true, 'message' => '支付成功'];
                
            } catch (\Exception $e) {
                Db::rollback();
                throw $e;
            }
            
        } catch (\Exception $e) {
            Log::error('Handle payment callback failed: ' . $e->getMessage());
            return ['success' => false, 'message' => $e->getMessage()];
        }
    }
    
    /**
     * 生成支付宝支付链接
     */
    private function generateAlipayUrl($order, $paymentId)
    {
        // 支付宝SDK集成实现
        $config = config('payment.alipay');
        
        $params = [
            'app_id' => $config['app_id'],
            'method' => 'alipay.trade.page.pay',
            'charset' => 'utf-8',
            'sign_type' => 'RSA2',
            'timestamp' => date('Y-m-d H:i:s'),
            'version' => '1.0',
            'notify_url' => $config['notify_url'],
            'return_url' => $config['return_url'],
            'biz_content' => json_encode([
                'out_trade_no' => $order['order_no'],
                'product_code' => 'FAST_INSTANT_TRADE_PAY',
                'total_amount' => $order['total_amount'],
                'subject' => '订单支付-' . $order['order_no']
            ])
        ];
        
        // 生成签名
        $params['sign'] = $this->generateAlipaySign($params, $config['private_key']);
        
        // 构建支付链接
        return $config['gateway'] . '?' . http_build_query($params);
    }
    
    /**
     * 验证回调数据
     */
    private function verifyCallback($paymentMethod, $callbackData)
    {
        switch ($paymentMethod) {
            case 'alipay':
                return $this->verifyAlipayCallback($callbackData);
            case 'wechat':
                return $this->verifyWechatCallback($callbackData);
            default:
                return ['valid' => false, 'message' => '不支持的支付方式'];
        }
    }
}
```

#### 四、数据库设计
```sql
-- 用户表
CREATE TABLE `users` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `username` varchar(50) NOT NULL COMMENT '用户名',
  `email` varchar(100) NOT NULL COMMENT '邮箱',
  `phone` varchar(20) DEFAULT NULL COMMENT '手机号',
  `password` varchar(255) NOT NULL COMMENT '密码',
  `avatar` varchar(255) DEFAULT NULL COMMENT '头像',
  `status` tinyint(1) DEFAULT '1' COMMENT '状态：1正常，0禁用',
  `created_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_username` (`username`),
  UNIQUE KEY `uk_email` (`email`),
  KEY `idx_phone` (`phone`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户表';

-- 商品表
CREATE TABLE `products` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(255) NOT NULL COMMENT '商品名称',
  `description` text COMMENT '商品描述',
  `category_id` int(11) unsigned NOT NULL COMMENT '分类ID',
  `brand_id` int(11) unsigned DEFAULT NULL COMMENT '品牌ID',
  `price` decimal(10,2) NOT NULL COMMENT '价格',
  `original_price` decimal(10,2) DEFAULT NULL COMMENT '原价',
  `stock` int(11) DEFAULT '0' COMMENT '库存',
  `sales` int(11) DEFAULT '0' COMMENT '销量',
  `images` json COMMENT '商品图片',
  `attributes` json COMMENT '商品属性',
  `status` tinyint(1) DEFAULT '1' COMMENT '状态：1上架，0下架',
  `sort_order` int(11) DEFAULT '0' COMMENT '排序',
  `created_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `idx_category` (`category_id`),
  KEY `idx_brand` (`brand_id`),
  KEY `idx_status` (`status`),
  KEY `idx_price` (`price`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='商品表';

-- 订单表
CREATE TABLE `orders` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `order_no` varchar(32) NOT NULL COMMENT '订单号',
  `user_id` int(11) unsigned NOT NULL COMMENT '用户ID',
  `total_amount` decimal(10,2) NOT NULL COMMENT '订单总金额',
  `discount_amount` decimal(10,2) DEFAULT '0.00' COMMENT '优惠金额',
  `shipping_amount` decimal(10,2) DEFAULT '0.00' COMMENT '运费',
  `payment_method` varchar(20) DEFAULT NULL COMMENT '支付方式',
  `status` enum('pending','paid','shipped','delivered','cancelled','refunded') DEFAULT 'pending' COMMENT '订单状态',
  `shipping_address` json COMMENT '收货地址',
  `remark` varchar(500) DEFAULT NULL COMMENT '备注',
  `paid_at` timestamp NULL DEFAULT NULL COMMENT '支付时间',
  `shipped_at` timestamp NULL DEFAULT NULL COMMENT '发货时间',
  `delivered_at` timestamp NULL DEFAULT NULL COMMENT '收货时间',
  `created_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_order_no` (`order_no`),
  KEY `idx_user_id` (`user_id`),
  KEY `idx_status` (`status`),
  KEY `idx_created_at` (`created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='订单表';

-- 购物车表
CREATE TABLE `cart_items` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `user_id` int(11) unsigned NOT NULL COMMENT '用户ID',
  `product_id` int(11) unsigned NOT NULL COMMENT '商品ID',
  `sku_id` int(11) unsigned DEFAULT NULL COMMENT 'SKU ID',
  `quantity` int(11) NOT NULL DEFAULT '1' COMMENT '数量',
  `price` decimal(10,2) NOT NULL COMMENT '加入时价格',
  `created_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_user_product_sku` (`user_id`,`product_id`,`sku_id`),
  KEY `idx_user_id` (`user_id`),
  KEY `idx_product_id` (`product_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='购物车表';
```

#### 五、性能优化策略
```php
<?php
// config/redis.php - Redis缓存配置
return [
    'default' => [
        'host' => '127.0.0.1',
        'port' => 6379,
        'auth' => '',
        'db' => 0,
        'max_connections' => 20,
    ],
    'cache' => [
        'host' => '127.0.0.1',
        'port' => 6379,
        'auth' => '',
        'db' => 1,
        'max_connections' => 10,
    ],
    'session' => [
        'host' => '127.0.0.1',
        'port' => 6379,
        'auth' => '',
        'db' => 2,
        'max_connections' => 5,
    ]
];

// app/service/CacheService.php - 缓存服务
<?php
namespace app\service;

use support\Redis;

class CacheService
{
    private $redis;
    
    public function __construct()
    {
        $this->redis = Redis::connection('cache');
    }
    
    /**
     * 商品缓存策略
     */
    public function cacheProduct($productId, $product, $ttl = 600)
    {
        $key = "product:{$productId}";
        $this->redis->setex($key, $ttl, json_encode($product));
        
        // 同时缓存到商品列表
        $this->redis->zadd('products:hot', $product['sales'], $productId);
    }
    
    /**
     * 库存缓存
     */
    public function cacheStock($productId, $skuId, $stock)
    {
        $key = "stock:{$productId}:{$skuId}";
        $this->redis->set($key, $stock);
    }
    
    /**
     * 购物车缓存
     */
    public function cacheCart($userId, $cartItems)
    {
        $key = "cart:{$userId}";
        $this->redis->setex($key, 3600, json_encode($cartItems));
    }
    
    /**
     * 分布式锁
     */
    public function lock($key, $ttl = 10)
    {
        $lockKey = "lock:{$key}";
        $identifier = uniqid();
        
        if ($this->redis->set($lockKey, $identifier, 'EX', $ttl, 'NX')) {
            return $identifier;
        }
        
        return false;
    }
    
    public function unlock($key, $identifier)
    {
        $lockKey = "lock:{$key}";
        $script = '
            if redis.call("get", KEYS[1]) == ARGV[1] then
                return redis.call("del", KEYS[1])
            else
                return 0
            end
        ';
        
        return $this->redis->eval($script, [$lockKey, $identifier], 1);
    }
}
```

**总结**：电商系统的核心在于高并发处理、数据一致性保证和用户体验优化。通过Webman框架的高性能特性，结合Redis缓存、消息队列、分布式锁等技术，可以构建一个稳定可靠的电商平台。关键是要在性能、一致性和用户体验之间找到最佳平衡点。
</details>
