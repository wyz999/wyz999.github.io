---
title: php面经-性能与安全篇
createTime: 2025/08/29 10:46:11
permalink: /php/面试题/performance-security/
---
## 1. 如何通过 PHP 代码设置和获取 Cookie、Session？Session 的存储方式有哪些？在分布式系统中，为什么不推荐用文件存储 Session？

<details>
<summary>答案与解析</summary>

- 思路要点：

  1. **Cookie 的设置与获取**：

     - 设置：通过 `setcookie()` 函数，需指定名称、值、过期时间等参数（需在输出任何内容前调用）。
     - 获取：通过超全局数组 `$_COOKIE` 读取，键为 Cookie 名称。
  2. **Session 的设置与获取**：

     - 必须先通过 `session_start()` 启动 Session（同样需在输出前调用）。
     - 设置：通过超全局数组 `$_SESSION` 赋值（如 `$_SESSION['user'] = 'test'`）。
     - 获取：直接访问 `$_SESSION` 数组的对应键。
     - 销毁：通过 `session_destroy()` 销毁当前 Session 数据，或 `unset($_SESSION['key'])` 销毁单个键。
  3. **Session 的存储方式**：

     - 文件存储（默认）：数据保存在服务器本地文件中（通常路径为 `php.ini` 中的 `session.save_path`）。
     - 数据库存储：将 Session 数据存入 MySQL 等数据库，需自定义 `session_set_save_handler()`。
     - 缓存存储：如 Redis、Memcached，通过配置 `session.save_handler = redis` 等实现。
     - 其他：如 Memcache、MongoDB 等分布式存储方案。
  4. **分布式系统中不推荐文件存储 Session 的原因**：

     - 数据不共享：文件存储是本地的，多台服务器（如负载均衡下的集群）无法共享 Session 文件，导致用户在不同服务器间切换时 Session 丢失（如登录状态失效）。
     - 性能瓶颈：高并发下，文件读写和锁机制（避免并发冲突）会导致 IO 阻塞，降低系统响应速度。
     - 扩展性差：文件存储难以水平扩展，新增服务器需同步 Session 文件，维护成本高。
- 示例代码：

```php
<?php
// 1. Cookie 的设置与获取
// 设置 Cookie（有效期1小时，路径为根目录，仅HTTPS可用，禁止JS访问）
setcookie('username', 'zhangsan', time() + 3600, '/', '', true, true);

// 获取 Cookie（需在设置后的请求中读取）
if (isset($_COOKIE['username'])) {
    echo "获取Cookie：" . $_COOKIE['username'] . "<br>";
}

// 删除 Cookie（设置过期时间为过去）
setcookie('username', '', time() - 3600, '/');


// 2. Session 的设置与获取
// 启动 Session（必须在输出前调用）
session_start();

// 设置 Session
$_SESSION['user_id'] = 123;
$_SESSION['is_login'] = true;

// 获取 Session
if (isset($_SESSION['user_id'])) {
    echo "获取Session用户ID：" . $_SESSION['user_id'] . "<br>";
    echo "登录状态：" . ($_SESSION['is_login'] ? '已登录' : '未登录') . "<br>";
}

// 销毁单个 Session 键
unset($_SESSION['is_login']);

// 销毁所有 Session 数据（需先启动Session）
session_destroy();
// 注意：session_destroy() 不会立即清空 $_SESSION，需手动 unset 或重新启动
$_SESSION = [];
session_regenerate_id(true); // 可选：刷新 Session ID 增强安全性
?>
```

</details>

## 2. 列举 PHP 项目中常见的安全漏洞，并说明对应的基础防御措施（如 XSS 用 htmlspecialchars() 过滤，CSRF 用令牌验证）

<details>
<summary>答案与解析</summary>

