---
title: php面经-数据库篇
createTime: 2025/08/29 10:46:11
permalink: /php/面试题/database/
---
## 1. MySQL 中常用的索引类型有哪些？创建联合索引时，为什么要遵循“最左前缀原则”？

<details>
<summary>答案与解析</summary>

- 思路要点：

  1. **MySQL 常用索引类型**：

     - **主键索引（PRIMARY KEY）**：特殊的唯一索引，不允许 NULL 值，一个表只能有一个主键索引，用于唯一标识表中记录，通常自动创建（若未显式指定，InnoDB 会隐式生成）。
     - **唯一索引（UNIQUE）**：确保索引列的值唯一（允许 NULL 值，且多个 NULL 不冲突），用于防止重复数据（如用户邮箱、手机号）。
     - **普通索引（INDEX）**：最基本的索引类型，无唯一性限制，仅用于加速查询，可创建多个。
     - **联合索引（Composite Index）**：由多个列组合创建的索引（如 `INDEX idx_name_age (name, age)`），用于优化多列联合查询。
     - **全文索引（FULLTEXT）**：用于长文本字段（如 `TEXT`）的全文搜索，支持关键词匹配，不适合精确查询。
     - **空间索引（SPATIAL）**：用于地理空间数据类型（如 `GEOMETRY`），加速空间位置相关查询（较少使用）。
  2. **联合索引需遵循“最左前缀原则”的原因**：

     - 联合索引的底层存储结构为 B+ 树，索引列按创建时的顺序排序（如 `(a, b, c)` 按 `a` 排序，`a` 相同时按 `b` 排序，以此类推）。
     - “最左前缀原则”指：查询条件中必须包含联合索引的最左侧列，且按索引列顺序连续匹配（如 `(a, b, c)` 索引，能匹配 `a`、`a AND b`、`a AND b AND c`，但无法匹配 `b`、`b AND c`、`a AND c` 等非最左前缀组合）。
     - 若不遵循该原则，MySQL 无法利用联合索引的有序性定位数据，会导致索引失效，退化为全表扫描，降低查询效率。

  - 示例代码：

  ```sql
  -- 1. 创建各种索引示例（基于用户表）
  CREATE TABLE users (
      id INT NOT NULL AUTO_INCREMENT,
      username VARCHAR(50) NOT NULL,
      email VARCHAR(100) NOT NULL,
      age INT,
      address TEXT,
      PRIMARY KEY (id), -- 主键索引
      UNIQUE INDEX idx_email (email), -- 唯一索引（防止邮箱重复）
      INDEX idx_age (age), -- 普通索引（加速按年龄查询）
      INDEX idx_username_age (username, age), -- 联合索引（用户名+年龄）
      FULLTEXT INDEX idx_address (address) -- 全文索引（用于地址搜索）
  );

  -- 2. 联合索引（username, age）的最左前缀原则演示
  -- 情况1：遵循最左前缀，能使用联合索引
  EXPLAIN SELECT * FROM users WHERE username = 'zhangsan'; -- 匹配索引第1列，有效
  EXPLAIN SELECT * FROM users WHERE username = 'zhangsan' AND age = 25; -- 匹配索引第1+2列，有效

  -- 情况2：不遵循最左前缀，索引失效（无法使用联合索引）
  EXPLAIN SELECT * FROM users WHERE age = 25; -- 缺少最左列username，索引失效
  EXPLAIN SELECT * FROM users WHERE username = 'zhangsan' OR age = 25; -- OR导致索引失效
  EXPLAIN SELECT * FROM users WHERE age = 25 AND username = 'zhangsan'; -- 顺序不影响（MySQL会优化顺序），但需包含最左列

  -- 3. 联合索引设计最佳实践：将区分度高的列放在左侧
  -- 例：若username区分度高于age，(username, age) 比 (age, username) 更高效
  -- 因为区分度高的列能更快缩小查询范围
  ```

  注：联合索引的“最左前缀”不仅适用于等值查询，也适用于范围查询（如 `username LIKE 'zhang%' AND age = 25` 可利用索引），但范围查询后的列无法再使用索引（如 `username = 'zhangsan' AND age > 25 AND status = 1` 中，`status` 列无法利用 `(username, age, status)` 索引）。

</details>

## 2. 写出一个查询用户表（users）中年龄大于 18 岁、注册时间在 30 天内的用户，并按注册时间倒序排列的 SQL 语句；如何确认该查询是否使用了索引（提示：EXPLAIN）？

<details>
<summary>答案与解析</summary>

- 思路要点：

  1. 查询条件分析：需同时满足“年龄 > 18 岁”和“注册时间在30天内”，其中“30天内”指注册时间晚于（或等于）当前时间减去30天，需使用MySQL日期函数计算；最终按注册时间倒序（最新的在前）排列。
  2. EXPLAIN的作用：通过在查询前添加 `EXPLAIN`，分析MySQL执行计划，判断是否使用索引（主要关注 `key` 列是否显示索引名称）。
- 示例 SQL（修正后）：

```sql
-- 正确查询：年龄>18岁、30天内注册的用户，按注册时间倒序
SELECT * 
FROM users 
WHERE age > 18 
  AND created_at >= DATE_SUB(CURDATE(), INTERVAL 30 DAY) -- 30天内（包含今天）
ORDER BY created_at DESC;

-- 若created_at包含时间戳（精确到时分秒），可使用更精确的条件：
SELECT * 
FROM users 
WHERE age > 18 
  AND created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY) -- 精确到30天前的当前时间
ORDER BY created_at DESC;
```

- EXPLAIN 使用说明：

1. **执行方法**：在查询语句前添加 `EXPLAIN`，例如：

   ```sql
   EXPLAIN
   SELECT * 
   FROM users 
   WHERE age > 18 
     AND created_at >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
   ORDER BY created_at DESC;
   ```
2. **关键字段解读**（判断是否使用索引）：

   - `type`：表示访问类型，常见值从优到差为 `const` > `eq_ref` > `ref` > `range` > `index` > `ALL`。若为 `range`（范围查询）、`ref` 等，说明可能使用了索引；若为 `ALL`，表示全表扫描（未使用索引）。
   - `key`：显示MySQL实际使用的索引名称。若该字段为 `NULL`，表示未使用任何索引；若显示索引名（如 `idx_age_created_at`），说明使用了对应索引。
   - `key_len`：表示索引使用的字节数，可判断联合索引中使用了哪些列。
   - `Extra`：额外信息，若出现 `Using index` 表示使用了覆盖索引；`Using where; Using filesort` 表示虽使用了索引过滤条件，但排序未使用索引（可能需要优化）。
3. **优化建议**：若查询未使用索引，可创建联合索引 `idx_age_created_at (age, created_at)`，因查询条件包含 `age` 和 `created_at`，且排序字段为 `created_at`，该索引可同时优化过滤和排序。

</details>

## 3. PHP 中操作 MySQL 的方式有哪些？PDO 相比原生 mysql_* 函数有什么优势（如预处理防注入、支持多数据库）？

<details>
<summary>答案与解析</summary>

- 思路要点：

  1. **PHP 操作 MySQL 的主要方式**：

     - **原生 SQL 方式**：通过 `mysql_*` 函数（已废弃，PHP5.5+ 不推荐）或 `mysqli_*` 函数（MySQL 增强扩展）直接执行 SQL 语句，需手动处理连接、查询、结果集和错误。
     - **PDO（PHP Data Objects）**：数据库访问抽象层，提供统一接口操作多种数据库（MySQL、PostgreSQL、SQLite 等），支持预处理语句和面向对象编程。
     - **ORM（对象关系映射）**：如 Laravel 的 Eloquent、Doctrine 等，通过对象封装数据库操作，将表映射为类、记录映射为对象，无需直接编写 SQL，简化开发。
  2. **PDO 相比原生 `mysql_*` 函数的优势**：

     - **安全性更高**：内置预处理语句（`prepare()` + `execute()`），自动参数化查询，有效防止 SQL 注入（`mysql_*` 需手动转义，易遗漏）。
     - **跨数据库支持**：同一套接口可操作多种数据库，切换数据库时无需大量修改代码（`mysql_*` 仅支持 MySQL）。
     - **面向对象特性**：基于类和对象设计，支持异常处理（`PDOException`），错误处理更灵活（`mysql_*` 主要是过程化函数，错误处理繁琐）。
     - **功能更完善**：支持事务管理、游标、LOB（大对象）处理等高级特性（`mysql_*` 功能有限，且部分函数已废弃）。
     - **性能更优**：预处理语句可重复执行，减少 SQL 解析开销（尤其适合批量操作）。

  - 示例代码：

  ```php
  <?php
  // 1. 原生 mysqli_* 函数示例（不推荐，仅作对比）
  $conn = mysqli_connect('localhost', 'user', 'pass', 'db');
  if (!$conn) {
      die('连接失败: ' . mysqli_connect_error());
  }
  $age = 18;
  // 需手动转义，否则有注入风险
  $sql = "SELECT * FROM users WHERE age > " . mysqli_real_escape_string($conn, $age);
  $result = mysqli_query($conn, $sql);
  while ($row = mysqli_fetch_assoc($result)) {
      // 处理结果
  }
  mysqli_close($conn);


  // 2. PDO 示例（推荐）
  try {
      $pdo = new PDO('mysql:host=localhost;dbname=db', 'user', 'pass');
      $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);

      // 预处理防注入
      $age = 18;
      $stmt = $pdo->prepare("SELECT * FROM users WHERE age > :age");
      $stmt->bindParam(':age', $age, PDO::PARAM_INT);
      $stmt->execute();

      // 获取结果
      while ($row = $stmt->fetch(PDO::FETCH_ASSOC)) {
          // 处理结果
      }
  } catch (PDOException $e) {
      die('错误: ' . $e->getMessage());
  }


  // 3. ORM 示例（Laravel Eloquent）
  use App\Models\User;

  // 无需编写 SQL，通过对象操作
  $users = User::where('age', '>', 18)->get();
  foreach ($users as $user) {
      // 直接访问对象属性
      echo $user->name;
  }
  ?>
  ```

</details>

## 4. 什么是 SQL 注入？如何通过 PDO 的预处理语句（prepare() + execute()）防止 SQL 注入？举例说明不安全的写法和安全的写法

<details>
<summary>答案与解析</summary>

- 思路要点：

  1. **SQL注入的定义**：SQL注入是一种恶意攻击手段，指攻击者通过在用户输入中插入SQL代码片段，使数据库执行非预期的SQL指令（如删除数据、泄露敏感信息等）。其本质是程序未对用户输入进行严格过滤，导致输入数据被当作SQL代码的一部分执行。
  2. **PDO预处理语句防止SQL注入的原理**：

     - 预处理语句将SQL模板（结构）与用户输入的参数分离，先由数据库编译SQL模板（确定执行逻辑），再传入参数执行。
     - 参数在传递过程中会被数据库自动视为纯数据（而非SQL代码），即使包含特殊字符（如单引号、关键字）也不会被解析为SQL指令，从而从根本上阻止注入。
- 不安全写法（直接拼接用户输入，存在注入风险）：

```php
<?php
// 模拟用户输入（攻击者可能输入恶意内容）
$userInputId = $_GET['id']; // 假设输入：1' OR '1'='1

// 直接拼接SQL语句（危险！）
$pdo = new PDO('mysql:host=localhost;dbname=test', 'root', '');
$sql = "SELECT * FROM users WHERE id = '" . $userInputId . "'"; 
// 拼接后实际执行的SQL：SELECT * FROM users WHERE id = '1' OR '1'='1'
// 该语句会查询所有用户数据，导致敏感信息泄露

$stmt = $pdo->query($sql);
$result = $stmt->fetchAll(PDO::FETCH_ASSOC);
print_r($result);
?>
```

**风险说明**：攻击者输入的 `1' OR '1'='1`被拼接进SQL后，原本的查询条件被篡改，变为“查询id=1或1=1”（恒为真），导致返回表中所有记录。

- 安全写法（PDO预处理语句，防止注入）：

```php
<?php
// 模拟用户输入（即使包含恶意内容也安全）
$userInputId = $_GET['id']; // 输入：1' OR '1'='1

$pdo = new PDO('mysql:host=localhost;dbname=test', 'root', '');
$pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);

// 1. 准备SQL模板（使用占位符:id）
$sql = "SELECT * FROM users WHERE id = :id";
$stmt = $pdo->prepare($sql); // 数据库预编译SQL结构

// 2. 绑定参数并执行（参数被视为纯数据）
$stmt->execute([':id' => $userInputId]); 
// 等价写法：$stmt->bindParam(':id', $userInputId, PDO::PARAM_INT); $stmt->execute();

$result = $stmt->fetchAll(PDO::FETCH_ASSOC);
print_r($result);
?>
```

