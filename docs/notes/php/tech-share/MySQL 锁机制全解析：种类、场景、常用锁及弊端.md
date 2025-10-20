---
title: MySQL 锁机制全解析：种类、场景、常用锁及弊端
createTime: 2025/09/15 17:34:32
permalink: /php/技术分享/55yqn37c/
---

# MySQL 锁机制全解析：种类、场景、常用锁及弊端（含实操代码+Webman适配）

MySQL 锁是保障并发场景下数据一致性的核心机制，但其设计复杂，不同锁的粒度、兼容性和适用场景差异极大。本文将从 **锁的分类维度** 出发，结合 **MySQL 实操代码** 与 **Webman 框架业务代码**，拆解每种锁的定义、场景、常用性及弊端，覆盖“手动加锁”“框架中落地”“问题排查”等关键环节，让锁机制从抽象概念落地到实际开发。


## 一、MySQL 锁的核心分类逻辑
MySQL 锁可从多个维度划分，最核心的是 **“锁粒度”** 和 **“锁兼容性”**。InnoDB 引擎还引入“意向锁”“间隙锁”等特殊锁，用于协调不同粒度的锁冲突。

整体分类框架如下：
```
MySQL 锁
├─ 按锁粒度划分
│  ├─ 全局锁（锁整个数据库）
│  ├─ 表锁（锁整张表）
│  └─ 行锁（锁表中某一行/多行）
├─ 按锁兼容性划分（基于粒度的子分类）
│  ├─ 共享锁（S 锁：读锁，多事务可共享）
│  └─ 排他锁（X 锁：写锁，仅单事务可持有）
└─ InnoDB 特殊锁（协调粒度与防幻读）
   ├─ 意向锁（IS/IX：表级，标记行锁意图）
   ├─ 间隙锁（Gap Lock：锁范围，防幻读）
   └─ 临键锁（Next-Key Lock：行+间隙，默认锁机制）
```


## 二、按锁粒度划分：全局锁、表锁、行锁
### 1. 全局锁（Global Lock）
#### 定义
锁定 **整个 MySQL 实例**（所有数据库），此时所有数据库的所有表都只能读，不能执行任何写操作（INSERT/UPDATE/DELETE/ALTER 等）。

#### 实操代码（MySQL 手动加锁/解锁/验证）
```sql
-- 1. 加全局锁（仅 root 权限可执行）
FLUSH TABLES WITH READ LOCK;

-- 2. 验证锁效果：尝试写操作（如插入数据），会阻塞
USE test_db; -- 切换到测试库
INSERT INTO product (name, price) VALUES ('测试商品', 99); -- 执行后会卡住，直到解锁

-- 3. 解锁（或断开当前连接也会自动解锁）
UNLOCK TABLES;

-- 4. 查看全局锁状态（通过进程列表判断）
SHOW PROCESSLIST; 
-- 若有 "Waiting for table metadata lock" 且状态为 "Locked"，说明存在全局锁
```

#### Webman 场景说明
全局锁**极少在 Webman 业务中主动使用**，因会阻塞所有写操作（如订单创建、商品更新）。仅在 Webman 系统维护时（如全量备份 MyISAM 表），通过脚本执行全局锁，而非业务代码：
```php
// Webman 维护脚本（如 script/backup.php）
use support\Db;

// 执行全局锁（仅备份时使用）
Db::statement('FLUSH TABLES WITH READ LOCK');

// 执行全量备份（模拟 mysqldump 逻辑，实际建议用系统命令）
$backupContent = Db::select('SELECT * FROM test_db.product');
file_put_contents('./backup/product_backup.sql', json_encode($backupContent));

// 解锁，恢复业务
Db::statement('UNLOCK TABLES');
```
**说明**：Webman 业务优先用 InnoDB 事务快照备份（`--single-transaction`），避免全局锁影响可用性。


### 2. 表锁（Table Lock）
#### 定义
锁定 **整张表**，分为“共享锁（读锁）”和“排他锁（写锁）”，支持手动加锁和自动加锁（MyISAM 写操作默认加写锁）。