- 要点清单：

  1. **XSS（跨站脚本攻击）**

     - 定义：攻击者在页面注入恶意JavaScript代码（如 `<script>alert('xss')</script>`），当其他用户访问页面时执行，窃取Cookie、篡改页面等。
     - 防御措施：
       - 输出用户输入时用 `htmlspecialchars()` 过滤（转义 `<`、`>`、`&`等特殊字符）。
       - 设置HTTP头 `Content-Security-Policy` 限制脚本来源（如 `default-src 'self'`）。
       - 对富文本内容使用专用过滤库（如HTML Purifier），禁止危险标签/属性（如 `onclick`、`<script>`）。
  2. **CSRF（跨站请求伪造）**

     - 定义：攻击者诱导已登录用户在第三方网站发起恶意请求（如转账、修改密码），利用用户的登录状态执行非预期操作。
     - 防御措施：
       - 为每个表单生成唯一令牌（CSRF Token），提交时验证（如Laravel的 `@csrf`指令）。
       - 检查请求的 `Referer`或 `Origin`头，限制仅允许信任的域名发起请求。
       - 设置Cookie的 `SameSite`属性（`SameSite=Strict`或 `Lax`），阻止跨站请求携带Cookie。
  3. **SQL注入**

     - 定义：攻击者通过用户输入插入SQL代码（如 `' OR 1=1#`），篡改查询逻辑，非法访问或修改数据库。
     - 防御措施：
       - 使用PDO预处理语句（`prepare()`+`execute()`）或参数化查询，避免直接拼接SQL。
       - 限制数据库用户权限（如查询用户仅授予 `SELECT`权限，禁止 `DROP`、`DELETE`等高危操作）。
       - 对用户输入进行类型验证（如整数参数强制转为 `int`）。
  4. **文件上传漏洞**

     - 定义：攻击者上传恶意文件（如包含PHP代码的 `.php`文件），通过访问文件执行恶意脚本，获取服务器权限。
     - 防御措施：
       - 严格验证文件类型：结合 `mime_content_type()`检查MIME类型，而非仅依赖文件后缀。
       - 限制文件后缀：仅允许安全类型（如 `.jpg`、`.png`），并将上传文件重命名为随机字符串（避免路径遍历）。
       - 将上传文件存储在Web访问目录外，或通过脚本间接读取（禁止直接访问上传文件）。
  5. **命令注入**

     - 定义：当代码使用 `exec()`、`system()`等函数执行系统命令时，攻击者通过用户输入注入额外命令（如 `; rm -rf /`）。
     - 防御措施：
       - 避免直接拼接用户输入到命令中，使用 `escapeshellarg()`或 `escapeshellcmd()`过滤参数。
       - 尽量用PHP内置函数替代系统命令（如用 `file_get_contents()`替代 `curl`命令）。
  6. **敏感信息泄露**

     - 定义：通过错误提示、日志或代码泄露数据库密码、API密钥、用户隐私等敏感信息。
     - 防御措施：
       - 生产环境关闭PHP错误显示（`display_errors = Off`），仅记录到日志（`log_errors = On`）。
       - 敏感配置（如数据库密码）存储在环境变量或非Web访问目录的配置文件中。
       - 传输敏感数据时使用HTTPS加密，避免明文传输。
  7. **密码安全漏洞**

     - 定义：明文存储密码、使用弱哈希算法（如MD5）或未加盐哈希，导致密码被轻易破解。
     - 防御措施：
       - 使用PHP内置函数 `password_hash()`（自动加盐，默认使用bcrypt算法）存储密码，`password_verify()`验证。
       - 强制用户使用强密码（包含大小写字母、数字、特殊字符，长度≥8位）。
- 示例代码：