**安全说明**：预处理语句中，`:id`是占位符，`$userInputId`仅作为参数传递。即使输入包含 `'`或 `OR`等特殊字符，数据库也会将其视为 `id`字段的查询值（如查询 `id = "1' OR '1'='1"`，而非解析为SQL逻辑），从而避免注入。

</details>

## 5. 如何优化 MySQL 的慢查询？请列举至少 3 种常见方法

<details>
<summary>答案与解析</summary>

- 方法清单：

  1. **合理添加索引**

     - 原理：索引通过B+树结构加速数据查找，减少全表扫描（full table scan）的开销。
     - 做法：针对查询中 `WHERE`、`JOIN`、`ORDER BY`涉及的列创建索引（如频繁过滤的 `status`、关联查询的 `user_id`）；优先创建联合索引（按最左前缀原则）；避免对低区分度列（如 `gender`）创建索引（收益低）。
     - 注意：索引会降低写入（INSERT/UPDATE/DELETE）性能，需平衡查询与写入需求，定期删除冗余或低效索引。
  2. **避免 `SELECT *`，只查询必要字段**

     - 原理：`SELECT *`会读取表中所有字段（包括大字段如 `TEXT`、`BLOB`），增加磁盘IO和网络传输量，且无法利用覆盖索引（Covering Index）。
     - 做法：明确指定所需字段，如 `SELECT id, name FROM users`而非 `SELECT * FROM users`。
     - 优势：减少数据传输量，若查询字段均在索引中，可直接从索引获取数据（覆盖索引），避免回表查询。
  3. **拆分大SQL，避免一次性处理过多数据**

     - 原理：大SQL（如批量删除10万行、复杂多表JOIN）会长时间占用锁资源（表锁/行锁），阻塞其他查询，甚至导致连接超时。
     - 做法：
       - 批量操作拆分：如 `DELETE FROM logs WHERE created_at < '2023-01-01'`拆分为 `LIMIT 1000`的循环删除（每次删1000行，间隔短时间）。
       - 复杂查询拆分：将多表JOIN的大查询拆分为多个单表查询，在应用层拼接结果（减少数据库压力）。
  4. **优化JOIN查询，减少关联表数量和数据量**

     - 原理：JOIN查询会产生临时表，表越多、数据量越大，临时表处理开销越高（尤其无索引时）。
     - 做法：
       - 限制JOIN表数量（建议不超过3-4张），避免多表嵌套关联。
       - 提前过滤数据：在 `JOIN`前用 `WHERE`筛选子表数据（如 `SELECT * FROM orders o JOIN users u ON o.user_id = u.id WHERE u.status = 1`，确保 `u.status`有索引）。
       - 优先使用 `INNER JOIN`（结果集小），避免 `LEFT JOIN`无过滤条件导致的全表关联。
  5. **优化子查询，改用JOIN或临时表**

     - 原理：子查询（尤其嵌套子查询）可能导致MySQL多次扫描表，效率低；JOIN通常通过索引一次性关联数据，性能更优。
     - 做法：将 `WHERE id IN (SELECT ...)`改为 `JOIN`，如：
       - 低效：`SELECT * FROM posts WHERE user_id IN (SELECT id FROM users WHERE status = 1)`
       - 优化：`SELECT p.* FROM posts p JOIN users u ON p.user_id = u.id WHERE u.status = 1`
- 示例代码（优化前后对比）：

```sql
-- 1. 索引优化示例（查询慢：无索引）
-- 慢查询：SELECT * FROM orders WHERE user_id = 123 AND created_at > '2024-01-01' ORDER BY total DESC;
-- 优化：创建联合索引（匹配WHERE和ORDER BY）
CREATE INDEX idx_user_created_total ON orders (user_id, created_at, total);


-- 2. 避免SELECT *示例
-- 低效：SELECT * FROM products WHERE category_id = 5;（返回所有字段，包括大字段description）
-- 优化：只查必要字段
SELECT id, name, price FROM products WHERE category_id = 5;


-- 3. 拆分大SQL示例（批量删除）
-- 危险：DELETE FROM logs WHERE created_at < '2023-01-01';（可能锁表10秒+）
-- 优化：循环批量删除
WHILE (SELECT COUNT(*) FROM logs WHERE created_at < '2023-01-01') > 0 DO
  DELETE FROM logs WHERE created_at < '2023-01-01' LIMIT 1000;
  SELECT SLEEP(0.1); -- 间隔0.1秒，避免阻塞
END WHILE;


-- 4. JOIN优化示例
-- 低效：多表JOIN无过滤（全表关联）
SELECT o.id, u.name, p.title 
FROM orders o
LEFT JOIN users u ON o.user_id = u.id
LEFT JOIN products p ON o.product_id = p.id;

-- 优化：添加过滤条件，限制数据量
SELECT o.id, u.name, p.title 
FROM orders o
LEFT JOIN users u ON o.user_id = u.id AND u.status = 1 -- 提前过滤有效用户
LEFT JOIN products p ON o.product_id = p.id AND p.stock > 0 -- 过滤有库存商品
WHERE o.created_at > '2024-01-01'; -- 限制订单时间范围
```

- 辅助工具：通过开启MySQL慢查询日志（`slow_query_log = 1`）记录慢查询，结合 `EXPLAIN`分析执行计划，定位优化点（如 `type=ALL`表示全表扫描，需加索引）。

</details>

## 6. 什么是 MySQL 事务？事务的 ACID 特性分别指什么？如何通过 PHP 的 PDO 实现事务处理（举例说明，如转账场景）？

<details>
<summary>答案与解析</summary>

- 思路要点：  
  1. **事务定义**：事务是数据库中一组不可分割的操作单元，要么全部执行成功（提交），要么全部执行失败（回滚），用于保证数据一致性（如转账、订单创建等场景）。  
  2. **ACID 特性拆解**：需逐一解释原子性（Atomicity）、一致性（Consistency）、隔离性（Isolation）、持久性（Durability）的核心含义，明确各特性解决的问题。  
  3. **PDO 事务实现**：利用 PDO 提供的 `beginTransaction()`（开启事务）、`commit()`（提交）、`rollBack()`（回滚）API，结合异常处理（`PDOException`）确保事务原子性，以“转账”为典型场景举例（A 账户减钱、B 账户加钱需同时成功）。


### 一、事务与 ACID 特性解析
#### 1. 事务的核心作用
解决“多步操作的一致性问题”——例如转账时，“A 账户扣 100 元”和“B 账户加 100 元”必须同时完成，若中间出错（如网络中断、SQL 错误），需回滚到操作前状态，避免出现“钱扣了没加”的异常数据。

#### 2. 事务的 ACID 特性
| 特性         | 英文全称                | 核心含义                                                                 | 示例（转账场景）                                                                 |
|--------------|-------------------------|--------------------------------------------------------------------------|----------------------------------------------------------------------------------|
| **原子性**   | Atomicity               | 事务是“不可分割的最小单元”，要么全执行，要么全不执行，无中间状态。       | 若“A 扣钱”成功但“B 加钱”失败，事务回滚，A 的钱恢复，避免“钱消失”。               |
| **一致性**   | Consistency             | 事务执行前后，数据库数据需满足预设的业务规则（如“总金额守恒”）。         | 转账前 A+B 总金额 = 1000 元，事务执行后总金额仍为 1000 元，不会多也不会少。       |
| **隔离性**   | Isolation               | 多个事务并发执行时，彼此隔离，一个事务的中间状态不会被其他事务看到。     | 事务 1 正在给 A 转账，事务 2 查询 A 账户时，看不到“扣钱未加钱”的中间状态。       |
| **持久性**   | Durability              | 事务提交后，数据修改会永久保存到数据库（即使数据库宕机，数据也不丢失）。 | 转账事务提交后，A 扣钱、B 加钱的结果会写入磁盘，重启 MySQL 后数据仍有效。       |


### 二、PHP PDO 实现事务处理（转账场景示例）
#### 1. 场景需求
实现“用户 A（id=1）向用户 B（id=2）转账 100 元”，需满足：
- A 账户余额 ≥ 100 元（前置校验）；
- A 余额减 100 元，B 余额加 100 元（两步操作）；
- 若任意一步失败，整体回滚。

#### 2. 数据库表结构（用户余额表）
```sql
CREATE TABLE `user_balance` (
  `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `user_id` INT NOT NULL UNIQUE COMMENT '用户ID',
  `balance` DECIMAL(10,2) NOT NULL DEFAULT 0.00 COMMENT '账户余额',
  `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

-- 初始化数据：A（user_id=1）余额 500 元，B（user_id=2）余额 300 元
INSERT INTO `user_balance` (`user_id`, `balance`) VALUES (1, 500.00), (2, 300.00);
```

#### 3. PDO 事务代码实现
```php
<?php
try {
    // 1. 连接数据库（InnoDB 引擎才支持事务，MyISAM 不支持）
    $pdo = new PDO(
        'mysql:host=localhost;dbname=test;charset=utf8',
        'root',
        'your_password',
        [
            PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION, // 开启异常模式（关键，捕获事务错误）
            PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC
        ]
    );

    // 2. 开启事务（关闭自动提交，后续 SQL 需手动提交）
    $pdo->beginTransaction();

    // 3. 业务逻辑：A 账户减 100 元（先校验余额）
    $fromUserId = 1;
    $toUserId = 2;
    $amount = 100.00;

    // 3.1 查 A 账户当前余额（加行锁，避免并发修改，InnoDB 行锁基于索引）
    $stmt = $pdo->prepare("SELECT balance FROM user_balance WHERE user_id = :user_id FOR UPDATE");
    $stmt->execute([':user_id' => $fromUserId]);
    $fromUser = $stmt->fetch();
    if (!$fromUser || $fromUser['balance'] < $amount) {
        throw new PDOException("A 账户余额不足，无法转账"); // 抛出异常触发回滚
    }

    // 3.2 A 账户扣钱
    $stmt = $pdo->prepare("UPDATE user_balance SET balance = balance - :amount WHERE user_id = :user_id");
    $stmt->execute([':amount' => $amount, ':user_id' => $fromUserId]);

    // 3.3 B 账户加钱（模拟可能出错的场景：如故意写错字段名触发错误）
    $stmt = $pdo->prepare("UPDATE user_balance SET balance = balance + :amount WHERE user_id = :user_id");
    $stmt->execute([':amount' => $amount, ':user_id' => $toUserId]);

    // 4. 提交事务（所有操作成功，永久保存数据）
    $pdo->commit();
    echo "转账成功！A 账户扣减 {$amount} 元，B 账户增加 {$amount} 元";

} catch (PDOException $e) {
    // 5. 事务回滚（任意一步出错，恢复到事务开启前状态）
    if (isset($pdo) && !$pdo->inTransaction()) {
        $pdo->rollBack();
    }
    echo "转账失败：" . $e->getMessage();
}
```

#### 4. 关键说明
- **引擎支持**：仅 `InnoDB` 存储引擎支持事务，`MyISAM` 不支持（创建表时需指定 `ENGINE=InnoDB`）；
- **异常模式**：必须开启 `PDO::ERRMODE_EXCEPTION`，否则 SQL 错误不会抛出异常，导致事务无法回滚；
- **锁机制**：查询余额时用 `FOR UPDATE` 加行锁，防止并发转账时“超卖”（如 A 余额 100 元，两个事务同时扣钱导致余额为负）；
- **事务边界**：`beginTransaction()` 后，所有 SQL 都在事务内，直到 `commit()` 或 `rollBack()`。


### 总结
事务通过 ACID 特性保证数据一致性，PDO 事务处理的核心是“开启事务→执行操作→成功提交/失败回滚”，结合异常处理确保原子性。实际开发中，需优先使用 `InnoDB` 引擎，并用行锁避免并发问题。
</details>


## 7. MySQL 中 InnoDB 和 MyISAM 两种存储引擎有什么核心区别？如何根据业务场景选择合适的存储引擎？

<details>
<summary>答案与解析</summary>

- 思路要点：  
  1. **核心区别维度**：从“事务支持”“锁机制”“索引特性”“存储结构”“崩溃恢复”五个关键维度对比，这是两者最本质的差异；  
  2. **业务场景匹配**：根据“是否需要事务”“读写比例”“并发量”三个核心业务指标选择，例如事务场景必选 InnoDB，读多写少且无事务场景可选 MyISAM；  
  3. **实践建议**：当前 MySQL 8.0 已默认使用 InnoDB，MyISAM 逐渐被淘汰，需说明 MyISAM 的局限性和 InnoDB 的优势。