#### 实操代码（MySQL 读锁/写锁示例）
##### （1）表共享锁（读锁）
```sql
-- 会话1：加表读锁
USE test_db;
LOCK TABLES product READ; -- 锁定 product 表为读模式

-- 会话1：读操作（允许）
SELECT * FROM product WHERE id = 1; -- 正常返回结果

-- 会话1：写操作（禁止，会报错）
UPDATE product SET price = 88 WHERE id = 1; -- 报错：Table 'product' was locked with a READ lock and can't be updated

-- 会话2：读操作（允许，共享读锁）
SELECT * FROM product WHERE id = 1; -- 正常返回结果

-- 会话2：写操作（阻塞，直到会话1解锁）
UPDATE product SET price = 88 WHERE id = 1; -- 卡住，等待锁释放

-- 会话1：解锁
UNLOCK TABLES;

-- 会话2：写操作自动执行
```

##### （2）表排他锁（写锁）
```sql
-- 会话1：加表写锁
USE test_db;
LOCK TABLES product WRITE; -- 锁定 product 表为写模式

-- 会话1：读/写操作（均允许）
SELECT * FROM product WHERE id = 1; -- 正常
UPDATE product SET price = 77 WHERE id = 1; -- 正常

-- 会话2：读操作（阻塞，写锁不共享）
SELECT * FROM product WHERE id = 1; -- 卡住

-- 会话1：解锁
UNLOCK TABLES;

-- 会话2：读操作自动执行
```

#### Webman 代码适配（表锁批量更新配置表）
Webman 中表锁仅用于 **批量更新全表且无并发的场景**（如配置表同步），通过 `Db` 类执行表锁 SQL，封装在 Service 层：
```php
// app/common/service/ConfigService.php（Webman 服务类）
namespace app\common\service;

use support\Db;
use think\Exception;

class ConfigService
{
    /**
     * 批量更新配置表（全表更新，用表锁避免并发冲突）
     * @param array $configs 配置数据（key=>value）
     * @return bool
     */
    public function batchUpdateConfig(array $configs): bool
    {
        try {
            // 1. 加表排他锁（禁止其他会话读写，确保批量更新原子性）
            Db::statement('LOCK TABLES system_config WRITE');
            
            // 2. 批量更新逻辑（清空旧配置→插入新配置，模拟全量同步）
            Db::table('system_config')->truncate(); // 清空表
            $insertData = array_map(function ($key, $value) {
                return ['config_key' => $key, 'config_value' => $value, 'updated_at' => date('Y-m-d H:i:s')];
            }, array_keys($configs), $configs);
            Db::table('system_config')->insertAll($insertData);
            
            // 3. 解锁（必须执行，避免锁残留）
            Db::statement('UNLOCK TABLES');
            
            return true;
        } catch (Exception $e) {
            // 异常时强制解锁，防止锁残留
            Db::statement('UNLOCK TABLES');
            throw new Exception("批量更新配置失败：{$e->getMessage()}");
        }
    }
}
```
**说明**：此代码对应 MySQL 表排他锁操作，Webman 中通过 `Db::statement` 执行原生 SQL 加锁，适用于“配置同步”等低并发全表操作，高并发场景优先用行锁。


### 3. 行锁（Row Lock）
#### 定义
锁定 **表中某一行/多行**，仅 InnoDB 支持，是高并发的核心保障。分为“行共享锁（S 锁）”和“行排他锁（X 锁）”，需在事务中使用。

#### 实操代码（MySQL S 锁/X 锁示例）
##### （1）行共享锁（S 锁：读锁，`FOR SHARE`）
```sql
-- 会话1：开启事务，加行 S 锁（MySQL 8.0+，5.7 用 FOR UPDATE 替代）
USE test_db;
START TRANSACTION;
SELECT * FROM product WHERE id = 1 FOR SHARE; -- 锁定 id=1 的行，允许其他会话读

-- 会话1：写操作（允许，S 锁持有者可写）
UPDATE product SET price = 55 WHERE id = 1; -- 正常执行

-- 会话2：开启事务，加行 S 锁（允许，共享读）
START TRANSACTION;
SELECT * FROM product WHERE id = 1 FOR SHARE; -- 正常返回

-- 会话2：写操作（阻塞，S 锁不允许其他会话写）
UPDATE product SET price = 44 WHERE id = 1; -- 卡住

-- 会话1：提交事务，释放锁
COMMIT;

-- 会话2：写操作自动执行
COMMIT;
```