```php
<?php
// 1. XSS防御示例
$userInput = '<script>alert("xss")</script>';
// 输出时过滤
echo "安全输出：" . htmlspecialchars($userInput, ENT_QUOTES, 'UTF-8') . "<br>";
// 输出结果：<script>alert("xss")</script>


// 2. CSRF防御示例（生成并验证令牌）
session_start();
// 生成令牌
if (empty($_SESSION['csrf_token'])) {
    $_SESSION['csrf_token'] = bin2hex(random_bytes(32));
}
// 表单中嵌入令牌
echo '<form method="post" action="submit.php">';
echo '<input type="hidden" name="csrf_token" value="' . $_SESSION['csrf_token'] . '">';
echo '<button type="submit">提交</button>';
echo '</form>';

// 验证令牌（submit.php）
if ($_POST['csrf_token'] !== $_SESSION['csrf_token']) {
    die("CSRF令牌验证失败");
}


// 3. 文件上传防御示例
if ($_FILES['file']['error'] === UPLOAD_ERR_OK) {
    $allowedTypes = ['image/jpeg', 'image/png'];
    $fileType = mime_content_type($_FILES['file']['tmp_name']);
  
    // 验证MIME类型
    if (!in_array($fileType, $allowedTypes)) {
        die("仅允许上传JPG/PNG图片");
    }
  
    // 重命名文件（避免路径遍历和同名覆盖）
    $fileName = uniqid() . '.' . pathinfo($_FILES['file']['name'], PATHINFO_EXTENSION);
    $uploadPath = '/var/www/uploads/' . $fileName; // 存储在Web目录外
  
    move_uploaded_file($_FILES['file']['tmp_name'], $uploadPath);
}


// 4. 密码安全示例
$userPassword = 'User@123';
// 加密存储
$hashedPassword = password_hash($userPassword, PASSWORD_DEFAULT);
// 验证密码
if (password_verify('User@123', $hashedPassword)) {
    echo "密码验证成功";
} else {
    echo "密码错误";
}
?>
```

</details>

## 3. 如何优化 PHP 页面加载速度？请从代码层面（如减少 include 次数）、服务器层面（如启用 OPcache）举例说明

<details>
<summary>答案与解析</summary>

- 要点清单：

  1. **代码层面优化**

     - **减少文件加载次数**：避免频繁使用 `include`/`require`加载文件，改用自动加载（`spl_autoload_register`），仅在需要时加载类文件；合并小型静态文件（如工具函数）为单个文件，减少IO操作。
     - **优化数据库交互**：减少重复查询（用变量缓存结果），避免 `SELECT *`只查必要字段，使用批量操作（如 `INSERT INTO ... VALUES (...), (...)`）替代循环单条插入，添加合适索引。
     - **减少计算开销**：避免在循环中执行复杂运算（如字符串拼接、函数调用），将重复计算的结果缓存到变量；用 `isset()`替代 `array_key_exists()`检查数组键（性能更优）。
     - **压缩输出内容**：通过 `ob_start('ob_gzhandler')`开启输出缓冲并启用Gzip压缩，减少HTML/JSON等响应数据的传输体积。
     - **避免全局变量和超全局数组滥用**：全局变量查找耗时，超全局数组（如 `$_POST`）每次访问都会触发哈希表查找，建议局部缓存后使用（如 `$post = $_POST;`）。
  2. **服务器层面优化**

     - **启用OPcache**：PHP的字节码缓存工具，预编译PHP代码为字节码并缓存，避免每次请求重新解析、编译代码（可提升50%+性能），需在 `php.ini`中配置启用。
     - **使用缓存系统**：将频繁访问的数据（如用户信息、配置项）缓存到Redis/Memcached，减少数据库查询或复杂计算（缓存失效策略需合理设计）。
     - **优化PHP-FPM配置**：根据服务器内存调整 `pm.max_children`（最大进程数）、`pm.start_servers`（启动进程数），避免进程过多导致内存溢出或过少导致请求排队。
     - **启用HTTP缓存**：通过Nginx/Apache配置静态资源（JS/CSS/图片）的 `Cache-Control`和 `Expires`头，让浏览器缓存资源，减少重复请求。
     - **使用CDN分发静态资源**：将图片、JS、CSS等静态文件部署到CDN，利用边缘节点加速访问，减轻源服务器压力。
- 示例代码与配置：