### 一、InnoDB 与 MyISAM 核心区别对比
| 对比维度                | InnoDB（默认引擎）                          | MyISAM（旧引擎）                          |
|-------------------------|-------------------------------------------|-------------------------------------------|
| **事务支持**            | 支持 ACID 事务（begin/commit/rollBack）    | 不支持事务，操作原子性差                  |
| **锁机制**              | 支持行锁（基于索引）和表锁，粒度小，并发高  | 仅支持表锁，粒度大，并发低（写操作阻塞全表） |
| **索引特性**            | 聚簇索引（主键索引与数据存储在一起），辅助索引指向主键 | 非聚簇索引（索引与数据分离），主键索引与普通索引无区别 |
| **存储结构**            | 数据存储在 `.ibd` 文件（表空间文件），支持表空间拆分 | 数据存 `.MYD` 文件，索引存 `.MYI` 文件，分开存储 |
| **崩溃恢复**            | 支持崩溃后数据恢复（基于 redo/undo 日志）    | 不支持崩溃恢复，数据易丢失（无日志）        |
| **外键约束**            | 支持外键（FOREIGN KEY），保证数据关联性      | 不支持外键，需在应用层维护关联逻辑          |
| **全文索引**            | MySQL 5.6+ 支持全文索引（仅 InnoDB 5.6+）    | 原生支持全文索引，但功能不如 InnoDB 完善    |
| **适用场景**            | 事务场景（订单、支付）、高并发读写场景      | 读多写少场景（博客、新闻）、无事务需求场景  |


### 二、关键差异深度解析
#### 1. 锁机制差异（最影响并发）
- **InnoDB 行锁**：  
  仅锁定“满足条件的行”，粒度小，适合高并发写操作。例如：  
  ```sql
  -- InnoDB 表：仅锁定 user_id=1 的行，其他行可正常读写
  UPDATE user_balance SET balance = 100 WHERE user_id = 1;
  ```
  注意：行锁依赖索引，若查询条件无索引（如 `WHERE age = 25` 且 `age` 无索引），会退化为表锁。

- **MyISAM 表锁**：  
  写操作（INSERT/UPDATE/DELETE）会锁定整个表，其他读写操作需等待锁释放。例如：  
  ```sql
  -- MyISAM 表：执行时锁定整个 user_balance 表，其他请求阻塞
  UPDATE user_balance SET balance = 100 WHERE user_id = 1;
  ```
  读操作（SELECT）不会阻塞其他读，但会阻塞写；写操作会阻塞所有读写。

#### 2. 索引结构差异（影响查询性能）
- **InnoDB 聚簇索引**：  
  主键索引（PRIMARY KEY）的叶子节点直接存储数据行，辅助索引（如 `INDEX idx_age (age)`）的叶子节点存储“主键值”，查询时需通过主键回表取数据（覆盖索引可避免回表）。  
  优势：主键查询极快，适合按主键频繁查询的场景。

- **MyISAM 非聚簇索引**：  
  所有索引（包括主键）的叶子节点都存储“数据行的物理地址”，查询时通过地址直接定位数据。  
  优势：索引结构简单，读操作性能略高；劣势：主键无特殊优化，且无事务支持。

#### 3. 崩溃恢复差异（影响数据安全性）
- **InnoDB**：通过 `redo 日志`（记录已执行的 SQL）和 `undo 日志`（记录数据修改前的状态），崩溃后可恢复到事务提交前的一致状态，数据安全性高。  
- **MyISAM**：无日志机制，崩溃后可能导致数据文件损坏（如 `.MYD` 或 `.MYI` 文件损坏），需通过 `myisamchk` 工具修复，且可能丢失数据。


### 三、业务场景选择建议
#### 1. 优先选择 InnoDB 的场景
- **事务需求场景**：订单创建、支付转账、用户注册（需保证数据一致性）；  
- **高并发读写场景**：电商商品详情页（多用户同时读写库存、销量）；  
- **数据安全性要求高**：金融数据、用户敏感信息（需崩溃恢复能力）；  
- **需外键约束**：多表关联场景（如订单表关联用户表、商品表）。

#### 2. 可选 MyISAM 的场景（极少）
- **纯读场景**：静态博客、新闻列表（无写操作，或写操作极少）；  
- **临时表/统计表**：数据可重建，无需事务和崩溃恢复（如每日访问量统计）；  
- **兼容旧系统**：维护基于 MyISAM 的 legacy 系统（建议逐步迁移到 InnoDB）。

#### 3. 实践示例（创建表时指定引擎）
```sql
-- 1. 事务场景（订单表）：必选 InnoDB
CREATE TABLE `orders` (
  `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `user_id` INT NOT NULL,
  `amount` DECIMAL(10,2) NOT NULL,
  `status` TINYINT NOT NULL DEFAULT 0,
  FOREIGN KEY (`user_id`) REFERENCES `users`(`id`) -- 外键约束（仅 InnoDB 支持）
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