##### （2）行排他锁（X 锁：写锁，`FOR UPDATE`）
```sql
-- 会话1：开启事务，加行 X 锁（电商扣库存核心场景）
USE test_db;
START TRANSACTION;
-- 锁定 id=1 的商品行，防止其他会话修改
SELECT * FROM product WHERE id = 1 FOR UPDATE; 

-- 扣减库存（核心业务逻辑）
UPDATE product SET stock = stock - 1 WHERE id = 1; 

-- 会话2：开启事务，加行 X 锁（阻塞，X 锁不共享）
START TRANSACTION;
SELECT * FROM product WHERE id = 1 FOR UPDATE; -- 卡住

-- 会话1：提交事务，释放锁
COMMIT;

-- 会话2：锁释放，查询到更新后的库存
COMMIT;
```

##### （3）反例：索引失效导致行锁升级表锁
```sql
-- 场景：product 表的 name 字段无索引，查询条件用 name，行锁升级为表锁
USE test_db;
START TRANSACTION;
-- name 无索引，MySQL 无法定位行，自动加表锁
SELECT * FROM product WHERE name = '测试商品' FOR UPDATE; 

-- 此时其他会话操作 product 表任何行都会阻塞（如 id=2 的行）
-- 解决方案：给 name 加索引
ALTER TABLE product ADD INDEX idx_name (name);
```

#### Webman 代码适配（行锁扣库存+订单创建）
Webman 中通过 **think-orm 的 `lock` 方法** 实现行锁，封装在 Service 层，结合事务确保原子性（电商订单创建核心逻辑）：
```php
// app/order/service/OrderService.php（Webman 订单服务）
namespace app\order\service;

use app\product\model\Product;
use app\order\model\Order;
use app\order\model\OrderItem;
use support\Db;
use think\Exception;

class OrderService
{
    /**
     * 创建订单（行锁扣库存，防超卖）
     * @param int $userId 用户ID
     * @param int $productId 商品ID
     * @param int $num 购买数量
     * @return array
     */
    public function createOrder(int $userId, int $productId, int $num): array
    {
        try {
            // 1. 开启 MySQL 事务（Webman 中通过 Db::transaction 封装）
            $order = Db::transaction(function () use ($userId, $productId, $num) {
                // 2. 加行排他锁（X 锁）：锁定目标商品行，think-orm 用 lock('for update') 实现
                $product = Product::where('id', $productId)
                    ->lock('for update') // 对应 MySQL 的 SELECT ... FOR UPDATE
                    ->find();
                
                // 3. 库存校验
                if (!$product || $product->stock < $num) {
                    throw new Exception("商品【{$productId}】库存不足");
                }
                
                // 4. 扣减库存（行锁已锁定，无并发冲突）
                $product->decrement('stock', $num);
                $product->save();
                
                // 5. 创建订单主记录
                $orderNo = $this->generateOrderNo(); // 生成唯一订单号
                $order = Order::create([
                    'order_no' => $orderNo,
                    'user_id' => $userId,
                    'total_amount' => $product->price * $num,
                    'pay_amount' => $product->price * $num,
                    'status' => 1, // 1=待支付
                    'created_at' => date('Y-m-d H:i:s')
                ]);
                
                // 6. 创建订单项
                OrderItem::create([
                    'order_id' => $order->id,
                    'product_id' => $productId,
                    'product_name' => $product->name,
                    'price' => $product->price,
                    'num' => $num
                ]);
                
                return $order;
            });
            
            return $order->toArray();
        } catch (Exception $e) {
            throw new Exception("创建订单失败：{$e->getMessage()}");
        }
    }
    
    /**
     * 生成唯一订单号
     * @return string
     */
    private function generateOrderNo(): string
    {
        return 'ORD' . date('YmdHis') . mt_rand(1000, 9999);
    }
}
```
**说明**：此代码对应 MySQL 行排他锁操作，Webman 中通过 `lock('for update')` 让 ORM 生成 `SELECT ... FOR UPDATE` 语句，在多进程下确保“库存扣减+订单创建”的原子性，防止超卖。


## 三、InnoDB 特殊锁：意向锁、间隙锁、临键锁
这些锁由 InnoDB 自动管理，无需手动加，但需理解其行为以避免异常阻塞。

### 1. 意向锁（Intention Lock）
#### 定义
**表级锁**，标记“事务即将对表中行加 S/X 锁”，避免表锁与行锁冲突。分为意向共享锁（IS）和意向排他锁（IX）。