```php
<?php
// 1. 代码层面：自动加载替代多次include
// 注册自动加载函数（composer项目默认支持）
spl_autoload_register(function ($className) {
    $prefix = 'App\\';
    $baseDir = __DIR__ . '/src/';
    $len = strlen($prefix);
    if (strncmp($prefix, $className, $len) !== 0) {
        return;
    }
    $relativeClass = substr($className, $len);
    $file = $baseDir . str_replace('\\', '/', $relativeClass) . '.php';
    if (file_exists($file)) {
        require $file;
    }
});
// 使用时自动加载，无需手动include
$user = new App\Models\User();


// 2. 代码层面：缓存数据库查询结果
function getUserInfo($userId) {
    static $cache = []; // 静态变量缓存结果
    if (isset($cache[$userId])) {
        return $cache[$userId];
    }
    // 仅首次查询数据库
    $pdo = new PDO('mysql:host=localhost;dbname=test', 'user', 'pass');
    $stmt = $pdo->prepare("SELECT id, name FROM users WHERE id = :id");
    $stmt->execute([':id' => $userId]);
    $user = $stmt->fetch(PDO::FETCH_ASSOC);
    $cache[$userId] = $user; // 存入缓存
    return $user;
}


// 3. 代码层面：输出压缩
ob_start('ob_gzhandler'); // 开启Gzip压缩
echo '<html><body>大量HTML内容...</body></html>';
ob_end_flush();


// 4. 服务器层面：OPcache配置（php.ini）
/*
[opcache]
zend_extension=opcache.so
opcache.enable=1                  // 启用OPcache
opcache.memory_consumption=128    // 分配128MB内存
opcache.interned_strings_buffer=8 // 字符串缓存8MB
opcache.max_accelerated_files=4000 // 最大缓存文件数
opcache.validate_timestamps=1     // 生产环境可设为0（关闭文件修改检查）
opcache.revalidate_freq=60        // 检查文件修改的间隔（秒）
*/


// 5. 服务器层面：Redis缓存示例
$redis = new Redis();
$redis->connect('127.0.0.1', 6379);

$key = 'hot_articles';
// 尝试从Redis获取缓存
$articles = $redis->get($key);
if (!$articles) {
    // 缓存未命中，查询数据库
    $pdo = new PDO('mysql:host=localhost;dbname=test', 'user', 'pass');
    $stmt = $pdo->query("SELECT * FROM articles WHERE is_hot = 1 LIMIT 10");
    $articles = $stmt->fetchAll(PDO::FETCH_ASSOC);
    // 存入缓存（设置10分钟过期）
    $redis->setex($key, 600, json_encode($articles));
} else {
    $articles = json_decode($articles, true);
}
```

- 补充说明：优化需结合实际场景（如高并发API侧重缓存和服务器配置，静态页面侧重CDN和HTTP缓存），可通过 `Xdebug`、`Blackfire`等工具分析性能瓶颈，针对性优化。

</details>

## 4. 什么是缓存穿透？如何通过 PHP 代码 + Redis 简单解决缓存穿透问题（如缓存空值）？

<details>
<summary>答案与解析（含缓存穿透、击穿、雪崩的区别与解决）</summary>

- 思路要点：

  1. **缓存穿透**：

     - 定义：请求的数据在缓存和数据库中均不存在，导致每次请求都穿透缓存直接访问数据库（如查询不存在的ID、恶意攻击）。
     - 影响：大量无效请求直击数据库，可能导致数据库过载宕机。
     - 解决方法：
       - 缓存空值：数据库查询为空时，在缓存中存入空值（如 `NULL`）并设置短期过期时间（如60秒），避免重复查库。
       - 布隆过滤器：预先将所有可能存在的有效key存入布隆过滤器，请求时先通过过滤器判断是否可能存在，不存在则直接拦截。
  2. **缓存击穿**：

     - 定义：某个热点key（被高频访问的key）在缓存中过期的瞬间，大量并发请求同时访问该key，导致所有请求都穿透到数据库查询。
     - 影响：热点key对应的数据库数据被瞬间高并发访问，可能导致数据库瞬间压力激增。
     - 解决方法：
       - 互斥锁（分布式锁）：缓存失效时，只有一个请求能获取锁并查询数据库，其他请求等待锁释放后从缓存获取数据（如用Redis的 `setnx`实现）。
       - 热点key永不过期：核心热点key不设置过期时间，通过后台异步更新缓存（避免主动过期）。
       - 提前预热：在热点key过期前，通过定时任务主动更新缓存，避免过期瞬间的请求穿透。
  3. **缓存雪崩**：

     - 定义：大量缓存key在同一时间点过期，或缓存服务宕机，导致所有请求集中涌向数据库，造成数据库雪崩式压力。
     - 影响：数据库承受远超日常的并发请求，可能直接宕机，引发整个系统连锁故障。
     - 解决方法：
       - 过期时间加随机值：为缓存key的过期时间增加随机偏移量（如 `3600 ± 60`秒），避免大量key同时过期。
       - 多级缓存：结合本地缓存（如PHP的 `static`变量、APC）和分布式缓存（Redis），即使分布式缓存失效，本地缓存可临时承接部分请求。
       - 服务降级/熔断：缓存失效时，通过熔断器（如Sentinel）限制访问数据库的请求量，返回默认数据（如“系统繁忙，请稍后再试”）。
       - 缓存集群高可用：部署Redis集群（主从+哨兵），避免单节点故障导致缓存服务整体不可用。
  4. **三者核心区别**：

     | 类型     | 核心原因                         | 影响范围                | 解决核心思路                       |
     | -------- | -------------------------------- | ----------------------- | ---------------------------------- |
     | 缓存穿透 | 请求不存在的数据（缓存和DB均无） | 单类无效请求持续冲击DB  | 拦截无效请求（空值缓存、布隆过滤） |
     | 缓存击穿 | 热点key过期瞬间的高并发请求      | 单个热点key对应的DB数据 | 控制并发（互斥锁）、避免过期       |
     | 缓存雪崩 | 大量key同时过期或缓存服务故障    | 整个DB甚至系统承受压力  | 分散过期时间、提升缓存可用性       |