-- 2. 纯读场景（新闻表）：可选 MyISAM（但建议用 InnoDB）
CREATE TABLE `news` (
  `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `title` VARCHAR(200) NOT NULL,
  `content` TEXT NOT NULL,
  `created_at` DATETIME NOT NULL
) ENGINE=MyISAM DEFAULT CHARSET=utf8;
```


### 总结
InnoDB 在事务、并发、数据安全性上全面优于 MyISAM，当前 MySQL 8.0 已移除 MyISAM 作为默认引擎的选项。实际开发中，除“纯读且无数据安全需求”的极端场景外，均应优先选择 InnoDB，避免因 MyISAM 的局限性导致业务风险。
</details>


## 8. 什么是乐观锁和悲观锁？在 MySQL 中如何实现这两种锁？结合 PHP 代码举例说明（如库存扣减场景）。

<details>
<summary>答案与解析</summary>

- 思路要点：  
  1. **锁的核心目的**：解决“并发修改冲突”（如多用户同时扣减同一商品库存，避免超卖）；  
  2. **两种锁的本质差异**：  
     - 悲观锁：“先加锁，再操作”，认为并发冲突一定会发生，通过数据库锁机制阻塞其他操作；  
     - 乐观锁：“先操作，后校验”，认为并发冲突概率低，通过版本号/时间戳等机制检测冲突，冲突后重试；  
  3. **落地场景匹配**：写操作频繁、冲突概率高（如秒杀）用悲观锁；读操作频繁、冲突概率低（如普通库存扣减）用乐观锁。


### 一、乐观锁与悲观锁核心概念对比
| 锁类型   | 核心思想                | 实现方式                          | 优点                                  | 缺点                                  | 适用场景                          |
|----------|-------------------------|-----------------------------------|---------------------------------------|---------------------------------------|-----------------------------------|
| **悲观锁** | 假设冲突必然发生，提前加锁 | 数据库行锁（SELECT ... FOR UPDATE）、表锁 | 无冲突重试逻辑，代码简单；数据一致性高 | 加锁阻塞，并发性能低；可能产生死锁    | 秒杀、高冲突写场景                |
| **乐观锁** | 假设冲突概率低，事后校验 | 版本号（version）、时间戳（updated_at） | 无锁阻塞，并发性能高；无死锁风险      | 冲突后需重试，代码复杂；高冲突时重试频繁 | 普通库存扣减、低冲突写场景        |


### 二、MySQL + PHP 实现示例（库存扣减场景）
假设存在商品库存表 `product_stock`，结构如下：
```sql
CREATE TABLE `product_stock` (
  `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `product_id` INT NOT NULL UNIQUE COMMENT '商品ID',
  `stock` INT NOT NULL DEFAULT 0 COMMENT '库存数量',
  `version` INT NOT NULL DEFAULT 1 COMMENT '版本号（乐观锁用）',
  `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

-- 初始化数据：商品ID=1001，库存=100，版本号=1
INSERT INTO `product_stock` (`product_id`, `stock`, `version`) VALUES (1001, 100, 1);
```


#### 1. 悲观锁实现（PHP + PDO）
**核心逻辑**：查询库存时加行锁（`FOR UPDATE`），确保其他事务无法同时修改该商品库存，扣减后直接提交。

```php
<?php
/**
 * 悲观锁实现库存扣减（商品ID=1001，扣减数量=1）
 */
function deductStockPessimistic($pdo, $productId, $deductNum = 1) {
    try {
        // 1. 开启事务
        $pdo->beginTransaction();

        // 2. 查询库存并加行锁（InnoDB 行锁基于索引，product_id 需有索引）
        $stmt = $pdo->prepare("
            SELECT stock 
            FROM product_stock 
            WHERE product_id = :product_id 
            FOR UPDATE; -- 加行锁，阻塞其他事务对该商品的修改
        ");
        $stmt->execute([':product_id' => $productId]);
        $stockInfo = $stmt->fetch();

        // 3. 校验库存
        if (!$stockInfo || $stockInfo['stock'] < $deductNum) {
            throw new PDOException("库存不足，当前库存：" . ($stockInfo['stock'] ?? 0));
        }

        // 4. 扣减库存
        $stmt = $pdo->prepare("
            UPDATE product_stock 
            SET stock = stock - :deduct_num 
            WHERE product_id = :product_id;
        ");
        $stmt->execute([
            ':deduct_num' => $deductNum,
            ':product_id' => $productId
        ]);

        // 5. 提交事务
        $pdo->commit();
        echo "悲观锁：库存扣减成功！剩余库存：" . ($stockInfo['stock'] - $deductNum);
        return true;

    } catch (PDOException $e) {
        // 6. 回滚事务
        if ($pdo->inTransaction()) {
            $pdo->rollBack();
        }
        echo "悲观锁：库存扣减失败：" . $e->getMessage();
        return false;
    }
}

// 调用
$pdo = new PDO(
    'mysql:host=localhost;dbname=test;charset=utf8',
    'root',
    'your_password',
    [PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION]
);
deductStockPessimistic($pdo, 1001, 1);
```


#### 2. 乐观锁实现（PHP + PDO）
**核心逻辑**：扣减库存时，通过版本号（`version`）校验数据是否被修改——仅当“当前版本号 == 数据库版本号”时才允许扣减，否则视为冲突，重试操作。

```php
<?php
/**
 * 乐观锁实现库存扣减（支持重试机制）
 */
function deductStockOptimistic($pdo, $productId, $deductNum = 1, $maxRetry = 3) {
    $retryCount = 0; // 重试次数

    while ($retryCount < $maxRetry) {
        try {
            // 1. 查询当前库存和版本号（无锁，仅读取）
            $stmt = $pdo->prepare("
                SELECT stock, version 
                FROM product_stock 
                WHERE product_id = :product_id;
            ");
            $stmt->execute([':product_id' => $productId]);
            $stockInfo = $stmt->fetch();

            // 2. 校验库存
            if (!$stockInfo || $stockInfo['stock'] < $deductNum) {
                echo "乐观锁：库存不足，当前库存：" . ($stockInfo['stock'] ?? 0);
                return false;
            }

            // 3. 扣减库存（通过 version 校验冲突）
            $stmt = $pdo->prepare("
                UPDATE product_stock 
                SET 
                    stock = stock - :deduct_num,
                    version = version + 1 -- 版本号自增，标记数据已修改
                WHERE 
                    product_id = :product_id 
                    AND version = :version; -- 关键：仅版本号匹配时才执行
            ");
            $stmt->execute([
                ':deduct_num' => $deductNum,
                ':product_id' => $productId,
                ':version' => $stockInfo['version'] // 用查询到的版本号做条件
            ]);

            // 4. 判断是否修改成功（受影响行数 > 0 表示无冲突）
            $affectedRows = $stmt->rowCount();
            if ($affectedRows > 0) {
                echo "乐观锁：库存扣减成功！剩余库存：" . ($stockInfo['stock'] - $deductNum) . "，当前版本：" . ($stockInfo['version'] + 1);
                return true;
            } else {
                // 5. 冲突：版本号不匹配，重试
                $retryCount++;
                echo "乐观锁：第 {$retryCount} 次冲突重试...\n";
                usleep(100000); // 暂停 0.1 秒，减少重试压力
            }

        } catch (PDOException $e) {
            echo "乐观锁：库存扣减失败：" . $e->getMessage();
            return false;
        }
    }

    // 重试次数用尽
    echo "乐观锁：重试 {$maxRetry} 次后仍冲突，扣减失败";
    return false;
}

// 调用
deductStockOptimistic($pdo, 1001, 1, 3);
```


### 三、关键注意事项
1. **悲观锁的锁范围**：  
   - 必须使用 `InnoDB` 引擎，`MyISAM` 不支持行锁；  
   - 查询条件需包含索引列（如 `product_id`），否则行锁会退化为表锁，阻塞所有商品的修改。

2. **乐观锁的重试机制**：  
   - 重试次数需合理（建议 3-5 次），避免无限重试导致资源浪费；  
   - 重试间隔（`usleep`）可减少并发冲突，尤其高并发场景。

3. **死锁风险**：  
   - 悲观锁可能产生死锁（如两个事务互相持有对方需要的锁），需避免“交叉加锁”（如事务 1 先锁 A 再锁 B，事务 2 先锁 B 再锁 A）；  
   - 乐观锁无死锁风险，因不主动加锁。


### 总结
悲观锁适合高冲突场景（代码简单、一致性强），乐观锁适合低冲突场景（并发高、无死锁）。实际开发中，需根据“冲突概率”和“并发量”选择——秒杀等极端场景用悲观锁，普通库存管理用乐观锁，且优先基于 `InnoDB` 行锁实现。
</details>


## 9. 当 MySQL 单表数据量过大（如超过 1000 万行）时，查询性能会显著下降，此时分库分表是常用解决方案。分库分表的核心思路是什么？常见的分表方案（水平分表、垂直分表）有什么区别？如何落地（举例说明）？

<details>
<summary>答案与解析</summary>

- 思路要点：  
  1. **分库分表的核心目的**：解决“单表数据量超限导致的性能瓶颈”——当单表数据量超过 1000 万行时，B+ 树索引深度增加，磁盘 IO 次数增多，查询变慢；  
  2. **核心思路**：“数据拆分”——将大表拆分为多个小表，大库拆分为多个小库，使每个小表/小库的数据量控制在 100-500 万行（最优性能区间）；  
  3. **方案差异**：垂直拆分（按“字段”拆分，解决字段过多或大字段问题）、水平拆分（按“数据行”拆分，解决行数过多问题），需明确两种方案的适用场景和落地方式。


### 一、分库分表核心概念与前提
#### 1. 需分库分表的信号
- 单表数据量持续增长，超过 1000 万行；  
- 常规优化（索引、SQL 优化）后，查询仍超过 1 秒（如分页查询 `LIMIT 1000000, 10`）；  
- 写入操作（INSERT/UPDATE）频繁，导致行锁等待时间长，并发性能低。

#### 2. 拆分原则
- **不拆分优先**：能通过索引、SQL 优化解决的，不优先分库分表（拆分增加系统复杂度）；  
- **水平拆分优先**：单表行数过多用水平拆分，字段过多或有大字段用垂直拆分；  
- **一致性哈希**：水平拆分时，用哈希算法（如用户 ID 取模）确保数据均匀分布，避免“数据倾斜”。


### 二、常见分表方案对比（水平分表 vs 垂直分表）
| 拆分类型   | 核心逻辑                | 拆分依据                          | 优点                                  | 缺点                                  | 适用场景                          |
|------------|-------------------------|-----------------------------------|---------------------------------------|---------------------------------------|-----------------------------------|
| **垂直分表** | 按“字段”拆分，将表拆分为“高频字段表”和“低频字段表” | 字段访问频率（高频/低频）、字段大小（普通字段/大字段） | 减少数据传输量（仅查高频字段）；大字段（如 TEXT）单独存储，提升查询速度 | 多表关联查询增多（需 JOIN 或应用层拼接）；事务复杂度增加 | 表字段过多（如 50+ 字段）；含大字段（TEXT/BLOB）；高频字段与低频字段分离场景 |
| **水平分表** | 按“数据行”拆分，将表拆分为多个结构相同的小表 | 时间范围（如按月份）、ID 范围（如 ID 1-100万）、哈希（如用户 ID 取模） | 单表数据量小，查询/写入性能高；扩展性强（可按需增加表） | 跨表查询复杂（如查询全量数据需 union）；需路由规则定位表 | 单表行数超 1000 万；数据有明显拆分维度（时间、ID） |


### 三、分表方案落地示例
#### 1. 垂直分表示例（用户表拆分）
**场景**：用户表 `users` 包含 30+ 字段，其中 `username`、`email`、`phone` 是高频查询字段，`avatar`（BLOB）、`intro`（TEXT）是低频大字段，查询时频繁读取所有字段导致性能低。

##### （1）拆分前表结构
```sql
CREATE TABLE `users` (
  `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `username` VARCHAR(50) NOT NULL,
  `email` VARCHAR(100) NOT NULL UNIQUE,
  `phone` VARCHAR(20) NOT NULL UNIQUE,
  `avatar` BLOB COMMENT '头像（大字段）',
  `intro` TEXT COMMENT '个人简介（大字段）',
  `created_at` DATETIME NOT NULL,
  `updated_at` DATETIME NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

##### （2）垂直拆分后表结构
- **高频字段表（`users_base`）**：存储高频查询字段，优先查询；  
  ```sql
  CREATE TABLE `users_base` (
    `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
    `username` VARCHAR(50) NOT NULL,
    `email` VARCHAR(100) NOT NULL UNIQUE,
    `phone` VARCHAR(20) NOT NULL UNIQUE,
    `created_at` DATETIME NOT NULL,
    `updated_at` DATETIME NOT NULL
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
  ```
- **低频大字段表（`users_extend`）**：存储低频大字段，按需查询；  
  ```sql
  CREATE TABLE `users_extend` (
    `user_id` INT NOT NULL PRIMARY KEY, -- 与 users_base.id 关联
    `avatar` BLOB COMMENT '头像',
    `intro` TEXT COMMENT '个人简介',
    `updated_at` DATETIME NOT NULL,
    FOREIGN KEY (`user_id`) REFERENCES `users_base`(`id`) ON DELETE CASCADE
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
  ```

##### （3）PHP 代码查询示例
```php
<?php
// 1. 高频查询（仅需用户名、邮箱）：直接查 users_base（性能高）
$stmt = $pdo->prepare("SELECT username, email FROM users_base WHERE id = :id");
$stmt->execute([':id' => 1001]);
$userBase = $stmt->fetch();

// 2. 需获取头像（低频场景）：关联查询 users_extend
$stmt = $pdo->prepare("
    SELECT b.username, e.avatar 
    FROM users_base b
    LEFT JOIN users_extend e ON b.id = e.user_id
    WHERE b.id = :id
");
$stmt->execute([':id' => 1001]);
$userFull = $stmt->fetch();
```


#### 2. 水平分表示例（订单表拆分）
**场景**：订单表 `orders` 数据量达 2000 万行，按“用户 ID 取模”拆分为 4 个表（`orders_0`、`orders_1`、`orders_2`、`orders_3`），每个表约 500 万行。

##### （1）拆分规则
- 拆分维度：用户 ID（`user_id`）；  
- 拆分算法：`表索引 = user_id % 4`（4 个表，索引 0-3）；  
- 示例：用户 ID=1001 → 1001%4=1 → 数据存入 `orders_1`；用户 ID=1002 → 1002%4=2 → 存入 `orders_2`。

##### （2）拆分后表结构（所有分表结构相同）
```sql
-- 创建 orders_0（其他 orders_1/2/3 结构相同）
CREATE TABLE `orders_0` (
  `id` BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `user_id` INT NOT NULL,
  `order_no` VARCHAR(50) NOT NULL UNIQUE,
  `amount` DECIMAL(10,2) NOT NULL,
  `status` TINYINT NOT NULL,
  `created_at` DATETIME NOT NULL,
  INDEX idx_user_id (`user_id`) -- 按 user_id 建索引，匹配拆分维度
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

-- 同理创建 orders_1、orders_2、orders_3
CREATE TABLE `orders_1` LIKE `orders_0`;
CREATE TABLE `orders_2` LIKE `orders_0`;
CREATE TABLE `orders_3` LIKE `orders_0`;
```

##### （3）PHP 代码实现“路由”与操作
```php
<?php
/**
 * 水平分表路由：根据 user_id 确定目标表
 */
function getOrderTableName($userId, $tablePrefix = 'orders_', $tableCount = 4) {
    $tableIndex = $userId % $tableCount; // 取模获取表索引
    return $tablePrefix . $tableIndex;
}

/**
 * 1. 插入订单（根据 user_id 路由到对应表）
 */
function insertOrder($pdo, $orderData) {
    $tableName = getOrderTableName($orderData['user_id']);
    $stmt = $pdo->prepare("
        INSERT INTO {$tableName} (user_id, order_no, amount, status, created_at)
        VALUES (:user_id, :order_no, :amount, :status, :created_at)
    ");
    return $stmt->execute([
        ':user_id' => $orderData['user_id'],
        ':order_no' => $orderData['order_no'],
        ':amount' => $orderData['amount'],
        ':status' => $orderData['status'],
        ':created_at' => $orderData['created_at']
    ]);
}

/**
 * 2. 查询用户订单（仅查该用户对应的分表，性能高）
 */
function getOrdersByUserId($pdo, $userId) {
    $tableName = getOrderTableName($userId);
    $stmt = $pdo->prepare("
        SELECT * FROM {$tableName} 
        WHERE user_id = :user_id 
        ORDER BY created_at DESC
    ");
    $stmt->execute([':user_id' => $userId]);
    return $stmt->fetchAll();
}

/**
 * 3. 跨表查询（如统计所有表的订单总数，需 union 所有分表）
 */
function getTotalOrders($pdo, $tableCount = 4) {
    $unionSql = [];
    for ($i = 0; $i < $tableCount; $i++) {
        $unionSql[] = "SELECT COUNT(*) AS count FROM orders_{$i}";
    }
    $sql = implode(" UNION ALL ", $unionSql); // UNION ALL 比 UNION 快（不去重）
    
    $stmt = $pdo->query($sql);
    $total = 0;
    while ($row = $stmt->fetch()) {
        $total += $row['count'];
    }
    return $total;
}

// 调用示例
$orderData = [
    'user_id' => 1001,
    'order_no' => 'ORD' . date('YmdHis') . rand(1000, 9999),
    'amount' => 299.00,
    'status' => 1,
    'created_at' => date('Y-m-d H:i:s')
];
insertOrder($pdo, $orderData); // 插入到 orders_1
$orders = getOrdersByUserId($pdo, 1001); // 从 orders_1 查询
$total = getTotalOrders($pdo); // 统计所有 4 个表的订单总数
```


### 四、分库分表进阶：中间件方案
手动分表需在代码中处理路由逻辑，复杂度高，生产环境推荐使用分库分表中间件，如：
- **Sharding-JDBC**：Java 生态常用，可通过配置实现分库分表，透明化路由逻辑；  
- **MyCat**：基于 MySQL 协议的中间件，支持分库分表、读写分离，PHP 可直接作为 MySQL 客户端连接；  
- **ProxySQL**：轻量级中间件，支持分库分表和读写分离，适合中小规模场景。

**中间件优势**：无需修改业务代码，通过配置定义拆分规则（如“user_id 取模分表”），中间件自动完成路由、跨表查询等操作，降低系统复杂度。


### 总结
分库分表的核心是“拆分数据以控制单表规模”，垂直分表解决“字段过多/大字段”问题，水平分表解决“行数过多”问题。落地时需先明确拆分维度（如用户 ID、时间），小规模场景可手动实现路由，大规模场景推荐使用中间件。拆分后需注意“跨表查询优化”和“数据一致性维护”（如事务、同步）。
</details>


## 10. MySQL 的事务隔离级别有哪些？不同隔离级别下会出现哪些并发问题（脏读、不可重复读、幻读）？MySQL 默认的隔离级别是什么？如何通过 SQL 和 PHP 代码设置隔离级别？

<details>
<summary>答案与解析</summary>

- 思路要点：  
  1. **并发问题前置**：先明确“脏读”“不可重复读”“幻读”的定义——这是事务隔离级别要解决的核心问题；  
  2. **隔离级别对应关系**：四个隔离级别（读未提交、读已提交、可重复读、串行化）与三个并发问题的“解决矩阵”，需逐一说明每个级别能解决哪些问题；  
  3. **落地配置**：通过 MySQL 全局/会话级 SQL 设置隔离级别，结合 PDO 的 `setAttribute()` 或执行 `SET TRANSACTION` 语句在 PHP 中生效，明确 MySQL 默认级别（InnoDB 为可重复读）。


### 一、事务并发问题定义
在多事务并发执行时，若隔离级别过低，会出现以下三类问题：
| 并发问题         | 定义                                                                 | 示例场景                                                                 |
|------------------|----------------------------------------------------------------------|--------------------------------------------------------------------------|
| **脏读（Dirty Read）** | 事务 A 读取了事务 B 未提交的修改数据，若 B 回滚，A 读取的就是“无效数据”。 | 事务 1 给用户 A 加 100 元（未提交），事务 2 读取 A 余额为 600 元；事务 1 回滚后，事务 2 读取的 600 元是脏数据。 |
| **不可重复读（Non-Repeatable Read）** | 事务 A 多次读取同一数据，事务 B 在 A 读取期间修改并提交了该数据，导致 A 多次读取结果不一致。 | 事务 1 第一次读用户 A 余额为 500 元；事务 2 给 A 加 100 元并提交；事务 1 再次读 A 余额为 600 元，两次结果不同。 |
| **幻读（Phantom Read）** | 事务 A 按条件查询数据，事务 B 在 A 查询期间插入/删除了符合条件的行，导致 A 再次查询时行数变化（“幻觉”新增/消失）。 | 事务 1 查询“余额 > 1000 元的用户”，返回 2 人；事务 2 新增 1 个余额 1500 元的用户并提交；事务 1 再次查询，返回 3 人。 |


### 二、MySQL 事务隔离级别与并发问题解决矩阵
MySQL 支持四个标准隔离级别（由低到高），隔离级别越高，并发性能越低，但数据一致性越强：

| 隔离级别                 | 英文全称                | 脏读 | 不可重复读 | 幻读 | 并发性能 | 核心原理                                                                 |
|--------------------------|-------------------------|------|------------|------|----------|--------------------------------------------------------------------------|
| **读未提交**             | Read Uncommitted        | 允许 | 允许       | 允许 | 最高     | 事务可读取其他事务未提交的修改，无任何隔离措施。                         |
| **读已提交**             | Read Committed          | 禁止 | 允许       | 允许 | 较高     | 事务仅读取其他事务已提交的修改，解决脏读；但同一事务内多次读可能不一致。 |
| **可重复读（默认）**     | Repeatable Read         | 禁止 | 禁止       | 基本禁止 | 中等     | 事务内多次读取同一数据结果一致（基于 MVCC 多版本控制），解决不可重复读；InnoDB 通过间隙锁解决幻读。 |
| **串行化**               | Serializable            | 禁止 | 禁止       | 禁止 | 最低     | 事务串行执行（一个执行完再执行下一个），完全隔离，无并发问题。           |

> 注：MySQL InnoDB 引擎在“可重复读”级别下，通过 **间隙锁（Next-Key Lock）** 防止幻读，这是 InnoDB 对标准隔离级别的增强（标准可重复读不解决幻读）。


### 三、隔离级别设置与 PHP 实现
#### 1. MySQL 中设置隔离级别
MySQL 支持 **全局级别**（所有新会话生效）和 **会话级别**（仅当前会话生效），语法如下：
```sql
-- 1. 查看当前隔离级别（MySQL 8.0+）
SELECT @@transaction_isolation; -- 会话级别
SELECT @@global.transaction_isolation; -- 全局级别

-- 2. 设置会话级别隔离级别（当前连接生效）
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 3. 设置全局级别隔离级别（需重启会话生效）
SET GLOBAL TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- 4. 可选：通过 my.cnf 配置默认隔离级别（永久生效）
[mysqld]
transaction-isolation = REPEATABLE-READ
```


#### 2. PHP PDO 中设置隔离级别
PDO 支持两种方式设置隔离级别：通过 `PDO::setAttribute()` 或执行 `SET TRANSACTION` 语句。

##### 方式 1：通过 `setAttribute()` 设置（推荐）
```php
<?php
$pdo = new PDO(
    'mysql:host=localhost;dbname=test;charset=utf8',
    'root',
    'your_password',
    [PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION]
);

// 设置当前会话隔离级别为“读已提交”
$pdo->setAttribute(
    PDO::ATTR_DEFAULT_FETCH_MODE, 
    PDO::FETCH_ASSOC
);
$pdo->setAttribute(
    PDO::ATTR_AUTOCOMMIT, 
    false // 关闭自动提交，开启事务前需设置
);

// 设置隔离级别（PDO 常量对应隔离级别）
$isolationLevel = PDO::TRANSACTION_READ_COMMITTED; 
// 可选常量：
// PDO::TRANSACTION_READ_UNCOMMITTED（读未提交）
// PDO::TRANSACTION_READ_COMMITTED（读已提交）
// PDO::TRANSACTION_REPEATABLE_READ（可重复读，默认）
// PDO::TRANSACTION_SERIALIZABLE（串行化）

$pdo->beginTransaction();
// 执行事务操作...
$pdo->commit();
```

##### 方式 2：执行 `SET TRANSACTION` 语句
```php
<?php
$pdo = new PDO(
    'mysql:host=localhost;dbname=test;charset=utf8',
    'root',
    'your_password',
    [PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION]
);

// 开启事务前执行 SQL 设置隔离级别
$pdo->exec("SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ");

$pdo->beginTransaction();
// 示例：验证“可重复读”——事务内多次读结果一致
$stmt1 = $pdo->query("SELECT balance FROM user_balance WHERE user_id = 1");
$balance1 = $stmt1->fetch()['balance'];
echo "第一次读取余额：{$balance1}\n";

// 模拟其他事务修改并提交（此处可手动在另一个客户端执行 UPDATE）
sleep(10); // 暂停 10 秒，期间在其他客户端执行：UPDATE user_balance SET balance = 600 WHERE user_id = 1;

$stmt2 = $pdo->query("SELECT balance FROM user_balance WHERE user_id = 1");
$balance2 = $stmt2->fetch()['balance'];
echo "第二次读取余额：{$balance2}\n"; // 可重复读级别下，balance1 == balance2

$pdo->commit();
```


### 四、关键说明与最佳实践
1. **MySQL 默认隔离级别**：  
   InnoDB 引擎默认使用 **可重复读（Repeatable Read）**，兼顾“一致性”和“并发性能”，适合大多数业务场景（如电商订单、用户管理）。

2. **隔离级别选择建议**：  
   - 普通业务场景：用默认的“可重复读”；  
   - 对一致性要求低、并发高（如统计报表）：用“读已提交”；  
   - 对一致性要求极高（如金融转账）：用“串行化”（需接受低并发）；  
   - 禁止使用“读未提交”（脏读风险高，无实际业务价值）。

3. **MVCC 与隔离级别的关系**：  
   InnoDB 通过 **MVCC（多版本并发控制）** 实现“读已提交”和“可重复读”——每个事务读取数据的“快照版本”，避免加锁阻塞读操作（非锁定读），这是 InnoDB 并发性能高的核心原因。


### 总结
事务隔离级别是平衡“数据一致性”和“并发性能”的关键，MySQL InnoDB 默认用“可重复读”解决脏读和不可重复读，通过间隙锁解决幻读。PHP 中可通过 PDO 常量或 SQL 语句设置隔离级别，实际开发中需根据业务一致性要求选择，优先使用默认级别，避免过度隔离导致性能下降。
</details>


## 11. 虽然创建了索引，但在某些查询场景下索引会失效，导致全表扫描。常见的索引失效场景有哪些？如何通过 EXPLAIN 分析索引是否失效？如何避免索引失效？

<details>
<summary>答案与解析</summary>

- 思路要点：  
  1. **索引失效的核心原因**：MySQL 优化器判断“使用索引的成本高于全表扫描”，或查询条件不符合索引使用规则（如函数操作索引列、数据类型不匹配）；  
  2. **场景归类**：按“索引列操作”“查询条件逻辑”“数据特性”三个维度归类常见失效场景，每个场景需结合 SQL 示例说明；  
  3. **分析与避免**：通过 `EXPLAIN` 关键字段（`type`=ALL、`key`=NULL）判断失效，针对性给出避免方案（如避免函数操作、使用正确数据类型）。


### 一、常见索引失效场景与示例
假设存在表 `user`，已创建索引：`INDEX idx_age_name (age, name)`（联合索引）、`INDEX idx_email (email)`（普通索引），表数据量 100 万行。

#### 1. 索引列上执行函数操作（最常见）
**失效原因**：函数操作改变了索引列的原始值，MySQL 无法利用索引的有序性定位数据，只能全表扫描。  
```sql
-- 失效场景：对 age 索引列使用函数（YEAR()）
EXPLAIN SELECT * FROM user WHERE YEAR(birthday) = 1990; 
-- 若 birthday 有索引，函数操作后索引失效，type=ALL（全表扫描）

-- 优化方案：避免函数操作，改为对常量使用函数（索引有效）
EXPLAIN SELECT * FROM user WHERE birthday BETWEEN '1990-01-01' AND '1990-12-31';
```


#### 2. 使用“不等于”（!=、<>）或“NOT IN”“IS NOT NULL”
**失效原因**：“不等于”条件会导致 MySQL 认为匹配行数较多，使用索引的成本高于全表扫描（尤其低区分度列）。  
```sql
-- 失效场景 1：!= 操作
EXPLAIN SELECT * FROM user WHERE age != 25; -- age 有索引，失效，type=ALL

-- 失效场景 2：NOT IN 操作
EXPLAIN SELECT * FROM user WHERE age NOT IN (20, 25, 30); -- 索引失效

-- 优化方案：若必须用不等于，可拆分为多个等于条件（适合值少的场景）
EXPLAIN SELECT * FROM user WHERE age = 18 OR age = 19 OR age = 21; -- 索引有效
```


#### 3. 字符串条件未加引号（数据类型不匹配）
**失效原因**：索引列是字符串类型（如 `VARCHAR`），查询条件用数字（无引号），MySQL 会自动进行“类型转换”（如 `email = 123` 转为 `CAST(email AS UNSIGNED) = 123`），等价于函数操作，导致索引失效。  
```sql
-- 失效场景：email 是 VARCHAR 类型，条件用数字（无引号）
EXPLAIN SELECT * FROM user WHERE email = 123456; -- idx_email 索引失效，type=ALL

-- 优化方案：字符串条件加引号（匹配数据类型）
EXPLAIN SELECT * FROM user WHERE email = '123456'; -- 索引有效，type=ref
```


#### 4. 联合索引不满足“最左前缀原则”
**失效原因**：联合索引（如 `(age, name)`）的底层按“左到右”顺序排序，查询条件必须包含最左侧列，且连续匹配，否则无法利用索引。  
```sql
-- 联合索引：idx_age_name (age, name)

-- 有效场景：包含最左列 age
EXPLAIN SELECT * FROM user WHERE age = 25; -- 有效，type=ref
EXPLAIN SELECT * FROM user WHERE age = 25 AND name = 'zhangsan'; -- 有效，type=ref

-- 失效场景：不包含最左列 age，或跳过中间列
EXPLAIN SELECT * FROM user WHERE name = 'zhangsan'; -- 无 age，失效，type=ALL
EXPLAIN SELECT * FROM user WHERE age > 25 AND name = 'zhangsan'; -- age 是范围查询，name 无法利用索引
```


#### 5. LIKE 以“%”开头（模糊查询）
**失效原因**：`LIKE '%xxx'` 是“前缀模糊”，MySQL 无法通过索引的有序性定位前缀不确定的数据，只能全表扫描；`LIKE 'xxx%'` 是“后缀模糊”，可利用索引。  
```sql
-- 失效场景：LIKE 以 % 开头
EXPLAIN SELECT * FROM user WHERE name LIKE '%zhang'; -- 索引失效，type=ALL

-- 有效场景：LIKE 以 % 结尾（前缀确定）
EXPLAIN SELECT * FROM user WHERE name LIKE 'zhang%'; -- 索引有效，type=range
```


#### 6. OR 连接的条件中包含非索引列
**失效原因**：OR 连接的多个条件中，若有一个列无索引，MySQL 会认为“使用索引只能过滤部分数据，剩余数据仍需全表扫描”，索性放弃索引，直接全表扫描。  
```sql
-- 失效场景：OR 连接非索引列 gender（gender 无索引）
EXPLAIN SELECT * FROM user WHERE age = 25 OR gender = 'male'; -- age 有索引，但 gender 无，整体失效

-- 优化方案 1：给非索引列加索引（gender 加索引后，索引有效）
CREATE INDEX idx_gender ON user (gender);
EXPLAIN SELECT * FROM user WHERE age = 25 OR gender = 'male'; -- 有效

-- 优化方案 2：拆分为两个独立查询，用 UNION ALL 合并（避免 OR）
EXPLAIN 
SELECT * FROM user WHERE age = 25
UNION ALL
SELECT * FROM user WHERE gender = 'male'; -- 两个查询分别用索引，性能更优
```


### 二、通过 EXPLAIN 分析索引是否失效
`EXPLAIN` 是 MySQL 执行计划分析工具，通过以下关键字段判断索引是否失效：

| 字段名   | 关键值含义                                                                 | 索引失效判断依据                          |
|----------|--------------------------------------------------------------------------|-------------------------------------------|
| `type`   | 访问类型，从优到差：`const` > `eq_ref` > `ref` > `range` > `index` > `ALL` | 若 `type` = `ALL`，表示全表扫描，索引失效；`type` = `range`/`ref` 等表示索引有效。 |
| `key`    | MySQL 实际使用的索引名称                                                 | 若 `key` = `NULL`，表示未使用任何索引，索引失效；若显示索引名（如 `idx_age`），表示索引有效。 |
| `rows`   | MySQL 预估扫描的行数                                                     | 索引失效时，`rows` 接近表总数据量；索引有效时，`rows` 远小于总数据量。 |
| `Extra`  | 额外信息                                                                 | 若出现 `Using filesort`（排序未用索引）、`Using temporary`（临时表），可能伴随索引失效。 |

#### 示例：分析索引失效
```sql
-- 执行 EXPLAIN 分析失效场景
EXPLAIN SELECT * FROM user WHERE age != 25;

-- 输出关键结果：
-- type: ALL（全表扫描）
-- key: NULL（未使用索引）
-- rows: 1000000（扫描全表 100 万行）
-- 结论：索引失效
```


### 三、避免索引失效的最佳实践
1. **索引列避免函数操作**：将函数操作转移到常量端（如 `birthday BETWEEN '1990-01-01' AND '1990-12-31'` 而非 `YEAR(birthday) = 1990`）；  
2. **匹配数据类型**：字符串条件加引号，数字条件不加引号，避免类型转换；  
3. **联合索引遵循最左前缀**：查询条件包含联合索引最左侧列，且按顺序使用（如 `(a,b,c)` 索引，优先用 `a`、`a AND b`、`a AND b AND c`）；  
4. **模糊查询用后缀匹配**：尽量使用 `LIKE 'xxx%'`，避免 `LIKE '%xxx'`；  
5. **OR 条件全加索引**：OR 连接的所有列都需有索引，或拆分为 `UNION ALL` 查询；  
6. **定期维护索引**：删除冗余索引（如 `(a,b)` 索引已存在，无需再建 `(a)` 索引），避免 MySQL 优化器选择低效索引；  
7. **小表无需索引**：数据量 < 1 万行的表，全表扫描速度快于索引查询（索引有额外 IO 开销），无需建索引。


### 总结
索引失效的核心是“查询条件不符合索引使用规则”或“MySQL 认为使用索引成本更高”。通过 `EXPLAIN` 可快速定位失效场景，实际开发中需避免函数操作索引列、遵循最左前缀原则、匹配数据类型，确保索引高效利用。
</details>


## 12. MySQL 中的行锁和表锁有什么区别？哪些场景会触发行锁或表锁？如何避免行锁导致的死锁问题？

<details>
<summary>答案与解析</summary>

- 思路要点：  
  1. **锁粒度差异**：行锁（锁定单行数据）vs 表锁（锁定整个表），粒度决定并发性能——行锁粒度小，并发高；表锁粒度大，并发低；  
  2. **触发机制**：InnoDB 支持行锁和表锁，MyISAM 仅支持表锁；行锁需基于索引触发，无索引则退化为表锁；  
  3. **死锁解决**：明确死锁产生的条件（资源竞争、循环等待），通过“固定加锁顺序”“超时设置”“避免长事务”等方式避免。


### 一、行锁与表锁核心区别对比
| 对比维度                | 行锁（Row Lock）                          | 表锁（Table Lock）                          |
|-------------------------|-------------------------------------------|-------------------------------------------|
| **锁定粒度**            | 单行数据（粒度小）                        | 整个表（粒度大）                          |
| **支持引擎**            | 仅 InnoDB（MyISAM 不支持）                | InnoDB、MyISAM 均支持                      |
| **并发性能**            | 高（仅阻塞锁定行，其他行可正常读写）      | 低（锁定全表，读写操作均阻塞）              |
| **触发条件**            | 基于索引的写操作（UPDATE/DELETE/INSERT）；SELECT ... FOR UPDATE | MyISAM 的所有写操作；InnoDB 无索引的写操作；LOCK TABLES 显式加锁 |
| **锁冲突概率**          | 低（仅同一行数据竞争）                    | 高（全表数据竞争）                        |
| **死锁风险**            | 有（多事务交叉加锁）                      | 无（锁定全表，无交叉竞争）                  |
| **适用场景**            | 高并发写场景（如电商库存、订单）          | 读多写少场景（如静态博客、报表）            |


### 二、行锁与表锁的触发场景示例
假设存在两张表：`innodb_user`（InnoDB 引擎，`id` 为主键索引）、`myisam_news`（MyISAM 引擎）。

#### 1. 行锁触发场景（InnoDB）
**核心条件**：写操作的查询条件包含索引列，InnoDB 会锁定“满足条件的行”。
```sql
-- 1. 基于主键索引的行锁（锁定 id=1 的行）
UPDATE innodb_user SET name = 'lisi' WHERE id = 1; 
-- 其他事务修改 id=2 的行不受影响，修改 id=1 的行需等待锁释放

-- 2. 基于辅助索引的行锁（锁定 age=25 的所有行）
CREATE INDEX idx_age ON innodb_user (age);
UPDATE innodb_user SET name = 'wangwu' WHERE age = 25; 
-- 锁定所有 age=25 的行，其他 age 行可正常修改

-- 3. 显式行锁（SELECT ... FOR UPDATE）
START TRANSACTION;
SELECT * FROM innodb_user WHERE id = 1 FOR UPDATE; -- 锁定 id=1 的行
COMMIT; -- 提交后释放锁
```

#### 2. 表锁触发场景
**场景 1：MyISAM 引擎的写操作**  
MyISAM 仅支持表锁，所有写操作都会锁定全表：
```sql
-- MyISAM 表：执行 UPDATE 时锁定整个 myisam_news 表
UPDATE myisam_news SET title = 'new title' WHERE id = 1; 
-- 其他事务读写 myisam_news 表均需等待锁释放
```

**场景 2：InnoDB 无索引的写操作**  
InnoDB 行锁依赖索引，若查询条件无索引，会退化为表锁：
```sql
-- innodb_user 表的 gender 无索引，UPDATE 会锁定全表
UPDATE innodb_user SET name = 'zhaoliu' WHERE gender = 'male'; 
-- 其他事务修改任何行都需等待锁释放
```

**场景 3：显式表锁（LOCK TABLES）**  
通过 `LOCK TABLES` 显式锁定表，InnoDB 和 MyISAM 均支持：
```sql
-- 显式锁定 innodb_user 表为写锁（仅当前事务可写，其他事务读写均阻塞）
LOCK TABLES innodb_user WRITE;
UPDATE innodb_user SET name = 'qianqi' WHERE id = 1;
UNLOCK TABLES; -- 释放锁

-- 显式锁定 innodb_user 表为读锁（所有事务可读，仅当前事务可写）
LOCK TABLES innodb_user READ;
```


### 三、行锁导致的死锁与避免方案
#### 1. 死锁产生的条件
死锁是“两个或多个事务互相持有对方需要的锁，且都不释放”，例如：
```sql
-- 事务 1
START TRANSACTION;
UPDATE innodb_user SET name = 'a' WHERE id = 1; -- 锁定 id=1 的行
UPDATE innodb_user SET name = 'b' WHERE id = 2; -- 等待事务 2 释放 id=2 的锁

-- 事务 2（与事务 1 同时执行）
START TRANSACTION;
UPDATE innodb_user SET name = 'c' WHERE id = 2; -- 锁定 id=2 的行
UPDATE innodb_user SET name = 'd' WHERE id = 1; -- 等待事务 1 释放 id=1 的锁
```
此时两个事务互相等待，形成死锁，MySQL 会自动终止其中一个事务（回滚），释放锁。


#### 2. 避免死锁的最佳实践
1. **固定加锁顺序**：所有事务按“相同顺序”加锁，避免交叉加锁。例如，所有事务都先锁 `id=1`，再锁 `id=2`：
   ```sql
   -- 事务 1 和事务 2 均按 id 升序加锁
   UPDATE innodb_user SET name = 'a' WHERE id = 1;
   UPDATE innodb_user SET name = 'b' WHERE id = 2;
   ```

2. **缩短事务时长**：事务中仅包含必要的 SQL 操作，避免长时间持有锁（如避免在事务中调用外部接口、等待用户输入）：
   ```php
   // 坏示例：事务中包含外部接口调用（长时间阻塞）
   $pdo->beginTransaction();
   updateUser($pdo); // SQL 操作
   callExternalApi(); // 外部接口（可能耗时 10 秒+）
   $pdo->commit();

   // 好示例：先调用接口，再开启事务
   $apiResult = callExternalApi(); // 先完成外部操作
   $pdo->beginTransaction();
   updateUser($pdo); // 快速执行 SQL
   $pdo->commit();
   ```

3. **避免长事务**：将大事务拆分为多个小事务，减少锁持有时间。例如，批量更新 1000 条数据，拆分为 10 次批量更新（每次 100 条）：
   ```sql
   -- 拆分前（长事务，锁持有时间长）
   UPDATE innodb_user SET status = 1 WHERE id BETWEEN 1 AND 1000;

   -- 拆分后（小事务，快速释放锁）
   UPDATE innodb_user SET status = 1 WHERE id BETWEEN 1 AND 100;
   UPDATE innodb_user SET status = 1 WHERE id BETWEEN 101 AND 200;
   -- ... 共 10 次
   ```

4. **设置锁超时时间**：通过 `innodb_lock_wait_timeout` 设置锁等待超时（默认 50 秒），超时后自动回滚事务，避免无限等待：
   ```sql
   -- 会话级别设置超时时间为 10 秒
   SET SESSION innodb_lock_wait_timeout = 10;
   ```

5. **使用乐观锁替代行锁**：低冲突场景用乐观锁（版本号），避免主动加行锁，从根本上消除死锁风险（参考第 8 题乐观锁实现）。


### 总结
行锁和表锁的核心差异是粒度与并发性能，InnoDB 行锁需基于索引触发，无索引则退化为表锁。死锁的本质是“交叉加锁+循环等待”，通过固定加锁顺序、缩短事务时长、避免长事务等方式可有效避免。实际开发中，优先使用 InnoDB 行锁提升并发，同时通过乐观锁或超时机制降低死锁风险。
</details>

### 13. MySQL 中如何实现读写分离？读写分离的核心目的是什么？结合 PHP 代码说明如何在项目中适配读写分离（如主从复制+中间件）？

<details>
<summary>答案与解析</summary>

- 思路要点：  
  1. **读写分离核心目的**：解决“单库读写压力不均衡”问题——主库承担写操作（INSERT/UPDATE/DELETE），从库承担读操作（SELECT），通过分担读压力提升整体性能，同时实现数据备份（从库是主库的副本）；  
  2. **实现基础**：依赖 MySQL 主从复制（主库binlog同步到从库，保证数据一致性）；  
  3. **适配方式**：分“中间件方案”（如 MyCat、ProxySQL，透明化路由）和“代码层方案”（PHP 中区分读写操作，定向连接主/从库），重点说明代码层适配（中小项目常用）和中间件适配（大规模项目推荐）。


### 一、读写分离的核心原理与实现基础
#### 1. 核心目的
- **减轻主库压力**：写操作（如订单创建、库存扣减）仅在主库执行，读操作（如商品查询、用户信息查询）分散到多个从库，避免主库因读请求过多导致性能瓶颈；  
- **提高读性能**：从库可横向扩展（增加从库数量），应对高并发读场景（如电商大促商品详情页查询）；  
- **数据备份与容灾**：从库是主库的实时副本，主库故障时可切换从库为主库，减少数据丢失风险。

#### 2. 实现基础：MySQL 主从复制
主从复制是读写分离的前提，通过“binlog 日志同步”实现数据一致，流程如下：
1. **主库（Master）**：开启 binlog 日志，记录所有写操作；  
2. **从库（Slave）**：启动 IO 线程，读取主库的 binlog 日志到本地（relay log，中继日志）；  
3. **从库（Slave）**：启动 SQL 线程，执行中继日志中的 SQL 语句，同步主库数据。

**主从复制核心配置**（简化版）：
- 主库配置（my.cnf）：
  ```ini
  [mysqld]
  server-id = 1 # 唯一标识，主库>从库
  log_bin = /var/lib/mysql/mysql-bin # 开启binlog
  binlog_format = ROW # 推荐ROW格式，数据一致性高
  ```
- 从库配置（my.cnf）：
  ```ini
  [mysqld]
  server-id = 2 # 与主库不同
  relay-log = /var/lib/mysql/relay-bin # 中继日志
  read_only = 1 # 从库设为只读（避免直接写从库）
  ```
- 从库关联主库（SQL 命令）：
  ```sql
  CHANGE MASTER TO
  MASTER_HOST='主库IP',
  MASTER_USER='repl_user', # 主库创建的复制账号
  MASTER_PASSWORD='repl_pass',
  MASTER_LOG_FILE='mysql-bin.000001', # 主库当前binlog文件名
  MASTER_LOG_POS=156; # 主库当前binlog位置
  START SLAVE; # 启动从库复制
  ```


### 二、PHP 项目适配读写分离的两种方案
#### 1. 代码层适配（中小项目，简单直接）
**核心逻辑**：在 PHP 中区分“写操作”和“读操作”——写操作（INSERT/UPDATE/DELETE）连接主库，读操作（SELECT）连接从库（可多个从库轮询负载均衡）。

**PHP 代码实现（PDO 示例）**：
```php
<?php
/**
 * 读写分离适配类：区分主库（写）和从库（读）
 */
class DbReadWriteSplit {
    // 主库配置（写操作）
    private $masterConfig = [
        'host' => '192.168.1.100', // 主库IP
        'dbname' => 'test',
        'username' => 'write_user',
        'password' => 'write_pass',
        'charset' => 'utf8'
    ];

    // 从库配置（读操作，支持多个从库轮询）
    private $slaveConfigs = [
        [
            'host' => '192.168.1.101', // 从库1
            'dbname' => 'test',
            'username' => 'read_user',
            'password' => 'read_pass',
            'charset' => 'utf8'
        ],
        [
            'host' => '192.168.1.102', // 从库2
            'dbname' => 'test',
            'username' => 'read_user',
            'password' => 'read_pass',
            'charset' => 'utf8'
        ]
    ];

    // 连接主库（写操作）
    public function getMasterPdo() {
        $config = $this->masterConfig;
        $dsn = "mysql:host={$config['host']};dbname={$config['dbname']};charset={$config['charset']}";
        return new PDO(
            $dsn,
            $config['username'],
            $config['password'],
            [
                PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
                PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC
            ]
        );
    }

    // 连接从库（读操作，轮询选择从库）
    public function getSlavePdo() {
        // 简单轮询：根据当前时间戳取模选择从库
        $slaveCount = count($this->slaveConfigs);
        $index = time() % $slaveCount;
        $config = $this->slaveConfigs[$index];
        
        $dsn = "mysql:host={$config['host']};dbname={$config['dbname']};charset={$config['charset']}";
        return new PDO(
            $dsn,
            $config['username'],
            $config['password'],
            [
                PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
                PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC
            ]
        );
    }

    // 执行 SQL（自动区分读写）
    public function execute($sql, $params = []) {
        // 判断 SQL 类型：写操作（INSERT/UPDATE/DELETE）连接主库，否则连接从库
        $sqlType = strtoupper(trim(substr($sql, 0, 6)));
        $isWrite = in_array($sqlType, ['INSERT', 'UPDATE', 'DELETE']);
        
        $pdo = $isWrite ? $this->getMasterPdo() : $this->getSlavePdo();
        $stmt = $pdo->prepare($sql);
        $stmt->execute($params);
        
        // 写操作返回受影响行数，读操作返回结果集
        return $isWrite ? $stmt->rowCount() : $stmt->fetchAll();
    }
}

// 调用示例
$db = new DbReadWriteSplit();

// 1. 读操作（自动连接从库）
$users = $db->execute("SELECT * FROM users WHERE age > :age", [':age' => 18]);
print_r($users);

// 2. 写操作（自动连接主库）
$affectedRows = $db->execute(
    "UPDATE users SET username = :username WHERE id = :id",
    [':username' => 'new_user', ':id' => 1001]
);
echo "受影响行数：{$affectedRows}";
```


#### 2. 中间件适配（大规模项目，推荐）
**核心逻辑**：通过中间件（如 MyCat、ProxySQL）统一管理主从库连接，PHP 项目只需连接中间件，无需关心主从库地址——中间件自动将写操作路由到主库，读操作路由到从库（支持负载均衡、故障切换）。

**实现步骤**：
1. **部署中间件**：以 MyCat 为例，配置主从库映射（schema.xml）：
   ```xml
   <!-- MyCat schema.xml 核心配置 -->
   <schema name="test" checkSQLschema="false" sqlMaxLimit="100">
       <table name="users" dataNode="dn1" primaryKey="id"/>
   </schema>
   <dataNode name="dn1" dataHost="localhost1" database="test"/>
   <dataHost name="localhost1" maxCon="1000" minCon="10" balance="1">
       <!-- 主库（写） -->
       <writeHost host="hostM1" url="192.168.1.100:3306" user="write_user" password="write_pass">
           <!-- 从库（读） -->
           <readHost host="hostS1" url="192.168.1.101:3306" user="read_user" password="read_pass"/>
           <readHost host="hostS2" url="192.168.1.102:3306" user="read_user" password="read_pass"/>
       </writeHost>
   </dataHost>
   ```
2. **PHP 项目连接中间件**：中间件提供统一连接地址（如 MyCat 默认端口 8066），PHP 无需区分主从，直接连接中间件：
   ```php
   <?php
   // PHP 连接 MyCat 中间件（无需关心主从）
   $pdo = new PDO(
       'mysql:host=192.168.1.103;port=8066;dbname=test;charset=utf8', // MyCat 地址
       'mycat_user', // MyCat 账号
       'mycat_pass',
       [PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION]
   );

   // 读操作（MyCat 自动路由到从库）
   $stmt = $pdo->query("SELECT * FROM users WHERE age > 18");
   $users = $stmt->fetchAll();

   // 写操作（MyCat 自动路由到主库）
   $stmt = $pdo->prepare("UPDATE users SET username = :name WHERE id = :id");
   $stmt->execute([':name' => 'mycat_test', ':id' => 1001]);
   ```


### 三、读写分离的注意事项
1. **数据一致性问题**：主从复制存在延迟（毫秒到秒级），可能导致“写后读”不一致（如刚创建的订单在从库查不到）。解决方案：  
   - 核心业务（如支付结果查询）强制读主库；  
   - 从库选择“延迟最小”的节点（中间件支持延迟排序）；  
   - 用“最终一致性”思想，允许短暂延迟（如订单列表页提示“数据可能延迟10秒”）。

2. **从库故障处理**：中间件可自动剔除故障从库，代码层需添加从库连接重试逻辑（如连接从库失败时切换到其他从库或主库）。

3. **账号权限控制**：主库仅授予“写权限”给写账号，从库仅授予“读权限”给读账号，避免从库被误写。


### 总结
读写分离的核心是“主库写、从库读”，基于主从复制实现数据一致，适配方式分代码层（中小项目简单高效）和中间件（大规模项目易扩展）。PHP 项目中需注意数据一致性和故障切换，核心业务可强制读主库，非核心业务用从库分担压力。
</details>


## 14. 如何配置和使用 MySQL 的慢查询日志？如何通过慢查询日志定位性能问题（举例说明）？如何优化慢查询日志的性能影响？

<details>
<summary>答案与解析</summary>

- 思路要点：  
  1. **慢查询日志核心作用**：记录“执行时间超过阈值”的 SQL 语句，是定位 MySQL 性能瓶颈的核心工具；  
  2. **配置维度**：需明确关键参数（`slow_query_log` 开启开关、`long_query_time` 时间阈值、`slow_query_log_file` 日志路径），区分全局/会话配置；  
  3. **分析方法**：结合工具（`mysqldumpslow`、`pt-query-digest`）和 `EXPLAIN`，从日志中提取慢查询并定位问题（如全表扫描、无索引）；  
  4. **性能优化**：避免日志过大影响磁盘 IO，通过“限制日志大小”“定期归档”“只记录有效慢查询”减少性能损耗。


### 一、慢查询日志的配置（MySQL 5.7+）
慢查询日志的配置通过 MySQL 系统变量实现，支持 **全局配置**（重启生效）和 **会话配置**（当前连接生效）。

#### 1. 关键配置参数
| 参数名                | 作用说明                                                                 | 默认值       | 推荐配置                |
|-----------------------|--------------------------------------------------------------------------|--------------|-------------------------|
| `slow_query_log`      | 是否开启慢查询日志（1=开启，0=关闭）                                      | 0（关闭）    | 1（生产环境建议开启）    |
| `long_query_time`     | 慢查询时间阈值（单位：秒），执行时间超过该值的 SQL 会被记录                | 10.0 秒      | 1-2 秒（根据业务调整）   |
| `slow_query_log_file` | 慢查询日志文件路径（需 MySQL 有写入权限）                                  | 无（需指定） | /var/log/mysql/slow.log  |
| `log_queries_not_using_indexes` | 是否记录“未使用索引”的 SQL（即使执行时间未超阈值）                        | 0（不记录）  | 1（方便定位无索引查询）  |
| `log_throttle_queries_not_using_indexes` | 每秒最多记录多少条“未使用索引”的 SQL（防止日志爆炸）                    | 0（无限制）  | 10（避免日志过大）       |


#### 2. 配置步骤（以 Linux 为例）
##### （1）临时配置（会话/全局，重启失效）
```sql
-- 1. 查看当前配置
SHOW VARIABLES LIKE '%slow_query%';
SHOW VARIABLES LIKE 'long_query_time';

-- 2. 会话级别开启（仅当前连接生效）
SET SESSION slow_query_log = 1;
SET SESSION long_query_time = 1; -- 阈值设为 1 秒
SET SESSION slow_query_log_file = '/var/log/mysql/slow.log';

-- 3. 全局级别开启（所有新连接生效，重启 MySQL 后失效）
SET GLOBAL slow_query_log = 1;
SET GLOBAL long_query_time = 1;
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';
```

##### （2）永久配置（my.cnf，重启生效）
```ini
[mysqld]
# 慢查询日志配置
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1
log_queries_not_using_indexes = 1
log_throttle_queries_not_using_indexes = 10

# 注意：确保 MySQL 对日志目录有写入权限
# 执行 chown mysql:mysql /var/log/mysql 赋予权限
```

##### （3）验证配置是否生效
```sql
-- 执行一条慢查询（如睡眠 2 秒）
SELECT SLEEP(2);

-- 查看日志文件是否有记录
cat /var/log/mysql/slow.log
```


### 二、慢查询日志的分析与问题定位
#### 1. 慢查询日志的格式解读
日志文件中每条记录包含“执行时间、执行用户、SQL 语句”等关键信息，示例：
```
# Time: 2024-05-20T10:30:00.123456Z
# User@Host: root[root] @ localhost []  Id: 12345
# Query_time: 2.500000  Lock_time: 0.000000  Rows_sent: 1000  Rows_examined: 100000
SET timestamp=1716234600;
SELECT * FROM users WHERE age > 25 ORDER BY created_at DESC;
```
关键字段解读：
- `Query_time`: SQL 执行时间（2.5 秒，超过阈值 1 秒，被记录）；  
- `Lock_time`: 锁等待时间（0 秒，无锁等待）；  
- `Rows_sent`: 返回给客户端的行数（1000 行）；  
- `Rows_examined`: 数据库扫描的行数（100000 行，全表扫描，问题所在）；  
- 最后一行：具体的慢查询 SQL。


#### 2. 分析工具与示例
##### （1）常用工具
- **mysqldumpslow**：MySQL 自带工具，用于统计慢查询（按执行次数、时间排序）；  
- **pt-query-digest**：Percona 工具集的核心工具，支持更详细的分析（如平均执行时间、锁时间占比）；  
- **MySQL Workbench**：可视化工具，可导入日志并生成分析报告。

##### （2）示例：定位“全表扫描”问题
1. **用 mysqldumpslow 查看高频慢查询**：
   ```bash
   # 按执行次数排序，显示前 10 条慢查询
   mysqldumpslow -s c -t 10 /var/log/mysql/slow.log
   ```
   输出可能包含：
   ```
   Count: 50  Time=2.50s (125s)  Lock=0.00s (0s)  Rows=1000.0 (50000), root[root]@localhost
     SELECT * FROM users WHERE age > N ORDER BY created_at DESC
   ```
   说明：该 SQL 执行了 50 次，总耗时 125 秒，每次平均 2.5 秒。

2. **用 EXPLAIN 分析 SQL 执行计划**：
   ```sql
   EXPLAIN SELECT * FROM users WHERE age > 25 ORDER BY created_at DESC;
   ```
   输出关键结果：
   - `type: ALL`（全表扫描）；  
   - `key: NULL`（未使用索引）；  
   - `Rows: 100000`（扫描 10 万行）。

3. **定位问题与优化**：  
   问题：`age` 和 `created_at` 无索引，导致全表扫描和文件排序（`Using filesort`）。  
   优化：创建联合索引 `idx_age_created`（匹配 WHERE 和 ORDER BY）：
   ```sql
   CREATE INDEX idx_age_created ON users (age, created_at);
   ```
   优化后再次执行 EXPLAIN：
   - `type: range`（索引范围查询）；  
   - `key: idx_age_created`（使用索引）；  
   - `Rows: 10000`（扫描行数减少到 1 万行）；  
   - `Extra: Using index condition`（使用索引条件过滤，无文件排序）。


#### 3. 分析“未使用索引”的慢查询
若开启 `log_queries_not_using_indexes = 1`，日志会记录“未用索引但执行快”的 SQL（如 `SELECT * FROM users WHERE gender = 'male'`），这类 SQL 虽当前快，但数据量增长后会变慢。  
优化方案：给 `gender` 加索引（若区分度低，可改用“覆盖索引”或“分区表”）。


### 三、优化慢查询日志的性能影响
慢查询日志会产生磁盘 IO 和 CPU 开销，尤其高并发场景，需通过以下方式减少影响：

1. **合理设置阈值**：避免将 `long_query_time` 设得过小（如 < 0.1 秒），否则会记录大量“非慢查询”，导致日志爆炸。建议生产环境设为 1-2 秒。

2. **限制日志大小与归档**：  
   - 用 `log_rotate`（Linux 工具）定期归档日志（如每天归档一次）；  
   - 配置 `max_slowlog_size`（部分版本支持）限制单个日志文件大小，满后自动切割。

3. **只记录关键信息**：  
   - 关闭 `log_queries_not_using_indexes`（仅在排查索引问题时开启）；  
   - 用 `log_slow_admin_statements` 控制是否记录“管理员语句”（如 `ALTER TABLE`），避免无关日志。

4. **使用内存日志（临时排查）**：  
   临时排查时，可将日志路径设为内存文件（如 `/dev/shm/slow.log`），避免磁盘 IO 开销，排查完成后关闭。

5. **生产环境分时段开启**：  
   若担心性能影响，可在“业务低峰期”（如凌晨 2-4 点）开启日志，排查问题后关闭，避免高峰时段开销。


### 总结
慢查询日志是 MySQL 性能优化的“指南针”，通过配置 `slow_query_log` 和 `long_query_time` 开启，结合 `mysqldumpslow` 和 `EXPLAIN` 定位全表扫描、无索引等问题。优化日志性能需平衡“排查需求”和“系统开销”，合理设置阈值、定期归档，避免日志过大影响 MySQL 性能。
</details>


## 15. MySQL 索引设计的核心原则有哪些？如何评估一个索引的有效性（如区分度、选择性）？举例说明错误的索引设计及优化方案。

<details>
<summary>答案与解析</summary>

- 思路要点：  
  1. **索引设计核心原则**：围绕“高效利用、避免冗余、适配查询”三个核心，涵盖字段选择（区分度、高频查询列）、联合索引顺序（最左前缀、高频列优先）、索引数量控制（避免过度索引）等；  
  2. **有效性评估指标**：区分度（字段值的唯一程度）、选择性（区分度的量化指标）、索引使用率（是否被频繁使用），通过公式和工具计算评估；  
  3. **错误案例与优化**：结合实际场景（如低区分度列建索引、冗余索引、联合索引顺序错误），给出具体优化方案，确保理论与实践结合。


### 一、MySQL 索引设计的核心原则
#### 1. 优先为“高频查询列”建索引
**原则说明**：索引的核心价值是加速查询，仅为 `WHERE`、`JOIN`、`ORDER BY`、`GROUP BY` 中频繁出现的列建索引，避免为“仅用于 `SELECT` 或低频查询”的列建索引（如用户表的 `intro` 字段，仅个人中心页面查询，无需索引）。  
**示例**：  
- 高频查询 SQL：`SELECT * FROM orders WHERE user_id = 123 AND status = 1 ORDER BY created_at DESC`；  
- 应建索引：`idx_user_status_created (user_id, status, created_at)`（覆盖 `WHERE` 和 `ORDER BY`）。


#### 2. 选择“高区分度”的列建索引
**原则说明**：区分度 = 唯一值数量 / 总记录数，区分度越高，索引过滤数据的能力越强（如 `user_id` 区分度接近 1，`gender` 区分度低）。避免为低区分度列（如 `gender`、`status` 取值少的列）单独建索引（过滤效果差，索引开销高于收益）。  
**反例**：为 `user` 表的 `gender` 列（取值仅 `male`/`female`）建独立索引，查询 `WHERE gender = 'male'` 会扫描 50% 数据，索引无效；  
**正例**：为 `user` 表的 `user_id`（唯一）、`email`（唯一）建索引，查询 `WHERE user_id = 123` 仅扫描 1 行。


#### 3. 联合索引遵循“最左前缀原则”，高频列、高区分度列放左侧
**原则说明**：联合索引的生效依赖“最左前缀匹配”，需将“高频出现的列”和“高区分度的列”放在左侧，最大化索引复用率。  
**示例**：  
- 常见查询场景：  
  1. `WHERE user_id = 123`；  
  2. `WHERE user_id = 123 AND status = 1`；  
  3. `WHERE user_id = 123 AND status = 1 ORDER BY created_at`；  
- 联合索引设计：`idx_user_status_created (user_id, status, created_at)`（`user_id` 高频且高区分度，放最左；`status` 次之；`created_at` 用于排序，放最后）；  
- 效果：三个查询场景均可利用该索引，无需建多个独立索引。


#### 4. 避免“冗余索引”和“重复索引”
**原则说明**：  
- 冗余索引：索引 A 是索引 B 的前缀（如已建 `(a,b)`，再建 `(a)` 就是冗余）；  
- 重复索引：相同列、相同顺序的索引（如 `INDEX idx_a (a)` 和 `INDEX idx_a2 (a)`）；  
冗余索引会增加写入开销（INSERT/UPDATE/DELETE 需同步维护多个索引），浪费磁盘空间。  
**示例**：  
- 错误：建了 `idx_user_id (user_id)`，又建 `idx_user_status (user_id, status)`（`idx_user_id` 是冗余）；  
- 优化：删除 `idx_user_id`，仅保留 `idx_user_status`（`user_id` 查询可利用 `idx_user_status` 的最左前缀）。


#### 5. 避免“过度索引”，控制单表索引数量
**原则说明**：单表索引数量建议不超过 5-8 个。索引越多，写入操作（INSERT/UPDATE/DELETE）的开销越大（需更新所有相关索引的 B+ 树），且 MySQL 优化器选择索引的时间也会增加。  
**优化方案**：合并相似索引（如将 `(a)` 和 `(a,b)` 合并为 `(a,b)`），删除长期未使用的索引（通过 `sys.schema_unused_indexes` 查看）。


#### 6. 大字段优先用“前缀索引”，避免全字段索引
**原则说明**：对 `VARCHAR(255)` 等大字段（如 `username`、`email`），无需为全字段建索引，可建“前缀索引”（取字段前 N 个字符），减少索引体积，提升查询速度。  
**示例**：  
- 错误：为 `email` 字段（`VARCHAR(100)`）建全字段索引 `INDEX idx_email (email)`；  
- 优化：建前缀索引 `INDEX idx_email_prefix (email(20))`（前 20 个字符已能区分大部分邮箱，索引体积减少 80%）；  
- 注意：前缀长度需足够（通过 `SELECT COUNT(DISTINCT LEFT(email, N)) / COUNT(*) FROM user` 验证，确保区分度接近全字段）。


#### 7. 避免“函数操作索引列”，适配索引使用规则
**原则说明**：索引列上的函数操作（如 `YEAR(birthday) = 1990`）会导致索引失效，需将函数操作转移到常量端，或建“函数索引”（MySQL 8.0+ 支持）。  
**示例**：  
- 错误：`SELECT * FROM user WHERE YEAR(birthday) = 1990`（`birthday` 有索引但失效）；  
- 优化方案 1：改为 `WHERE birthday BETWEEN '1990-01-01' AND '1990-12-31'`（索引有效）；  
- 优化方案 2：建函数索引 `INDEX idx_year_birthday (YEAR(birthday))`（MySQL 8.0+ 支持，适合无法修改 SQL 的场景）。


### 二、评估索引有效性的核心指标与方法
#### 1. 区分度（Discrimination）
- **定义**：字段唯一值的比例，区分度越高，索引过滤能力越强；  
- **计算公式**：`区分度 = COUNT(DISTINCT column) / COUNT(*)`；  
- **判断标准**：  
  - 高区分度：> 0.8（如 `user_id`、`email`），适合建索引；  
  - 低区分度：< 0.1（如 `gender`、`status`），不适合单独建索引。  
- **计算示例**：  
  ```sql
  -- 计算 user 表 age 字段的区分度
  SELECT 
      COUNT(DISTINCT age) / COUNT(*) AS age_discrimination,
      COUNT(DISTINCT user_id) / COUNT(*) AS user_id_discrimination
  FROM user;
  -- 结果：age_discrimination=0.3，user_id_discrimination=0.99（user_id 更适合建索引）
  ```


#### 2. 选择性（Selectivity）
- **定义**：选择性 = 1 - 区分度，与区分度相反，选择性越低，索引越有效；  
- **场景应用**：用于判断“索引过滤后的数据量”，选择性低（区分度高）的索引，过滤后的数据量少，查询快。


#### 3. 索引使用率
- **定义**：索引被查询使用的频率，长期未使用的索引是冗余的，应删除；  
- **查看方法**：  
  - MySQL 5.7+ 可通过 `sys.schema_unused_indexes` 查看未使用的索引：  
    ```sql
    SELECT * FROM sys.schema_unused_indexes WHERE table_name = 'user';
    ```
  - 通过慢查询日志或 `Performance Schema` 查看索引使用情况：  
    ```sql
    -- 查看索引使用统计
    SELECT 
        table_name,
        index_name,
        rows_read,
        rows_selected
    FROM performance_schema.table_io_waits_summary_by_index_usage
    WHERE table_name = 'user';
    ```


#### 4. 执行计划分析（EXPLAIN）
通过 `EXPLAIN` 查看索引是否被有效使用，关键指标：  
- `type`：`ref`/`range` 表示索引有效，`ALL` 表示全表扫描（索引失效）；  
- `key`：显示实际使用的索引名称，`NULL` 表示未使用索引；  
- `rows`：预估扫描行数，越少越好（索引有效时行数远小于总数据量）；  
- `Extra`：`Using index` 表示覆盖索引（无需回表），`Using filesort`/`Using temporary` 表示索引未优化排序/临时表（需优化）。


### 三、错误索引设计案例与优化方案
#### 案例 1：低区分度列单独建索引
**错误设计**：为 `orders` 表的 `status` 列（取值：0=待支付、1=已支付、2=已取消，区分度 0.03）建独立索引 `idx_status (status)`；  
**问题**：查询 `WHERE status = 1` 会扫描 30% 数据（约 30 万行），索引开销高于全表扫描，且写入时需维护索引；  
**优化方案**：  
- 不单独建 `idx_status`，将 `status` 作为联合索引的非最左列（如 `idx_user_status (user_id, status)`）；  
- 若需按 `status` 查询（如“统计已支付订单数”），建覆盖索引 `idx_status_count (status, id)`（仅扫描索引，无需回表）：  
  ```sql
  CREATE INDEX idx_status_count ON orders (status, id);
  -- 查询时：SELECT COUNT(id) FROM orders WHERE status = 1;（利用覆盖索引，快速统计）
  ```


#### 案例 2：联合索引顺序错误
**错误设计**：`orders` 表常见查询 `WHERE status = 1 AND user_id = 123`，建联合索引 `idx_status_user (status, user_id)`；  
**问题**：`status` 区分度低（仅 3 个值），放最左列导致索引过滤能力弱，查询时需扫描大量行；  
**优化方案**：交换列顺序，建 `idx_user_status (user_id, status)`（`user_id` 区分度高，放最左，先过滤出 `user_id=123` 的订单，再过滤 `status=1`，扫描行数从 1 万减少到 10 行）。


#### 案例 3：冗余索引与重复索引
**错误设计**：`user` 表建了以下索引：  
- `idx_user_id (user_id)`（主键索引已包含 `user_id`，冗余）；  
- `idx_user_email (email)`；  
- `idx_email_user (email, user_id)`（`idx_user_email` 是冗余，因 `(email, user_id)` 的最左前缀是 `email`）；  
**问题**：写入时需维护 3 个索引，磁盘空间浪费，写入性能下降；  
**优化方案**：  
- 删除 `idx_user_id`（主键索引 `PRIMARY KEY (user_id)` 已满足 `user_id` 查询需求）；  
- 删除 `idx_user_email`（`idx_email_user` 的最左前缀 `email` 可覆盖 `email` 查询需求）；  
- 最终保留：`PRIMARY KEY (user_id)` 和 `idx_email_user (email, user_id)`。


### 总结
MySQL 索引设计的核心是“适配查询、高效过滤、避免冗余”，需优先为高频、高区分度列建索引，联合索引按“最左前缀+高频优先”排序。评估索引有效性需关注区分度、使用率和执行计划，通过删除冗余索引、优化联合索引顺序、使用前缀索引等方式提升索引效率。错误索引设计会导致性能反降，需结合业务查询场景持续优化。
</details>