#### 实操代码（MySQL 查看意向锁）
```sql
-- 1. 会话1：开启事务，加行 X 锁（自动加表 IX 锁）
USE test_db;
START TRANSACTION;
SELECT * FROM product WHERE id = 1 FOR UPDATE;

-- 2. 查看意向锁（通过 INNODB_LOCKS 表）
SELECT * FROM information_schema.INNODB_LOCKS;
-- 结果中会出现：
-- LOCK_TYPE = 'TABLE'，LOCK_MODE = 'IX'（意向排他锁）
-- LOCK_TYPE = 'RECORD'，LOCK_MODE = 'X'（行排他锁）

-- 3. 会话2：尝试加表写锁（会阻塞，因表有 IX 锁）
LOCK TABLES product WRITE; -- 卡住，直到会话1提交

-- 4. 会话1：提交事务，释放锁
COMMIT;
```

#### Webman 场景说明
意向锁由 InnoDB 自动加，Webman 中无需手动操作，但需注意“表锁与行锁的冲突”。例如，Webman 中若有“批量更新表结构”的操作，需先确保无业务进程持有行锁：
```php
// Webman 表结构更新脚本（避免与行锁冲突）
use support\Db;

// 检查是否有行锁持有（通过查询 INNODB_LOCKS）
$locks = Db::table('information_schema.INNODB_LOCKS')
    ->where('TABLE_NAME', 'product')
    ->where('LOCK_MODE', 'like', 'IX%') // 过滤意向排他锁（表示有行锁）
    ->find();

if ($locks) {
    die("存在行锁持有，无法更新表结构，请稍后重试");
}

// 无行锁时，执行表结构更新（自动加表写锁）
Db::statement('ALTER TABLE product ADD COLUMN sales INT DEFAULT 0');
```


### 2. 间隙锁（Gap Lock）
#### 定义
锁定 **索引区间的间隙**（不包含具体行），防止幻读，仅 InnoDB RR 隔离级别生效。

#### 实操代码（MySQL 间隙锁阻塞插入）
```sql
-- 前提：product 表 id 为自增主键，现有数据 id=1,3,5（间隙为 (1,3),(3,5)）
USE test_db;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ; -- 确保 RR 隔离级别

-- 会话1：开启事务，范围查询加间隙锁
START TRANSACTION;
SELECT * FROM product WHERE id BETWEEN 2 AND 4 FOR UPDATE; -- 锁定间隙 (1,3) 和 (3,5)

-- 会话2：尝试插入间隙中的数据（阻塞）
START TRANSACTION;
INSERT INTO product (id, name, price) VALUES (2, '间隙商品', 22); -- 卡住

-- 会话1：提交事务，释放锁
COMMIT;

-- 会话2：插入成功
COMMIT;
```

#### Webman 代码适配（间隙锁防幻读）
Webman 中通过“范围查询+行锁”触发间隙锁，适用于“批量操作连续ID数据”的场景（如批量关闭订单）：
```php
// app/order/service/OrderService.php（Webman 批量关闭订单）
namespace app\order\service;

use app\order\model\Order;
use support\Db;
use think\Exception;

class OrderService
{
    /**
     * 批量关闭超时订单（间隙锁防幻读）
     * @param int $minId 订单ID最小值
     * @param int $maxId 订单ID最大值
     * @return int 关闭成功的订单数
     */
    public function batchCloseExpiredOrder(int $minId, int $maxId): int
    {
        try {
            // 1. 开启事务，RR 隔离级别默认生效（触发间隙锁）
            $closeCount = Db::transaction(function () use ($minId, $maxId) {
                // 2. 范围查询加行锁：锁定 ID 200-300 的订单，触发间隙锁防幻读
                // 此时无法插入 ID 在 (200,300) 区间的新订单，避免“关闭期间新增订单被遗漏”
                $orders = Order::where('id', 'between', [$minId, $maxId])
                    ->where('status', 1) // 1=待支付
                    ->where('created_at', '<=', date('Y-m-d H:i:s', time() - 1800)) // 30分钟超时
                    ->lock('for update') // 行锁+间隙锁
                    ->select();
                
                // 3. 批量关闭订单
                $orderIds = array_column($orders->toArray(), 'id');
                if (empty($orderIds)) {
                    return 0;
                }
                
                return Order::whereIn('id', $orderIds)
                    ->update([
                        'status' => 3, // 3=已取消
                        'canceled_at' => date('Y-m-d H:i:s'),
                        'cancel_reason' => '超时未支付'
                    ]);
            });
            
            return $closeCount;
        } catch (Exception $e) {
            throw new Exception("批量关闭订单失败：{$e->getMessage()}");
        }
    }
}
```
**说明**：此代码在 Webman 中触发 InnoDB 间隙锁，锁定 `id between 200 and 300` 的区间，防止批量关闭期间插入新订单导致“幻读”（即关闭后出现新的超时订单未处理）。