- 示例代码：

  ```php
  <?php
  $redis = new Redis();
  $redis->connect('127.0.0.1', 6379);


  // 1. 缓存穿透解决（缓存空值）
  function getProduct($productId, Redis $redis) {
      $key = "product:{$productId}";
      $cache = $redis->get($key);

      if ($cache !== false) {
          return $cache === 'NULL' ? null : json_decode($cache, true);
      }

      // 查数据库
      $product = queryProductFromDb($productId);
      if ($product) {
          $redis->setex($key, 3600, json_encode($product)); // 有效数据缓存1小时
      } else {
          $redis->setex($key, 60, 'NULL'); // 空值缓存60秒
      }
      return $product;
  }


  // 2. 缓存击穿解决（互斥锁）
  function getHotProduct($productId, Redis $redis) {
      $key = "hot_product:{$productId}";
      $lockKey = "lock:{$productId}";

      // 先查缓存
      $cache = $redis->get($key);
      if ($cache !== false) {
          return json_decode($cache, true);
      }

      // 缓存失效，尝试获取锁
      $lock = $redis->setnx($lockKey, 1);
      $redis->expire($lockKey, 10); // 锁10秒过期，避免死锁

      if ($lock) {
          // 获得锁，查数据库并更新缓存
          $product = queryProductFromDb($productId);
          $redis->setex($key, 3600 + rand(0, 60), json_encode($product)); // 加随机过期时间
          $redis->del($lockKey); // 释放锁
          return $product;
      } else {
          // 未获得锁，等待后重试
          sleep(0.1);
          return getHotProduct($productId, $redis);
      }
  }


  // 3. 缓存雪崩解决（过期时间加随机值）
  function setCacheWithRandomExpire($key, $data, $baseExpire = 3600) {
      global $redis;
      // 过期时间 = 基础时间 + 随机偏移量（0-300秒），避免同时过期
      $expire = $baseExpire + rand(0, 300);
      $redis->setex($key, $expire, json_encode($data));
  }

  // 示例：批量设置缓存时添加随机过期时间
  $products = [/* 从数据库查询的批量商品数据 */];
  foreach ($products as $p) {
      setCacheWithRandomExpire("product:{$p['id']}", $p);
  }


  // 模拟数据库查询
  function queryProductFromDb($id) {
      // 实际项目中为数据库查询逻辑
      return $id > 0 && $id < 1000 ? ['id' => $id, 'name' => "商品{$id}"] : null;
  }
  ?>
  ```

</details>

## 5. 文件上传功能中，需要做哪些安全校验（如文件类型、文件大小、文件内容检测）？请用 PHP 代码示例说明关键校验逻辑

<details>
<summary>答案与解析</summary>

