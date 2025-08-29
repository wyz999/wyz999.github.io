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