### 3. 临键锁（Next-Key Lock）
#### 定义
**行锁 + 间隙锁的组合**，锁定“当前行 + 该行右侧的间隙”，是 InnoDB RR 隔离级别下的 **默认锁机制**。

#### 实操代码（MySQL 临键锁效果）
```sql
-- 前提：product 表 id=1,3,5
USE test_db;
START TRANSACTION;
-- 加临键锁：锁定 id=3 的行 + 右侧间隙 (3,5)
SELECT * FROM product WHERE id = 3 FOR UPDATE;

-- 会话2：插入 id=4 的数据（阻塞，临键锁锁定 (3,5) 间隙）
START TRANSACTION;
INSERT INTO product (id, name, price) VALUES (4, '临键商品', 44); -- 卡住

-- 会话1：提交事务
COMMIT;

-- 会话2：插入成功
COMMIT;
```

#### Webman 场景说明
临键锁是 InnoDB 默认锁机制，Webman 中无需额外配置，只需确保隔离级别为 RR（默认）。例如，Webman 中更新单个商品时，临键锁会自动锁定“当前行+右侧间隙”，防止幻读：
```php
// app/product/service/ProductService.php（Webman 更新商品）
namespace app\product\service;

use app\product\model\Product;
use support\Db;
use think\Exception;

class ProductService
{
    /**
     * 更新商品价格（临键锁自动生效）
     * @param int $productId 商品ID
     * @param float $newPrice 新价格
     * @return bool
     */
    public function updateProductPrice(int $productId, float $newPrice): bool
    {
        try {
            return Db::transaction(function () use ($productId, $newPrice) {
                $product = Product::where('id', $productId)
                    ->lock('for update') // 自动触发临键锁
                    ->find();
                
                if (!$product) {
                    throw new Exception("商品【{$productId}】不存在");
                }
                
                // 更新价格（临键锁确保更新期间无新商品插入当前行右侧间隙）
                $product->price = $newPrice;
                return $product->save();
            });
        } catch (Exception $e) {
            throw new Exception("更新商品价格失败：{$e->getMessage()}");
        }
    }
}
```
**说明**：更新 `id=3` 的商品时，临键锁会锁定 `id=3` 行 + `(3,5)` 间隙，防止其他会话插入 `id=4` 的商品，避免“更新期间出现新商品导致查询结果不一致”。


## 四、死锁示例与排查（行锁核心问题）
行锁易因“锁顺序不一致”导致死锁，需通过 MySQL 代码复现和 Webman 代码规避。

### 1. 死锁复现代码（MySQL）
```sql
-- 前提：product 表 id=1 和 id=2 的行存在
-- 会话1：先锁 id=1，再尝试锁 id=2
USE test_db;
START TRANSACTION;
SELECT * FROM product WHERE id = 1 FOR UPDATE; -- 锁 id=1
SELECT SLEEP(10); -- 延迟10秒，给会话2加锁时间
SELECT * FROM product WHERE id = 2 FOR UPDATE; -- 尝试锁 id=2（阻塞）

-- 会话2：先锁 id=2，再尝试锁 id=1（在会话1 SLEEP 期间执行）
USE test_db;
START TRANSACTION;
SELECT * FROM product WHERE id = 2 FOR UPDATE; -- 锁 id=2
SELECT * FROM product WHERE id = 1 FOR UPDATE; -- 尝试锁 id=1（阻塞）

-- 10秒后，MySQL 检测到死锁，自动回滚其中一个事务
```