- 思路要点：
  文件上传是PHP项目中高风险功能之一，需通过多层校验防止恶意文件（如PHP脚本、病毒）上传。核心安全校验包括：

  1. **上传错误检查**：验证文件是否成功上传（排除临时错误、文件过大等系统级错误）。
  2. **文件大小限制**：同时校验PHP配置限制和业务层面限制（如最大5MB）。
  3. **文件类型校验**：
     - 禁止依赖文件后缀（易篡改），需结合MIME类型（通过 `finfo`获取实际内容类型）。
     - 限制允许的类型（如仅允许 `image/jpeg`、`image/png`、`application/pdf`）。
  4. **文件内容检测**：
     - 图片文件：通过GD库/Imagick尝试解析，验证是否为有效图片（防止伪装成图片的脚本）。
     - 通用文件：检查文件头特征（如JPG文件头为 `FFD8FF`）。
  5. **文件名安全处理**：
     - 过滤特殊字符（如 `../`防止路径遍历攻击）。
     - 重命名为随机字符串（避免同名覆盖、暴露原文件名）。
  6. **存储路径安全**：将文件存储在Web访问目录外，或通过脚本间接读取（禁止直接访问上传文件）。
- 示例代码：

```php
<?php
// 配置：允许的文件类型（MIME类型）和后缀
$allowedMimeTypes = [
    'image/jpeg',
    'image/png',
    'application/pdf'
];
$allowedExtensions = ['jpg', 'jpeg', 'png', 'pdf'];
$maxFileSize = 5 * 1024 * 1024; // 5MB
$uploadDir = '/var/www/private_uploads/'; // 存储在Web目录外（关键！）

// 检查上传是否有错误
if ($_FILES['file']['error'] !== UPLOAD_ERR_OK) {
    $errors = [
        UPLOAD_ERR_INI_SIZE => '文件超过PHP配置的upload_max_filesize',
        UPLOAD_ERR_FORM_SIZE => '文件超过表单指定的MAX_FILE_SIZE',
        UPLOAD_ERR_PARTIAL => '文件仅部分上传',
        UPLOAD_ERR_NO_FILE => '未上传文件',
        UPLOAD_ERR_NO_TMP_DIR => '缺少临时文件夹',
        UPLOAD_ERR_CANT_WRITE => '文件写入失败',
    ];
    die('上传错误：' . ($errors[$_FILES['file']['error']] ?? '未知错误'));
}

// 1. 校验文件大小
if ($_FILES['file']['size'] > $maxFileSize) {
    die('文件过大，最大允许5MB');
}

// 2. 校验文件后缀（辅助，不能单独依赖）
$originalName = $_FILES['file']['name'];
$extension = strtolower(pathinfo($originalName, PATHINFO_EXTENSION));
if (!in_array($extension, $allowedExtensions)) {
    die('不允许的文件后缀，仅支持' . implode(', ', $allowedExtensions));
}

// 3. 校验MIME类型（通过文件内容获取，更可靠）
$finfo = new finfo(FILEINFO_MIME_TYPE);
$mimeType = $finfo->file($_FILES['file']['tmp_name']);
if (!in_array($mimeType, $allowedMimeTypes)) {
    die('不允许的文件类型，实际类型：' . $mimeType);
}

// 4. 内容检测（以图片为例，验证是否为有效图片）
if (in_array($mimeType, ['image/jpeg', 'image/png'])) {
    $image = @imagecreatefromstring(file_get_contents($_FILES['file']['tmp_name']));
    if (!$image) {
        die('无效的图片文件（可能是伪装的恶意文件）');
    }
    imagedestroy($image); // 释放资源
}

// 5. 处理文件名（防止路径遍历和同名覆盖）
$safeFileName = uniqid() . '.' . $extension; // 随机文件名
$destination = $uploadDir . $safeFileName;

// 6. 移动文件到安全目录
if (!move_uploaded_file($_FILES['file']['tmp_name'], $destination)) {
    die('文件移动失败');
}

// 7. 限制文件权限（仅允许读取，禁止执行）
chmod($destination, 0644);

echo '文件上传成功，存储路径：' . $destination;
?>
```

- 关键说明：
  - 单一校验易被绕过（如仅校验后缀可被篡改），需组合多层校验（后缀+MIME+内容）。
  - 存储路径必须脱离Web根目录（如 `/var/www/html`外），避免攻击者直接访问上传的恶意脚本。
  - 随机重命名文件名可防止“路径遍历攻击”（如上传文件名为 `../../../../var/www/html/shell.php`）。
  - 对于高安全性场景，可结合第三方工具（如ClamAV）进行病毒扫描。

</details>