### 2. Webman 代码适配（死锁规避：统一锁顺序）
Webman 中通过 **“统一锁顺序”** 规避死锁，例如多商品下单时按 `product_id` 升序锁定：
```php
// app/order/service/OrderService.php（Webman 多商品下单，规避死锁）
namespace app\order\service;

use app\product\model\Product;
use app\order\model\Order;
use app\order\model\OrderItem;
use support\Db;
use think\Exception;

class OrderService
{
    /**
     * 多商品下单（统一锁顺序，规避死锁）
     * @param int $userId 用户ID
     * @param array $productItems 商品列表（[{product_id:1, num:2}, {product_id:2, num:1}]）
     * @return array
     */
    public function createMultiProductOrder(int $userId, array $productItems): array
    {
        try {
            // 1. 提取商品ID并按升序排序（核心：统一锁顺序）
            $productIds = array_column($productItems, 'product_id');
            sort($productIds); // 按 product_id 升序，确保所有事务锁顺序一致
            
            // 2. 开启事务
            $order = Db::transaction(function () use ($userId, $productItems, $productIds) {
                // 3. 按升序锁定所有商品（避免死锁）
                $products = [];
                foreach ($productIds as $productId) {
                    $product = Product::where('id', $productId)
                        ->lock('for update')
                        ->find();
                    if (!$product) {
                        throw new Exception("商品【{$productId}】不存在");
                    }
                    $products[$productId] = $product;
                }
                
                // 4. 库存校验
                $totalAmount = 0;
                foreach ($productItems as $item) {
                    $product = $products[$item['product_id']];
                    if ($product->stock < $item['num']) {
                        throw new Exception("商品【{$item['product_id']}】库存不足");
                    }
                    $totalAmount += $product->price * $item['num'];
                }
                
                // 5. 扣减库存
                foreach ($productItems as $item) {
                    $product = $products[$item['product_id']];
                    $product->decrement('stock', $item['num']);
                    $product->save();
                }
                
                // 6. 创建订单（省略订单项创建逻辑）
                $orderNo = $this->generateOrderNo();
                $order = Order::create([
                    'order_no' => $orderNo,
                    'user_id' => $userId,
                    'total_amount' => $totalAmount,
                    'status' => 1
                ]);
                
                return $order;
            });
            
            return $order->toArray();
        } catch (Exception $e) {
            throw new Exception("多商品下单失败：{$e->getMessage()}");
        }
    }
}
```
**说明**：Webman 中通过 `sort($productIds)` 统一按商品ID升序锁定，无论用户下单时商品顺序如何，事务都按“1→2→3”的顺序加锁，从根源规避死锁。


## 五、常用锁选择建议（MySQL + Webman 对应关系）
| 业务场景                  | MySQL 锁类型              | Webman 代码核心逻辑                                  | 注意事项                          |
|---------------------------|---------------------------|-------------------------------------------------------|-----------------------------------|
| 电商扣库存（高并发）      | 行排他锁（X 锁）          | `Product::where('id', $id)->lock('for update')->find();` | 确保 `id` 有索引，避免升级表锁 |
| 多商品下单（防死锁）      | 行排他锁（按ID升序）      | 先排序商品ID，再循环加锁                              | 统一锁顺序是关键                  |
| 批量关闭订单（防幻读）    | 间隙锁（Gap Lock）        | `Order::where('id', 'between', [$min, $max])->lock('for update')->select();` | RR 隔离级别默认生效 |
| 配置表全量更新（低并发）  | 表排他锁（WRITE）         | `Db::statement('LOCK TABLES config WRITE');` + 批量更新 | 避开业务高峰，及时解锁            |
| 读数据不允许写（不修改）  | 行共享锁（S 锁）          | `User::where('id', $id)->lock('for share')->find();`   | MySQL 8.0+ 支持，5.7 用 X 锁替代 |


## 六、总结
1. **引擎决定锁能力**：MyISAM 仅表锁，InnoDB 支持行锁+特殊锁，Webman 高并发业务必用 InnoDB；
2. **MySQL 与 Webman 联动**：Webman 通过 think-orm 的 `lock` 方法封装 MySQL 锁逻辑，让锁机制落地到业务服务层；
3. **核心避坑点**：行锁需索引、死锁需统一锁顺序、间隙锁需注意隔离级别，这些在 Webman 代码中需主动处理；
4. **场景优先**：无绝对“好锁”，只有“适合场景的锁”——高并发用行锁，全表操作用工表锁，防幻读用间隙锁。

通过 MySQL 代码理解锁的底层逻辑，再通过 Webman 代码落地到实际业务，是掌握 MySQL 锁机制的关键。实际开发中需结合业务并发量、数据一致性要求，选择最合适的锁方案。