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

## 6. Redis 的 RDB 和 AOF 两种持久化机制分别是什么？它们的工作原理有什么区别？如何根据业务场景选择合适的持久化方案？

<details>
<summary>答案与解析</summary>

- 思路要点：Redis 持久化的核心是避免内存数据因服务宕机丢失，RDB 以“快照”形式存储全量数据，AOF 以“日志”形式记录增量命令，需从“数据安全性”“性能开销”“恢复速度”三个维度对比，结合业务对数据完整性的要求选择方案。

### 一、RDB 持久化（Redis Database）
#### 1. 定义
RDB 是 Redis 的**快照式持久化**机制，通过定时生成内存数据的完整二进制快照文件（`dump.rdb`），将某一时刻的全量数据持久化到磁盘，服务重启时加载该文件恢复数据。

#### 2. 工作原理
- **核心流程**：
  1. 触发 RDB 时，Redis 执行 `fork()` 创建子进程（父进程继续处理客户端请求，子进程负责生成快照，避免阻塞）；
  2. 子进程遍历内存中的数据结构，将数据写入临时 RDB 文件；
  3. 写入完成后，临时文件原子替换旧 RDB 文件（保证文件完整性，避免中断导致损坏）；
  4. 子进程退出，快照生成完成。
- **触发方式**：
  - 自动触发：通过 `redis.conf` 配置规则，如 `save 900 1`（900秒内有1次写入）、`save 300 10`（300秒内10次写入）；
  - 手动触发：执行 `BGSAVE`（后台异步生成，推荐生产环境）或 `SAVE`（阻塞主进程，仅用于紧急备份）；
  - 特殊触发：执行 `FLUSHALL`（清空数据时生成空快照）、主从复制时主节点向从节点发送 RDB 文件。


### 二、AOF 持久化（Append Only File）
#### 1. 定义
AOF 是 Redis 的**日志式持久化**机制，通过实时记录所有写操作命令（如 `SET`、`HSET`）到 `appendonly.aof` 日志文件，服务重启时通过重新执行文件中的命令恢复数据。

#### 2. 工作原理
- **核心流程**：
  1. 客户端执行写操作时，Redis 先将命令追加到内存中的 AOF 缓冲区（减少磁盘 IO 频率）；
  2. 根据 `appendfsync` 配置的刷盘策略，将缓冲区内容写入磁盘中的 AOF 文件；
  3. 随着命令累积，AOF 文件会膨胀，Redis 通过“**AOF 重写**”合并重复命令（如多次 `SET user:1 zhangsan` 合并为1次），压缩文件体积；
  4. 服务重启时，按顺序执行 AOF 文件中的命令，恢复数据。
- **关键配置**：
  - 刷盘策略：`appendfsync everysec`（默认，每秒刷盘1次，最多丢失1秒数据）、`appendfsync always`（每写1次刷盘，安全性最高但性能差）、`appendfsync no`（由操作系统决定刷盘，性能最好但安全性最低）；
  - 重写触发：`auto-aof-rewrite-percentage 100`（文件比上次重写后增长100%）且 `auto-aof-rewrite-min-size 64mb`（文件超64MB）时自动触发，或手动执行 `BGREWRITEAOF`。


### 三、RDB 与 AOF 的核心区别
| 对比维度        | RDB 持久化                  | AOF 持久化                  |
|-----------------|-----------------------------|-----------------------------|
| 存储内容        | 全量二进制快照              | 增量写操作命令（文本格式）  |
| 数据完整性      | 低（快照间隔内数据丢失，如间隔5分钟丢5分钟数据） | 高（最多丢失1秒数据，`always` 策略零丢失） |
| 文件大小        | 小（二进制压缩存储）        | 大（需重写压缩）            |
| 恢复速度        | 快（直接加载二进制数据）    | 慢（需逐行执行命令）        |
| 性能影响        | 低（仅快照时消耗 CPU/内存，间隔内无影响） | 中（实时追加命令，刷盘策略影响性能） |


### 四、持久化方案选择策略
#### 1. 仅用 RDB
- 适用场景：**性能优先、数据完整性要求低**的业务，如非核心缓存（热点商品列表、临时会话）、允许丢失几分钟数据的场景；
- 优势：恢复速度快，磁盘占用小，对 Redis 运行性能影响小；
- 风险：宕机时丢失快照间隔内的数据，若 RDB 文件损坏可能导致数据无法恢复。

#### 2. 仅用 AOF
- 适用场景：**数据完整性要求高**的业务，如金融交易记录、用户核心信息（手机号、身份证号），不允许丢失超过1秒数据；
- 优势：数据安全性高，AOF 文件可读性强（可手动修改错误命令恢复数据）；
- 风险：AOF 文件体积大，恢复速度慢，重写过程会消耗 CPU/内存。

#### 3. RDB + AOF 混合模式（推荐核心业务）
- 适用场景：**兼顾性能与数据安全性**的业务，如电商订单缓存、支付会话；
- 工作原理：
  1. 日常通过 AOF 记录实时写命令（保证数据完整性，丢失≤1秒数据）；
  2. 定期生成 RDB 快照（如每6小时1次，用于快速恢复）；
  3. 服务重启时，优先加载 AOF 文件（数据更完整），若 AOF 文件不存在则加载 RDB 文件；
- 优势：避免单一持久化的缺陷，既保证数据安全，又兼顾恢复速度。


### 五、示例配置与操作
#### 1. RDB 核心配置（`redis.conf`）
```ini
# 自动触发快照规则（满足任一即触发）
save 900 1
save 300 10
save 60 10000

# RDB 文件存储路径与名称
dir ./
dbfilename dump.rdb

# 快照生成失败时，停止接收写操作（避免数据不一致）
stop-writes-on-bgsave-error yes

# 启用 LZF 压缩 RDB 文件（节省磁盘空间，轻微消耗 CPU）
rdbcompression yes
```

#### 2. AOF 核心配置（`redis.conf`）
```ini
# 启用 AOF 持久化（默认关闭，需手动开启）
appendonly yes

# AOF 文件名称
appendfilename "appendonly.aof"

# 刷盘策略（推荐 everysec）
appendfsync everysec

# AOF 重写配置
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# AOF 文件损坏时，拒绝加载（避免脏数据）
aof-load-truncated no
```

#### 3. 混合模式验证
```bash
# 1. 开启 AOF + RDB（确保 redis.conf 中 appendonly yes 且保留 save 规则）
127.0.0.1:6379> SET user:1 zhangsan
OK

# 2. 手动触发 RDB 快照
127.0.0.1:6379> BGSAVE
Background saving started

# 3. 查看 AOF 文件（包含 SET 命令）
cat appendonly.aof
# 输出示例：*3\r\n$3\r\nSET\r\n$6\r\nuser:1\r\n$8\r\nzhangsan\r\n

# 4. 查看 RDB 文件（生成 dump.rdb）
ls -l dump.rdb
```

</details>


## 7. PHP 服务端如何配置 HTTPS？为什么接口通信必须使用 HTTPS？如何强制所有接口请求自动跳转至 HTTPS？

<details>
<summary>答案与解析</summary>

- 思路要点：HTTPS 的核心是在 HTTP 基础上添加 TLS/SSL 加密层，解决 HTTP 明文传输的安全风险。配置需经历“证书申请→服务器配置→PHP 适配”三步，强制跳转需通过服务器规则或 PHP 代码拦截 HTTP 请求，需先明确 HTTPS 的必要性，再提供可落地的配置方案。

### 一、为什么接口通信必须使用 HTTPS？
HTTP 协议存在三大安全缺陷，HTTPS 通过 **TLS/SSL 协议** 弥补这些缺陷，是接口通信的安全底线：

| 风险类型       | HTTP 问题描述                                  | HTTPS 解决方案                                  |
|----------------|-----------------------------------------------|-----------------------------------------------|
| **数据泄露**   | 所有数据（密码、Token、手机号）明文传输，中间节点（路由器、运营商）可窃听 | 采用 AES 对称加密传输数据，仅客户端与服务器持有密钥 |
| **数据篡改**   | 攻击者可修改传输中的数据（如篡改订单金额），且无校验机制 | 采用 SHA-256 消息摘要 + 数字签名，验证数据完整性 |
| **身份冒充**   | 攻击者可伪装成服务器接收请求（如钓鱼网站），客户端无法识别 | 服务器配置 CA 颁发的数字证书，客户端验证证书合法性 |

**核心结论**：只要接口涉及“用户隐私数据”“敏感业务操作”（登录、支付、订单提交），必须使用 HTTPS，否则会导致数据泄露、业务篡改、身份伪造等严重安全问题。


### 二、PHP 服务端配置 HTTPS 的完整流程
HTTPS 配置的核心是“服务器（Nginx/Apache）挂载 SSL 证书”，PHP 本身无需特殊编译，但需适配 HTTPS 环境（如识别 HTTPS 请求、设置安全 Cookie）。

#### 步骤1：申请 SSL 证书
SSL 证书是 HTTPS 的“身份凭证”，需向 CA（证书颁发机构）申请，分为免费和付费两种：
- **免费证书**：适合个人项目、测试环境，如 Let's Encrypt（有效期90天，支持自动续期）、阿里云/腾讯云免费证书；
- **付费证书**：适合生产环境，如 EV 证书（地址栏显示绿色企业名称，安全性最高）、OV 证书（验证企业身份），价格几百至几千元/年。

**Let's Encrypt 申请示例（CentOS）**：
```bash
# 1. 安装 Certbot 工具（自动申请与续期）
yum install certbot python3-certbot-nginx -y

# 2. 生成证书（自动验证域名所有权，需开放80端口）
certbot --nginx -d yourdomain.com -d www.yourdomain.com

# 3. 证书默认存储路径
# 证书链：/etc/letsencrypt/live/yourdomain.com/fullchain.pem
# 私钥：/etc/letsencrypt/live/yourdomain.com/privkey.pem（绝对不能泄露）
```

#### 步骤2：服务器配置 HTTPS（Nginx 为例，最常用）
修改 Nginx 配置文件（通常路径：`/etc/nginx/conf.d/yourdomain.conf`），核心是“监听443端口+挂载证书+启用 TLS 协议”：
```nginx
# HTTPS 服务器配置（监听443端口，HTTPS 默认端口）
server {
    listen 443 ssl;
    server_name yourdomain.com www.yourdomain.com; # 你的域名

    # 1. 挂载 SSL 证书
    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem; # 证书链
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem; # 私钥

    # 2. TLS 协议优化（禁用不安全协议，仅保留 TLS1.2/TLS1.3）
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on; # 优先使用服务器端加密套件
    ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH"; # 安全加密套件

    # 3. 证书缓存优化（减少 TLS 握手开销）
    ssl_session_cache shared:SSL:10m; # 共享缓存，10MB
    ssl_session_timeout 10m; # 会话超时时间

    # 4. PHP 项目代理配置（转发请求到 PHP-FPM）
    root /var/www/your-php-project; # PHP 项目根目录
    index index.php;

    location ~ \.php$ {
        fastcgi_pass 127.0.0.1:9000; # PHP-FPM 地址（若用 Unix 套接字：unix:/var/run/php-fpm/www.sock）
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;

        # 关键：告诉 PHP 当前请求是 HTTPS（否则 PHP 会误认为是 HTTP）
        fastcgi_param HTTPS on;
    }

    # 5. 静态资源缓存（可选，优化性能）
    location ~* \.(js|css|png|jpg|jpeg|gif)$ {
        expires 30d; # 浏览器缓存30天
    }
}
```

#### 步骤3：PHP 适配 HTTPS 环境
PHP 需正确识别 HTTPS 请求，避免出现“HTTPS 环境下认为是 HTTP”的问题：
1. **判断当前请求是否为 HTTPS**：
   ```php
   /**
    * 可靠判断 HTTPS 请求（兼容反向代理场景，如 Cloudflare）
    */
   function isHttps(): bool {
       // 1. 直接 HTTPS 访问（通过 $_SERVER['HTTPS']）
       if (isset($_SERVER['HTTPS']) && strtolower($_SERVER['HTTPS']) === 'on') {
           return true;
       }
       // 2. 反向代理场景（通过 X-Forwarded-Proto 头）
       if (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') {
           return true;
       }
       // 3. 端口判断（443 是 HTTPS 默认端口）
       return $_SERVER['SERVER_PORT'] === 443;
   }
   ```

2. **设置安全 Cookie 属性**：HTTPS 环境下，Cookie 需添加 `secure`（仅 HTTPS 传输）和 `httponly`（禁止 JS 访问），防止 Cookie 被窃取：
   ```php
   // 示例：设置登录 Token Cookie（有效期1小时）
   setcookie(
       'user_token',
       'abc123xyz789',
       time() + 3600,
       '/', // 作用域：根目录
       'yourdomain.com', // 域名
       true, // secure：仅 HTTPS 传输
       true  // httponly：禁止 JS 读取，防 XSS 窃取
   );
   ```


### 三、强制所有接口请求自动跳转至 HTTPS
若用户访问 HTTP 地址（如 `http://yourdomain.com/api/user`），需自动跳转至 HTTPS，推荐优先用服务器配置（性能更优），其次用 PHP 代码（灵活性高）。

#### 方案1：Nginx 配置强制跳转（推荐）
添加 HTTP 服务器块（监听80端口），通过 `301 永久重定向` 将所有 HTTP 请求跳转至 HTTPS：
```nginx
# HTTP 服务器块（仅用于跳转，不处理业务）
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    # 方案A：所有请求跳转至 HTTPS（推荐）
    return 301 https://$host$request_uri;

    # 方案B：仅接口路径跳转（如 /api 开头的请求）
    # location /api {
    #     return 301 https://$host$request_uri;
    # }
}
```
- 说明：`$host` 是当前请求的域名，`$request_uri` 是完整请求路径（如 `/api/user?id=1`），确保跳转后路径不变。

#### 方案2：PHP 代码强制跳转（适合无法修改服务器配置的场景）
在 PHP 入口文件（如 `index.php`）最顶部添加跳转逻辑，拦截 HTTP 请求：
```php
<?php
/**
 * PHP 代码强制 HTTPS 跳转（入口文件 index.php 顶部）
 */
function forceHttps() {
    // 1. 判断是否为 HTTPS 请求
    $isHttps = (isset($_SERVER['HTTPS']) && strtolower($_SERVER['HTTPS']) === 'on')
        || (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https');

    // 2. 本地开发环境（127.0.0.1/::1）跳过跳转，避免影响开发
    $isLocal = in_array($_SERVER['REMOTE_ADDR'], ['127.0.0.1', '::1']);

    if (!$isHttps && !$isLocal) {
        // 3. 构建 HTTPS 地址并跳转（301 永久重定向，利于 SEO）
        $httpsUrl = 'https://' . $_SERVER['HTTP_HOST'] . $_SERVER['REQUEST_URI'];
        header('Location: ' . $httpsUrl, true, 301);
        exit(); // 必须 exit，避免后续代码执行
    }
}

// 执行强制跳转
forceHttps();

// 后续业务代码...
require_once 'vendor/autoload.php';
// ...
```


### 关键说明
1. **证书续期**：免费证书（如 Let's Encrypt）有效期短，需配置自动续期，添加定时任务：
   ```bash
   # 每月1号凌晨自动续期，--quiet 静默执行
   echo "0 0 1 * * certbot renew --quiet --nginx" >> /var/spool/cron/root
   ```
2. **TLS 安全性检测**：可通过 [SSL Labs](https://www.ssllabs.com/ssltest/) 检测证书配置，确保禁用 TLS1.0/TLS1.1（存在安全漏洞）；
3. **反向代理注意**：若使用 Cloudflare、Nginx 反向代理，需在代理服务器配置中添加 `proxy_set_header X-Forwarded-Proto $scheme;`，否则 PHP 无法正确识别 HTTPS。

</details>


## 8. 用户敏感数据（如手机号、身份证号）在 PHP 服务端应如何安全存储与传输？常用的加密算法（如 AES）在 PHP 中如何实现？

<details>
<summary>答案与解析</summary>

- 思路要点：敏感数据安全的核心是“传输不泄露、存储不明文”，传输依赖 HTTPS 保障链路安全，存储依赖可逆加密算法（AES）保障数据本身安全。需明确“传输安全”与“存储安全”的不同方案，掌握 AES-256-CBC 在 PHP 中的标准实现，规避“硬编码密钥”“忽略 IV 向量”等常见错误。

### 一、敏感数据的安全传输方案
传输过程需确保数据不被窃听、篡改，核心依赖 HTTPS，同时配合细节优化。

#### 1. 基础：强制 HTTPS 传输
- 所有包含敏感数据的请求（登录、提交身份证号、支付）必须通过 HTTPS 发起，禁止 HTTP（参考第7题的强制跳转方案）；
- 敏感数据通过 **请求体（POST 表单/JSON）** 传输，禁止在 URL 中携带（URL 会被浏览器缓存、服务器日志记录，存在泄露风险）。

#### 2. 进阶：请求签名与防重放（高安全场景）
为防止传输中数据被篡改或重复提交（重放攻击），可添加“请求签名”和“时间戳+随机串”：
```php
/**
 * 客户端：生成请求签名（防止篡改）
 */
$transportSecret = 'your-transport-secret-key'; // 客户端与服务器约定的密钥（不传输）
$data = [
    'phone' => '13800138000',
    'timestamp' => time(), // 时间戳（防重放）
    'nonce' => bin2hex(random_bytes(8)) // 随机串（防重放，每次请求不同）
];

// 1. 按 key 排序（避免参数顺序影响签名）
ksort($data);
// 2. 拼接为查询字符串（如 "nonce=abc&phone=13800138000&timestamp=1693456789"）
$signStr = http_build_query($data);
// 3. HMAC-SHA256 生成签名
$data['signature'] = hash_hmac('sha256', $signStr, $transportSecret);

// 4. 发送请求（通过 POST 体传输 data）
$ch = curl_init('https://yourdomain.com/api/submit-phone');
curl_setopt($ch, CURLOPT_POST, true);
curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type: application/json']);
curl_exec($ch);


/**
 * 服务器：验证请求签名（防止篡改）与防重放
 */
function verifyRequest($requestData, $transportSecret) {
    // 1. 校验签名参数
    if (!isset($requestData['signature'])) {
        throw new Exception('签名缺失');
    }
    $signature = $requestData['signature'];
    unset($requestData['signature']); // 移除签名，重新计算

    // 2. 防重放：校验时间戳（请求需在5分钟内）
    $now = time();
    if (abs($now - $requestData['timestamp']) > 300) {
        throw new Exception('请求已过期');
    }

    // 3. 防重放：校验随机串（Redis 记录已使用的 nonce，1小时内不重复）
    $redis = new Redis();
    $redis->connect('127.0.0.1', 6379);
    $nonceKey = 'request:nonce:' . $requestData['nonce'];
    if ($redis->exists($nonceKey)) {
        throw new Exception('重复请求');
    }
    $redis->setex($nonceKey, 3600, 1); // 记录 nonce，1小时过期

    // 4. 验证签名（用 hash_equals 避免时序攻击）
    ksort($requestData);
    $signStr = http_build_query($requestData);
    $calcSignature = hash_hmac('sha256', $signStr, $transportSecret);
    if (!hash_equals($calcSignature, $signature)) {
        throw new Exception('签名验证失败');
    }

    return true;
}
```


### 二、敏感数据的安全存储方案
存储需确保“即使数据库泄露，也无法直接获取原数据”，核心用 **AES-256-CBC 可逆加密算法**（需恢复原数据，如展示手机号、实名认证），禁止明文存储或弱加密（如 Base64，仅编码非加密）。

#### 1. 加密算法选择：AES-256-CBC
- **AES**：高级加密标准，对称加密算法（加密/解密用同一密钥），安全性高、性能好，适合敏感数据存储；
- **密钥长度**：256 位（需 32 字节密钥），比 128 位更安全，PHP `openssl` 扩展默认支持；
- **模式选择**：CBC 模式（需 IV 向量），比 ECB 模式更安全（ECB 模式相同明文加密结果相同，易被破解）；
- **补位方式**：PKCS#7 补位（当明文长度不是 AES 块大小的整数倍时，补充对应字节，如块大小16字节，缺3字节则补3个 `0x03`）。

#### 2. AES-256-CBC 在 PHP 中的标准实现
依赖 PHP `openssl` 扩展（默认启用），核心是“安全生成密钥/IV 向量”“加密/解密函数封装”“密钥安全管理”。

##### （1）核心加密/解密工具函数
```php
/**
 * AES-256-CBC 加密（PKCS#7 补位）
 * @param string $plaintext 待加密的明文（敏感数据，如手机号、身份证号）
 * @param string $key 加密密钥（必须 32 字节，256 位）
 * @param string $iv IV 向量（必须 16 字节，每次加密生成新的）
 * @return string 加密后的 base64 字符串（便于存储，避免乱码）
 * @throws Exception 若 openssl 扩展未启用或参数错误
 */
function aesEncrypt(string $plaintext, string $key, string $iv): string {
    // 校验依赖与参数
    if (!extension_loaded('openssl')) {
        throw new Exception('AES 加密需启用 openssl 扩展');
    }
    if (strlen($key) !== 32) {
        throw new Exception('AES-256 密钥必须 32 字节');
    }
    if (strlen($iv) !== 16) {
        throw new Exception('AES-CBC 模式 IV 向量必须 16 字节');
    }

    // 加密（OPENSSL_RAW_DATA 输出原始二进制，而非 base64）
    $ciphertext = openssl_encrypt(
        $plaintext,
        'aes-256-cbc',
        $key,
        OPENSSL_RAW_DATA, // 关键参数：输出二进制
        $iv
    );

    // 二进制转 base64（便于存储到数据库，避免乱码）
    return base64_encode($ciphertext);
}

/**
 * AES-256-CBC 解密
 * @param string $ciphertext 加密后的 base64 字符串
 * @param string $key 解密密钥（与加密密钥相同，32 字节）
 * @param string $iv IV 向量（与加密 IV 相同，16 字节）
 * @return string 解密后的明文
 * @throws Exception 若解密失败或参数错误
 */
function aesDecrypt(string $ciphertext, string $key, string $iv): string {
    if (!extension_loaded('openssl')) {
        throw new Exception('AES 解密需启用 openssl 扩展');
    }
    if (strlen($key) !== 32) {
        throw new Exception('AES-256 密钥必须 32 字节');
    }
    if (strlen($iv) !== 16) {
        throw new Exception('AES-CBC 模式 IV 向量必须 16 字节');
    }

    // 1. base64 解码为原始二进制
    $ciphertextRaw = base64_decode($ciphertext);
    if ($ciphertextRaw === false) {
        throw new Exception('加密字符串 base64 解码失败');
    }

    // 2. 解密（自动去除 PKCS#7 补位）
    $plaintext = openssl_decrypt(
        $ciphertextRaw,
        'aes-256-cbc',
        $key,
        OPENSSL_RAW_DATA,
        $iv
    );

    if ($plaintext === false) {
        throw new Exception('AES 解密失败（密钥或 IV 错误）');
    }

    return $plaintext;
}
```

##### （2）密钥与 IV 向量的安全管理
- **密钥生成**：必须是随机、足够长的字符串，推荐用 `random_bytes()` 生成（安全的随机数），仅生成1次，妥善保存：
  ```php
  // 生成 32 字节 AES-256 密钥（仅生成1次）
  $aesKey = random_bytes(32);
  // 转为 base64 便于存储（如存储到环境变量，不写在代码中）
  echo base64_encode($aesKey); 
  // 输出示例：X7Z8y9aBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFg
  ```

- **IV 向量生成**：每次加密都需生成唯一的 IV 向量（用 `random_bytes(16)`），IV 无需保密，但需与密文一起存储（如数据库同一字段或关联字段）；

- **密钥存储**：禁止硬编码到代码，推荐存储在 **环境变量**（如 Linux `.env` 文件、Docker 环境变量）或 **密钥管理服务**（如阿里云 KMS）：
  ```php
  // 从环境变量加载密钥（Laravel 等框架可直接用 env() 函数）
  $aesKey = base64_decode(getenv('AES_SECRET_KEY')); // 环境变量中存储 base64 编码的密钥
  ```

##### （3）实际使用示例（存储手机号）
```php
<?php
try {
    // 1. 从环境变量加载 AES 密钥（生产环境推荐）
    $aesKey = base64_decode(getenv('AES_SECRET_KEY'));

    // 2. 待加密的敏感数据（手机号）
    $plainPhone = '13800138000';

    // 3. 生成唯一 IV 向量（每次加密都生成新的）
    $iv = random_bytes(16);

    // 4. 加密手机号
    $encryptedPhone = aesEncrypt($plainPhone, $aesKey, $iv);

    // 5. 存储到数据库（需同时存储 IV 向量，否则无法解密）
    $pdo = new PDO('mysql:host=localhost;dbname=user_db;charset=utf8mb4', 'db_user', 'db_pass');
    $stmt = $pdo->prepare("INSERT INTO users (phone_encrypted, phone_iv, create_time) VALUES (?, ?, NOW())");
    $stmt->execute([$encryptedPhone, base64_encode($iv)]); // IV 转 base64 存储

    // 6. 从数据库查询并解密
    $stmt = $pdo->prepare("SELECT phone_encrypted, phone_iv FROM users WHERE id = ?");
    $stmt->execute([1]); // 假设用户 ID 为 1
    $user = $stmt->fetch(PDO::FETCH_ASSOC);

    $decryptedPhone = aesDecrypt(
        $user['phone_encrypted'],
        $aesKey,
        base64_decode($user['phone_iv']) // IV 解码
    );

    echo "解密后的手机号：" . $decryptedPhone; // 输出：13800138000
} catch (Exception $e) {
    echo "错误：" . $e->getMessage();
}
```


### 三、补充：敏感数据脱敏展示
解密后无需全量展示（如仅需验证手机号归属，无需显示完整号码），可通过“脱敏”保护数据：
```php
/**
 * 手机号脱敏（显示前3位+后4位，中间用*代替）
 */
function maskPhone(string $phone): string {
    if (!preg_match('/^1\d{10}$/', $phone)) {
        return $phone; // 非手机号不脱敏
    }
    return substr($phone, 0, 3) . '****' . substr($phone, 7, 4);
}

/**
 * 身份证号脱敏（显示前6位+后4位，中间用*代替）
 */
function maskIdCard(string $idCard): string {
    if (strlen($idCard) !== 18) {
        return $idCard; // 非18位身份证不脱敏
    }
    return substr($idCard, 0, 6) . str_repeat('*', 8) . substr($idCard, 14, 4);
}

// 示例
echo maskPhone('13800138000'); // 输出：138****8000
echo maskIdCard('110101199001011234'); // 输出：110101********1234
```


### 关键说明
1. **加密扩展检查**：生产环境需确保 `openssl` 扩展启用，可通过 `phpinfo()` 或 `extension_loaded('openssl')` 验证；
2. **错误处理**：解密失败时（如密钥错误、数据篡改），需捕获异常，避免泄露敏感信息；
3. **数据备份**：加密密钥丢失即无法恢复数据，需定期备份密钥（离线存储，如加密 U 盘），同时备份加密后的数据库；
4. **合规性**：根据《个人信息保护法》，敏感数据存储需遵循“最小必要”原则（如仅存储手机号后6位可满足业务则不存全号），并明确数据留存期限。

</details>


## 9. 常用的 PHP 性能分析工具有哪些（如 Xdebug、XHProf）？它们的工作原理是什么？如何通过工具定位项目中的 CPU 或内存性能瓶颈？

<details>
<summary>答案与解析</summary>

- 思路要点：PHP 性能分析的核心是“找到耗时久、耗内存多的代码片段”，工具选择需结合“环境（开发/生产）”和“需求（CPU/内存/调用链）”。需先对比主流工具的差异，再拆解每种工具的工作原理，最后通过“实际操作流程”演示瓶颈定位，确保理论与实践结合。

### 一、主流 PHP 性能分析工具对比
不同工具的设计目标不同，需根据场景选择：

| 工具          | 核心特点                                  | 性能开销 | 适用环境       | 分析维度                | 优点                                  | 缺点                                  |
|---------------|-------------------------------------------|----------|----------------|-------------------------|---------------------------------------|---------------------------------------|
| **Xdebug**    | 全量追踪函数调用、断点调试、代码覆盖率     | 高（2-10倍性能损耗） | 开发/测试环境   | CPU 耗时、内存占用、调用链  | 功能全（调试+分析）、易用性高、文档完善 | 开销大，生产环境启用会导致服务卡顿      |
| **XHProf**    | 轻量级采样/全量追踪，专注性能分析         | 低（5-10%性能损耗） | 开发/生产环境   | CPU 耗时、内存占用、调用链  | 轻量，支持生产环境采样                | 官方停止维护（推荐用 Tideways 替代）   |
| **Tideways**  | XHProf 继任者，支持分布式追踪、SQL 耗时分析 | 低（3-8%性能损耗）  | 开发/生产环境   | CPU、内存、SQL 耗时、外部调用 | 兼容 XHProf 数据，支持微服务追踪      | 部分高级功能（如分布式追踪）需付费     |
| **Blackfire** | 商业化工具，可视化火焰图、自动优化建议    | 中（10-15%性能损耗） | 开发/生产环境   | CPU、内存、IO 分析、依赖分析 | 界面友好，自动生成优化建议            | 免费版有功能限制，付费成本高          |


### 二、核心工具的工作原理与使用
#### 1. Xdebug（开发环境首选，全量追踪）
##### （1）工作原理
Xdebug 是 PHP 扩展，通过“钩子机制”注入到 PHP 执行流程：
1. PHP 启动时加载 Xdebug 扩展，Xdebug 注册“函数调用钩子”“内存分配钩子”“代码执行钩子”；
2. 执行 PHP 脚本时，每次函数调用、内存分配/释放、代码行执行，Xdebug 都会记录“时间戳”“内存变化量”“调用栈”“代码行号”；
3. 脚本执行结束后，Xdebug 将采集到的原始数据（如 `cachegrind.out.1693456789.1234` 文件）写入磁盘；
4. 通过可视化工具（如 KCacheGrind、WebGrind）解析原始数据，生成“函数耗时排序”“调用链图”“内存变化曲线”。

##### （2）配置与定位 CPU 瓶颈（以 Web 项目为例）
**步骤1：安装与配置 Xdebug**（PHP 7.4 + Linux）：
```bash
# 1. 通过 pecl 安装 Xdebug（PHP 7.4 对应 Xdebug 2.9.x）
pecl install xdebug-2.9.8

# 2. 修改 php.ini（添加以下配置）
[xdebug]
zend_extension=xdebug.so
xdebug.profiler_enable=0 # 关闭自动启动，按需开启
xdebug.profiler_enable_trigger=1 # 允许通过 URL 参数触发分析
xdebug.profiler_output_dir=/tmp/xdebug # 性能分析文件输出目录
xdebug.profiler_output_name=cachegrind.out.%t.%p # 文件名格式（时间戳+进程ID）
xdebug.show_local_vars=1 # 调试时显示局部变量（可选）
```

**步骤2：触发性能分析**：
- Web 项目：访问目标接口时添加 `XDEBUG_PROFILE=1` 参数，如 `http://your-project.com/api/user?id=1&XDEBUG_PROFILE=1`，Xdebug 会在 `/tmp/xdebug` 目录生成 `cachegrind.out.1693456789.1234` 文件；
- CLI 脚本：执行脚本时通过 `-d` 参数启用分析，如 `php -d xdebug.profiler_enable=1 /var/www/script.php`。

**步骤3：分析报告（用 KCacheGrind）**：
1. 安装 KCacheGrind（Linux：`yum install kcachegrind`，Windows：下载 [QCacheGrind](https://sourceforge.net/projects/qcachegrindwin/)）；
2. 打开生成的 `cachegrind.out.*` 文件，关键视图：
   - **Flat Profile**：按“函数耗时占比”排序，直接定位耗时最久的函数（如 `UserService::getUserInfo` 耗时占比 60%，是主要瓶颈）；
   - **Call Graph**：可视化函数调用链，展示“谁调用了耗时函数”“耗时函数调用了哪些子函数”（如 `index.php` → `UserController::index` → `UserService::getUserInfo` → `DB::query`）；
   - **Caller/Callee**：查看函数的“调用者”（Caller）和“被调用者”（Callee），判断耗时是自身逻辑还是依赖调用（如 `UserService::getUserInfo` 耗时主要来自 `DB::query`，说明是数据库查询瓶颈）。

**示例瓶颈解决**：若 `Flat Profile` 中 `UserService::getUserInfo` 耗时最高，查看代码发现循环调用 `DB::query()` 获取用户订单（N+1 查询），优化为批量查询（`SELECT * FROM orders WHERE user_id IN (1,2,3)`），耗时从 500ms 降至 50ms。


#### 2. Tideways（生产环境首选，轻量采样）
##### （1）工作原理
Tideways 是 XHProf 的继任者，通过“采样追踪”降低性能开销，适合生产环境：
1. 支持两种追踪模式：
   - **采样模式**（默认）：按比例（如 1%）随机选择请求进行全量追踪，不影响整体服务性能；
   - **全量模式**：对所有请求追踪，仅用于开发/测试环境调试；
2. 采集数据包括：函数调用链、SQL 执行耗时（需集成 PDO/MySQLi 扩展）、Redis 调用耗时、内存分配/释放、外部 API 调用耗时；
3. 数据可写入本地文件（如 `xhprof.1693456789.1234.xhprof`）或发送到 Tideways 云端，通过 Web 界面查看“火焰图”“耗时统计”“内存变化”。

##### （2）配置与定位内存瓶颈（生产环境）
**步骤1：安装 Tideways**（PHP 8.0 + Linux）：
```bash
# 1. 下载并编译扩展
wget https://github.com/tideways/php-xhprof-extension/archive/refs/tags/v5.0.4.tar.gz
tar -zxvf v5.0.4.tar.gz
cd php-xhprof-extension-5.0.4
phpize
./configure --with-php-config=/usr/bin/php-config
make && make install

# 2. 修改 php.ini（添加以下配置）
[tideways]
extension=tideways_xhprof.so
tideways.profiler_enable=0 # 关闭自动启动
tideways.profiler_sampling_rate=1 # 采样率 1%（生产环境推荐 0.1-5%）
tideways.profiler_output_dir=/tmp/tideways # 输出目录
tideways.profiler_enable_trigger=1 # 允许通过 HTTP 头触发（可选）
```

**步骤2：追踪内存泄漏**：
内存泄漏的特征是“脚本执行中内存持续增长，不释放”，通过 Tideways 定位：
1. 生产环境触发采样：访问目标接口，Tideways 会随机采样 1% 的请求，生成 `xhprof.1693456789.1234.xhprof` 文件；
2. 分析内存变化：
   - 下载 `xhprof_html` 工具（Tideways 自带，或从 [XHProf 仓库](https://github.com/phacility/xhprof) 获取）；
   - 启动临时 Web 服务：`php -S 0.0.0.0:8080 -t xhprof_html/`；
   - 访问 `http://127.0.0.1:8080`，选择生成的 xhprof 文件，查看“Memory Usage”列（内存占用）和“Memory Peak”列（内存峰值）；
   - 定位“内存增量大且未释放”的函数，如 `FileService::readLargeFile` 每次调用内存增加 10MB，且未用 `unset()` 释放变量。

**示例瓶颈解决**：`FileService::readLargeFile` 用 `file_get_contents()` 一次性读取 100MB 日志文件，导致内存飙升，改为流式读取（`fopen()`+`fread()`）并及时释放变量：
```php
// 优化前（内存泄漏）
function readLargeFile($filePath) {
    $content = file_get_contents($filePath); // 一次性加载大文件到内存
    return $content;
}

// 优化后（流式读取，低内存占用）
function readLargeFile($filePath) {
    $handle = fopen($filePath, 'rb');
    $content = '';
    while (!feof($handle)) {
        $chunk = fread($handle, 4096); // 每次读取 4KB
        $content .= $chunk;
        // 处理 chunk（如解析日志）后可 unset，进一步降低内存
        // unset($chunk);
    }
    fclose($handle);
    return $content;
}
```


#### 3. Blackfire（可视化优化，适合团队协作）
##### （1）工作原理
Blackfire 是商业化工具，通过“探针注入”和“云端分析”提供一站式优化方案：
1. 在 PHP 项目中安装 Blackfire 探针（扩展），探针在 PHP 执行流程中采集“函数调用耗时”“内存变化”“IO 操作（数据库/Redis/文件）”“外部依赖调用”；
2. 采集到的数据通过 HTTPS 发送到 Blackfire 云端，云端对数据进行聚合分析，生成“火焰图”“调用树”“性能对比报告”；
3. 自动识别性能问题（如“重复数据库查询”“未缓存的热点数据”“大文件读取未释放内存”），并提供具体优化建议（如“添加 Redis 缓存”“使用批量查询”）。

##### （2）核心优势：火焰图分析
火焰图是定位 CPU 瓶颈的直观工具，**横轴代表“耗时占比”，纵轴代表“调用栈深度”，颜色越深代表耗时越久**：
- 例：分析电商商品详情页时，火焰图中 `ProductService::getComments` 占据横轴 40% 宽度（耗时占比最高），且调用栈显示其内部循环调用 `UserService::getUserName` 20次（N+1 查询）；
- Blackfire 自动标记该问题为“重复数据库查询”，并建议“改为批量查询用户名称（`SELECT name FROM users WHERE id IN (1,2,...,20)`）”，优化后耗时从 300ms 降至 50ms。


### 三、通用瓶颈定位流程（以 CPU 耗时为例）
无论使用哪种工具，定位性能瓶颈的核心流程一致：

1. **确定目标**：明确要分析的场景（如“商品详情页加载慢”“接口响应超 3 秒”“CLI 脚本执行超时”）；
2. **采集数据**：根据环境选择工具（开发用 Xdebug，生产用 Tideways），触发数据采集；
3. **分析报告**：
   - 第一步：找“总耗时占比 Top 5”的函数/操作，优先优化最大耗时项（如耗时占比 50% 的函数比 10% 的函数优化价值更高）；
   - 第二步：判断耗时类型（自身逻辑耗时 vs 依赖调用耗时）：
     - 自身逻辑耗时：如复杂循环、递归、字符串拼接，需优化算法（如用 `implode()` 替代 `.` 拼接长字符串）；
     - 依赖调用耗时：如数据库查询（无索引）、Redis 慢查询、外部 API 调用，需优化依赖（如添加数据库索引、优化 Redis 命令、缓存外部 API 结果）；
   - 第三步：验证优化效果（优化后重新采集数据，对比耗时变化，确保优化有效）；

4. **常见 CPU 瓶颈及解决方案**：
   | 瓶颈类型               | 表现特征                                  | 解决方案                                  |
   |------------------------|-------------------------------------------|-------------------------------------------|
   | N+1 数据库查询         | 循环中调用 DB 查询，函数调用次数多，SQL 耗时占比高 | 改为批量查询（`WHERE id IN (?)`）、添加关联查询（`JOIN`） |
   | 无索引 SQL 查询        | 单条 SQL 耗时超 100ms，数据库 CPU 高       | 为查询字段添加索引（如 `ALTER TABLE users ADD INDEX idx_phone (phone)`） |
   | 未缓存热点数据         | 同一数据多次查询数据库/Redis，无内存缓存   | 用 Redis 缓存热点数据（如 `product:detail:{id}`）、本地缓存（如 PHP `static` 变量） |
   | 复杂循环/递归          | 函数自身耗时高，无外部依赖调用            | 优化算法（如用迭代替代递归）、减少循环内计算（将重复计算的结果缓存到变量） |


### 关键说明
1. **生产环境注意事项**：
   - Xdebug 开销大，**绝对禁止在生产环境启用**（会导致服务卡顿甚至不可用）；
   - Tideways 采样率建议设为 0.1-5%，避免采样过多影响性能；
   - 分析完成后，建议关闭生产环境的性能分析工具，仅在需要时临时开启；

2. **内存瓶颈补充**：
   - 用 `memory_get_usage(true)` 手动打印内存变化（开发环境调试），定位内存增长节点；
   - 避免在循环中创建大对象、拼接长字符串、读取大文件未释放，及时用 `unset()` 释放不再使用的变量；

3. **工具互补**：开发环境用 Xdebug 调试细节（如代码行级耗时），生产环境用 Tideways 采样定位宏观瓶颈，最后用 Blackfire 生成优化报告，形成“调试-采样-优化”闭环。

</details>


## 10. 处理 10MB 以上的大文件上传时，PHP 需解决哪些核心问题（如内存溢出、上传超时）？如何通过 PHP 结合前端实现分片上传与断点续传功能？

<details>
<summary>答案与解析</summary>

- 思路要点：大文件上传的核心痛点是“内存占用超限”“上传超时”“网络中断后需重新上传”，解决方案围绕“分片拆分”（减小单次上传体积）和“状态记录”（断点续传）展开。需先拆解 PHP 原生上传的问题，再提供“前端分片+后端接收+断点续传”的完整方案，包含可落地的代码示例。

### 一、PHP 处理大文件上传的核心问题
原生 PHP 上传（依赖 `$_FILES`）处理 10MB+ 文件时，易触发以下问题，需针对性解决：

| 问题类型               | 原因分析                                  | 直接后果                                  |
|------------------------|-------------------------------------------|-------------------------------------------|
| **文件大小超限**       | PHP 配置 `upload_max_filesize`（默认 2MB）、`post_max_size`（默认 8MB）小于文件大小 | `$_FILES['file']['error']` 为 `UPLOAD_ERR_INI_SIZE` 或 `UPLOAD_ERR_FORM_SIZE`，上传失败 |
| **内存溢出**           | 用 `file_get_contents($_FILES['file']['tmp_name'])` 一次性加载大文件到内存，超过 `memory_limit`（默认 128MB） | 抛出“Allowed memory size of 134217728 bytes exhausted”错误，脚本终止 |
| **上传超时**           | 大文件上传耗时久，超过 `max_execution_time`（默认 30 秒）或 Nginx `client_body_timeout`（默认 60 秒） | 上传中断，临时文件被删除，需重新上传        |
| **网络中断重传**       | 无状态记录，网络中断后需从头上传            | 用户体验差，浪费带宽（如 100MB 文件传 90% 中断后需重新传） |
| **并发写入冲突**       | 多用户同时上传大文件，临时目录 IO 竞争      | 上传文件损坏，合并失败                    |


### 二、核心解决方案：分片上传 + 断点续传
#### 1. 整体流程
将大文件（如 100MB）切割为多个小分片（如 5MB/片），分多次上传，后端记录上传状态，支持中断后继续上传，流程如下：

1. **前端预处理**：
   - 计算文件 MD5（唯一标识，避免重复上传）；
   - 将文件按固定大小（如 5MB）切割为分片，编号从 0 开始（如 100MB 文件分 20 片，编号 0-19）；
2. **断点检查**：
   - 前端向后端请求“已上传的分片列表”；
   - 前端仅上传未在列表中的分片（避免重复上传）；
3. **分片上传**：
   - 前端用 FormData 逐片上传（携带参数：文件 MD5、分片编号、总分片数、分片数据）；
   - 后端接收分片，暂存到临时目录（如 `./temp/{fileMD5}/`），并记录分片上传状态；
4. **合并分片**：
   - 前端确认所有分片上传完成，请求后端合并；
   - 后端按分片编号排序，将所有分片合并为完整文件，删除临时分片；
5. **清理冗余**：
   - 后端定期删除过期未合并的临时分片（如超过 24 小时），避免磁盘占用过大。


### 三、完整实现代码（前端 + 后端）
#### 1. 前端实现（HTML + JavaScript，基于 File API）
核心功能：文件选择、MD5 计算、断点检查、分片上传、合并请求，需引入 `spark-md5` 库（计算文件 MD5，避免重复上传）。

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>大文件分片上传</title>
    <style>
        #progress { margin: 10px 0; color: #333; }
    </style>
</head>
<body>
    <input type="file" id="fileInput" accept="*" />
    <button onclick="startUpload()">开始上传</button>
    <div id="progress">进度：0%</div>

    <!-- 引入 spark-md5 库（计算文件 MD5） -->
    <script src="https://cdn.jsdelivr.net/npm/spark-md5@3.0.2/spark-md5.min.js"></script>
    <script>
        // 配置
        const FILE_CHUNK_SIZE = 5 * 1024 * 1024; // 分片大小：5MB
        const UPLOAD_URL = 'upload.php'; // 后端接口地址
        let file = null; // 选中的文件
        let fileMD5 = ''; // 文件唯一标识（MD5）
        let totalChunks = 0; // 总分片数
        let uploadedChunks = []; // 已上传分片编号

        // 1. 选择文件
        document.getElementById('fileInput').addEventListener('change', (e) => {
            file = e.target.files[0];
            if (!file) return;
            // 计算总分片数
            totalChunks = Math.ceil(file.size / FILE_CHUNK_SIZE);
            console.log(`文件信息：名称=${file.name}，大小=${(file.size/1024/1024).toFixed(2)}MB，总分片数=${totalChunks}`);
        });

        // 2. 计算文件 MD5（避免重复上传，分片读取文件，避免内存溢出）
        function calculateFileMD5() {
            return new Promise((resolve, reject) => {
                const spark = new SparkMD5.ArrayBuffer(); // SparkMD5 实例
                const fileReader = new FileReader(); // 文件读取器
                let offset = 0; // 读取偏移量

                // 分片读取文件
                function loadNextChunk() {
                    const blob = file.slice(offset, offset + FILE_CHUNK_SIZE); // 切割当前分片
                    fileReader.readAsArrayBuffer(blob); // 读取为 ArrayBuffer

                    // 读取完成回调
                    fileReader.onload = (e) => {
                        spark.append(e.target.result); // 追加到 MD5 计算
                        offset += FILE_CHUNK_SIZE; // 更新偏移量

                        if (offset < file.size) {
                            loadNextChunk(); // 继续读取下一分片
                        } else {
                            fileMD5 = spark.end(); // 计算完成，获取 MD5
                            resolve(fileMD5);
                        }
                    };

                    // 读取错误回调
                    fileReader.onerror = (e) => {
                        reject(new Error(`计算 MD5 失败：${e.message}`));
                    };
                }

                loadNextChunk(); // 开始读取
            });
        }

        // 3. 断点检查：查询后端已上传的分片列表
        async function checkUploadedChunks() {
            try {
                const response = await fetch(`${UPLOAD_URL}?action=check&fileMD5=${fileMD5}&fileName=${encodeURIComponent(file.name)}`);
                const data = await response.json();
                if (data.success) {
                    uploadedChunks = data.uploadedChunks || []; // 后端返回已上传分片编号
                    updateProgress(); // 更新进度
                    // 若文件已完整上传，直接提示
                    if (data.isComplete) {
                        alert(`文件 ${file.name} 已存在，无需重复上传！`);
                        return false;
                    }
                } else {
                    throw new Error(`断点检查失败：${data.msg}`);
                }
                return true;
            } catch (e) {
                alert(e.message);
                return false;
            }
        }

        // 4. 上传单个分片
        async function uploadChunk(chunkIndex) {
            try {
                // 切割当前分片的 blob 数据
                const start = chunkIndex * FILE_CHUNK_SIZE;
                const end = Math.min(start + FILE_CHUNK_SIZE, file.size);
                const chunkBlob = file.slice(start, end);

                // 构建 FormData（携带分片数据与元信息）
                const formData = new FormData();
                formData.append('fileMD5', fileMD5);
                formData.append('chunkIndex', chunkIndex);
                formData.append('totalChunks', totalChunks);
                formData.append('chunk', chunkBlob); // 分片数据（二进制）

                // 发送上传请求
                const response = await fetch(UPLOAD_URL, {
                    method: 'POST',
                    body: formData // FormData 自动处理 Content-Type
                });

                const data = await response.json();
                if (data.success) {
                    uploadedChunks.push(chunkIndex); // 记录已上传分片
                    updateProgress(); // 更新进度
                    return true;
                } else {
                    throw new Error(`分片 ${chunkIndex} 上传失败：${data.msg}`);
                }
            } catch (e) {
                console.error(e);
                return false;
            }
        }

        // 5. 批量上传未完成分片（控制并发，避免浏览器同时发送过多请求）
        async function uploadAllChunks() {
            // 获取未上传的分片编号
            const unUploadedChunks = [];
            for (let i = 0; i < totalChunks; i++) {
                if (!uploadedChunks.includes(i)) {
                    unUploadedChunks.push(i);
                }
            }

            // 所有分片已上传，请求合并
            if (unUploadedChunks.length === 0) {
                await mergeChunks();
                return;
            }

            // 控制并发数为 3（避免浏览器请求队列阻塞）
            const concurrency = 3;
            let currentIndex = 0; // 当前上传的分片索引

            // 递归上传分片
            async function run() {
                if (currentIndex >= unUploadedChunks.length) {
                    await mergeChunks(); // 所有分片上传完成，请求合并
                    return;
                }

                const chunkIndex = unUploadedChunks[currentIndex];
                currentIndex++;

                // 分片上传失败重试（最多 3 次）
                let retryCount = 3;
                while (retryCount > 0) {
                    if (await uploadChunk(chunkIndex)) {
                        break; // 上传成功，继续下一分片
                    }
                    retryCount--;
                    if (retryCount === 0) {
                        throw new Error(`分片 ${chunkIndex} 多次上传失败，请刷新页面重试`);
                    }
                    await new Promise(resolve => setTimeout(resolve, 1000)); // 重试间隔 1 秒
                }

                await run(); // 上传下一分片
            }

            // 启动并发上传（创建 concurrency 个任务）
            const tasks = Array(concurrency).fill().map(run);
            await Promise.all(tasks);
        }

        // 6. 请求后端合并分片
        async function mergeChunks() {
            try {
                const response = await fetch(`${UPLOAD_URL}?action=merge&fileMD5=${fileMD5}&fileName=${encodeURIComponent(file.name)}`);
                const data = await response.json();
                if (data.success) {
                    alert(`文件 ${file.name} 上传完成！保存路径：${data.savePath}`);
                } else {
                    throw new Error(`文件合并失败：${data.msg}`);
                }
            } catch (e) {
                alert(e.message);
            }
        }

        // 7. 更新上传进度
        function updateProgress() {
            const progressRate = Math.round((uploadedChunks.length / totalChunks) * 100);
            document.getElementById('progress').innerText = `进度：${progressRate}%`;
        }

        // 8. 入口函数：开始上传
        async function startUpload() {
            if (!file) {
                alert('请先选择文件');
                return;
            }

            try {
                // 步骤1：计算文件 MD5
                document.getElementById('progress').innerText = '正在计算文件 MD5...';
                await calculateFileMD5();

                // 步骤2：断点检查
                document.getElementById('progress').innerText = '正在检查已上传分片...';
                const canUpload = await checkUploadedChunks();
                if (!canUpload) return;

                // 步骤3：上传所有未完成分片
                await uploadAllChunks();
            } catch (e) {
                alert(`上传异常：${e.message}`);
                console.error(e);
            }
        }
    </script>
</body>
</html>
```

#### 2. 后端实现（PHP，处理分片接收、暂存、合并）
核心功能：断点检查、分片接收、分片合并、过期临时分片清理，需调整 PHP 配置避免上传超限。

```php
<?php
/**
 * 大文件分片上传后端接口（upload.php）
 */
// 调整 PHP 配置（处理大文件上传）
ini_set('memory_limit', '256M'); // 临时提高内存限制（避免分片处理内存溢出）
ini_set('max_execution_time', 300); // 延长执行时间（5分钟，适应大文件合并）
ini_set('upload_max_filesize', '10M'); // 单个分片最大大小（需大于分片大小，如 10M > 5M）
ini_set('post_max_size', '10M'); // POST 数据最大大小（与 upload_max_filesize 一致）

// 配置
$TEMP_DIR = './temp/'; // 分片临时存储目录（需在 Web 根目录外，避免直接访问）
$SAVE_DIR = './uploads/'; // 最终文件保存目录
$EXPIRE_TIME = 24 * 3600; // 临时分片过期时间（24小时，过期自动清理）

// 初始化目录（确保目录存在且可写）
initDirs();

// 处理不同动作（check：断点检查，merge：合并分片，默认：接收分片）
$action = $_GET['action'] ?? '';
switch ($action) {
    case 'check':
        handleCheck();
        break;
    case 'merge':
        handleMerge();
        break;
    default:
        handleUpload();
        break;
}

/**
 * 1. 初始化目录 + 清理过期临时分片
 */
function initDirs() {
    global $TEMP_DIR, $SAVE_DIR, $EXPIRE_TIME;

    // 临时目录：按文件 MD5 分目录存储分片（避免冲突）
    if (!is_dir($TEMP_DIR)) {
        mkdir($TEMP_DIR, 0755, true); // 递归创建目录，权限 0755（所有者可读写执行，其他只读）
    }

    // 最终保存目录
    if (!is_dir($SAVE_DIR)) {
        mkdir($SAVE_DIR, 0755, true);
    }

    // 清理过期临时分片（每天执行一次，可改为 Linux 定时任务）
    cleanExpiredTemp();
}

/**
 * 2. 清理过期的临时分片（超过 EXPIRE_TIME 未合并）
 */
function cleanExpiredTemp() {
    global $TEMP_DIR, $EXPIRE_TIME;
    $now = time();

    // 遍历临时目录下的所有子目录（每个子目录对应一个文件的分片）
    $dirIterator = new DirectoryIterator($TEMP_DIR);
    foreach ($dirIterator as $fileInfo) {
        if ($fileInfo->isDir() && !$fileInfo->isDot()) {
            $chunkDir = $fileInfo->getPathname(); // 分片目录路径
            $mtime = $fileInfo->getMTime(); // 目录最后修改时间

            // 目录超过过期时间，删除该目录及所有分片
            if ($now - $mtime > $EXPIRE_TIME) {
                deleteDir($chunkDir);
                echo "清理过期分片目录：{$chunkDir}\n";
            }
        }
    }
}

/**
 * 3. 递归删除目录（用于清理临时分片）
 */
function deleteDir($dirPath) {
    if (!is_dir($dirPath)) {
        return;
    }

    // 遍历目录下的所有文件/子目录，先删文件，再删目录
    $fileIterator = new RecursiveIteratorIterator(
        new RecursiveDirectoryIterator($dirPath, RecursiveDirectoryIterator::SKIP_DOTS),
        RecursiveIteratorIterator::CHILD_FIRST // 先处理子节点（文件），再处理父节点（目录）
    );

    foreach ($fileIterator as $file) {
        if ($file->isDir()) {
            rmdir($file->getPathname());
        } else {
            unlink($file->getPathname());
        }
    }

    rmdir($dirPath); // 删除空目录
}

/**
 * 4. 处理断点检查：返回已上传的分片列表
 */
function handleCheck() {
    global $TEMP_DIR, $SAVE_DIR;
    $fileMD5 = $_GET['fileMD5'] ?? '';
    $fileName = urldecode($_GET['fileName'] ?? '');

    // 校验参数
    if (empty($fileMD5) || empty($fileName)) {
        jsonResponse(false, '参数缺失（fileMD5 或 fileName）');
            // 检查文件是否已完整上传（避免重复上传）
    $savePath = $SAVE_DIR . $fileName;
    if (file_exists($savePath)) {
        jsonResponse(true, '文件已存在', [
            'uploadedChunks' => [],
            'isComplete' => true
        ]);
    }

    // 检查分片目录（存储当前文件的所有分片）
    $chunkDir = $TEMP_DIR . $fileMD5 . '/';
    $uploadedChunks = [];
    if (is_dir($chunkDir)) {
        // 读取所有分片文件（文件名格式：chunk_0, chunk_1, ..., chunk_19）
        $chunkFiles = glob($chunkDir . 'chunk_*');
        foreach ($chunkFiles as $file) {
            $basename = basename($file);
            // 提取分片编号（匹配 chunk_ 后的数字）
            if (preg_match('/^chunk_(\d+)$/', $basename, $matches)) {
                $uploadedChunks[] = (int)$matches[1];
            }
        }
        sort($uploadedChunks); // 按编号排序，便于前端判断
    }

    jsonResponse(true, '断点检查完成', [
        'uploadedChunks' => $uploadedChunks,
        'isComplete' => false
    ]);
}

/**
 * 5. 处理分片上传：接收分片并暂存到临时目录
 */
function handleUpload() {
    global $TEMP_DIR;
    // 获取 POST 参数
    $fileMD5 = $_POST['fileMD5'] ?? '';
    $chunkIndex = $_POST['chunkIndex'] ?? '';
    $totalChunks = $_POST['totalChunks'] ?? '';
    $chunk = $_FILES['chunk'] ?? [];

    // 1. 校验参数
    if (empty($fileMD5) || !is_numeric($chunkIndex) || !is_numeric($totalChunks) || empty($chunk)) {
        jsonResponse(false, '参数缺失或无效（fileMD5/chunkIndex/chunk）');
    }
    $chunkIndex = (int)$chunkIndex;
    $totalChunks = (int)$totalChunks;

    // 2. 检查上传错误（如文件过大、临时文件丢失等）
    if ($chunk['error'] !== UPLOAD_ERR_OK) {
        $errorMsg = match ($chunk['error']) {
            UPLOAD_ERR_INI_SIZE => '分片超过 PHP 配置的 upload_max_filesize',
            UPLOAD_ERR_FORM_SIZE => '分片超过表单指定的 MAX_FILE_SIZE',
            UPLOAD_ERR_PARTIAL => '分片仅部分上传',
            UPLOAD_ERR_NO_FILE => '未接收到分片数据',
            default => "分片上传错误（代码：{$chunk['error']}）"
        };
        jsonResponse(false, $errorMsg);
    }

    // 3. 创建分片目录（按文件 MD5 分目录，避免不同文件分片冲突）
    $chunkDir = $TEMP_DIR . $fileMD5 . '/';
    if (!is_dir($chunkDir)) {
        if (!mkdir($chunkDir, 0755, true)) {
            jsonResponse(false, '分片目录创建失败（权限不足或路径无效）');
        }
    }

    // 4. 暂存分片文件（文件名格式：chunk_0，便于后续按编号排序）
    $chunkPath = $chunkDir . "chunk_{$chunkIndex}";
    // 移动临时文件到分片目录（overwrite 为 true，覆盖已存在的分片，支持重传）
    if (!move_uploaded_file($chunk['tmp_name'], $chunkPath)) {
        jsonResponse(false, '分片保存失败（临时文件移动错误）');
    }

    // 5. 限制分片文件权限（仅允许读写，禁止执行）
    chmod($chunkPath, 0644);

    jsonResponse(true, "分片 {$chunkIndex} 上传成功");
}

/**
 * 6. 处理分片合并：将所有分片按编号合并为完整文件
 */
function handleMerge() {
    global $TEMP_DIR, $SAVE_DIR;
    $fileMD5 = $_GET['fileMD5'] ?? '';
    $fileName = urldecode($_GET['fileName'] ?? '');

    // 1. 校验参数
    if (empty($fileMD5) || empty($fileName)) {
        jsonResponse(false, '参数缺失（fileMD5 或 fileName）');
    }

    // 2. 检查分片目录是否存在
    $chunkDir = $TEMP_DIR . $fileMD5 . '/';
    if (!is_dir($chunkDir)) {
        jsonResponse(false, '分片目录不存在（可能已被清理或未上传任何分片）');
    }

    // 3. 读取所有分片文件并按编号排序
    $chunkFiles = glob($chunkDir . 'chunk_*');
    if (empty($chunkFiles)) {
        jsonResponse(false, '未找到任何分片文件');
    }

    // 按分片编号排序（确保合并顺序正确）
    usort($chunkFiles, function ($a, $b) {
        // 提取分片编号
        preg_match('/^chunk_(\d+)$/', basename($a), $matchA);
        preg_match('/^chunk_(\d+)$/', basename($b), $matchB);
        return (int)$matchA[1] - (int)$matchB[1];
    });

    // 4. 合并分片到完整文件（二进制写入，避免文本模式换行符问题）
    $savePath = $SAVE_DIR . $fileName;
    $destFp = fopen($savePath, 'wb'); // 'wb' 模式：二进制写入，覆盖已有文件
    if (!$destFp) {
        jsonResponse(false, '完整文件创建失败（权限不足或磁盘空间不足）');
    }

    // 逐个读取分片并写入完整文件
    foreach ($chunkFiles as $chunkPath) {
        $srcFp = fopen($chunkPath, 'rb'); // 'rb' 模式：二进制读取
        if (!$srcFp) {
            fclose($destFp);
            jsonResponse(false, "分片 {$chunkPath} 读取失败");
        }

        // 分片内容写入完整文件（每次读取 4KB，避免内存溢出）
        while (!feof($srcFp)) {
            fwrite($destFp, fread($srcFp, 4096));
        }

        fclose($srcFp); // 关闭分片文件
    }

    fclose($destFp); // 关闭完整文件
    chmod($savePath, 0644); // 限制文件权限（仅读写）

    // 5. 合并成功后，删除临时分片目录（释放磁盘空间）
    deleteDir($chunkDir);

    jsonResponse(true, '文件合并成功', [
        'savePath' => $savePath,
        'fileSize' => filesize($savePath) // 返回文件大小，供前端校验
    ]);
}

/**
 * 7. 统一 JSON 响应函数
 */
function jsonResponse(bool $success, string $msg, array $data = []): void {
    // 设置响应头（确保前端正确解析 JSON）
    header('Content-Type: application/json; charset=utf-8');
    // 组合响应数据
    $response = array_merge([
        'success' => $success,
        'msg' => $msg
    ], $data);
    // 输出 JSON 并终止脚本
    echo json_encode($response, JSON_UNESCAPED_UNICODE); // 保留中文不转义
    exit();
}
?>
</details>


## 11. Redis 的发布订阅（Pub/Sub）机制是什么？它的工作流程如何？与专业消息队列（如 RabbitMQ）相比，Redis Pub/Sub 有哪些优缺点？

<details>
<summary>答案与解析</summary>

- 思路要点：Redis Pub/Sub 是基于“频道（Channel）”的消息传递机制，核心是“广播式通信”，需先明确其定义与工作流程，再从“消息可靠性”“功能完整性”“性能”三个维度与 RabbitMQ 对比，结合业务场景选择工具。

### 一、Redis Pub/Sub 机制定义
Redis 发布订阅（Publish/Subscribe）是一种**消息通信模式**，允许“发布者（Publisher）”向指定“频道（Channel）”发送消息，所有“订阅者（Subscriber）”订阅该频道后，可实时接收消息，实现“一对多”的广播通信（如实时通知、弹幕、日志同步）。

- 核心概念：
  - **频道（Channel）**：消息的分类标识（如 `chat:room1`、`log:error`），发布者按频道发消息，订阅者按频道收消息；
  - **发布者（Publisher）**：发送消息的客户端，无需关注订阅者，仅需指定频道发送；
  - **订阅者（Subscriber）**：接收消息的客户端，需先订阅频道，才能接收该频道的后续消息；
  - **模式订阅（Pattern Subscribe）**：支持通配符订阅（如 `chat:*` 订阅所有以 `chat:` 开头的频道）。


### 二、Redis Pub/Sub 工作流程
以“用户发送弹幕到直播间”为例，流程如下：

1. **订阅频道**：
   - 多个客户端（观众）执行 `SUBSCRIBE chat:room1`，订阅“直播间1”的弹幕频道；
   - Redis 为每个订阅者维护“频道-订阅者”映射表（如 `chat:room1` → [客户端A, 客户端B, 客户端C]）。

2. **发布消息**：
   - 客户端（主播/用户）执行 `PUBLISH chat:room1 "大家好！"`，向 `chat:room1` 频道发布消息；
   - Redis 收到发布请求后，查询 `chat:room1` 对应的所有订阅者。

3. **消息推送**：
   - Redis 将消息“大家好！”实时推送给所有订阅 `chat:room1` 的客户端（A、B、C）；
   - 订阅者客户端收到消息后，输出消息内容（如弹幕展示在页面上）。

4. **模式订阅示例**：
   - 客户端执行 `PSUBSCRIBE chat:*`，订阅所有直播间的弹幕频道；
   - 当 `chat:room1`、`chat:room2` 频道有消息时，该客户端会收到所有消息。


### 三、Redis Pub/Sub 与 RabbitMQ 的对比
| 对比维度                | Redis Pub/Sub                          | RabbitMQ（专业消息队列）              |
|-------------------------|---------------------------------------|---------------------------------------|
| **消息可靠性**          | 低（无持久化，订阅者离线则消息丢失；Redis 宕机后未消费消息丢失） | 高（支持消息持久化、确认机制（ACK）、死信队列，确保消息不丢失） |
| **消息堆积处理**        | 不支持（发布者发消息时，若订阅者未在线，消息直接丢弃；无队列存储消息） | 支持（消息先存入队列，订阅者离线后上线可消费堆积消息） |
| **功能完整性**          | 简单（仅广播、模式订阅，无路由、过滤、重试机制） | 丰富（支持交换机路由、消息过滤、延迟队列、重试机制、死信队列） |
| **性能**                | 高（基于内存操作，无复杂处理，适合高并发广播） | 中（需处理路由、持久化等，性能略低于 Redis，但支持更高吞吐量） |
| **适用场景**            | 实时性要求高、消息可丢失的场景（弹幕、实时通知、日志广播） | 消息不可丢失、需复杂路由的场景（订单处理、支付回调、异步任务） |
| **部署与维护**          | 简单（与 Redis 服务一体，无需额外部署） | 复杂（需独立部署，配置交换机、队列、绑定关系） |


### 四、Redis Pub/Sub 的优缺点与适用场景
#### 1. 优点
- **轻量高效**：基于 Redis 原生功能，无需额外组件，部署维护成本低；
- **实时性强**：消息实时广播，无中间存储环节，延迟低（毫秒级）；
- **支持模式订阅**：通配符订阅简化多频道管理（如 `log:*` 订阅所有日志频道）；
- **高并发支持**：基于 Redis 内存操作，能承受高并发发布/订阅（万级 QPS 无压力）。

#### 2. 缺点
- **消息不可靠**：无持久化机制，Redis 宕机或订阅者离线，消息直接丢失；
- **无消息堆积**：消息仅实时推送，不存储，无法处理“订阅者晚于发布者启动”的场景；
- **功能简单**：缺乏路由、重试、死信等企业级功能，无法满足复杂业务需求。

#### 3. 适用场景
- **实时通知**：如用户登录提醒、订单状态实时更新（允许偶尔丢失）；
- **弹幕系统**：直播间弹幕无需持久化，实时性优先；
- **日志广播**：多台服务器同步接收日志（如错误日志实时推送至监控系统）；
- **简单广播场景**：无需复杂处理，仅需“一对多”实时通信。


### 五、Redis Pub/Sub 代码示例（PHP + Redis）
#### 1. 订阅者代码（接收消息）
```php
<?php
// 订阅者：订阅 chat:room1 频道，接收弹幕消息
$redis = new Redis();
$redis->connect('127.0.0.1', 6379);

// 回调函数：接收消息时触发
function messageHandler($redis, $channel, $message) {
    echo "收到频道 [{$channel}] 的消息：{$message}\n";
}

// 订阅频道（支持多个频道，如 ['chat:room1', 'chat:room2']）
$redis->subscribe(['chat:room1'], 'messageHandler');

// 阻塞等待消息（持续运行，直到手动终止）
while (true) {
    $redis->psubscribe([], 'messageHandler'); // 保持连接，接收后续消息
}
```

#### 2. 发布者代码（发送消息）
```php
<?php
// 发布者：向 chat:room1 频道发送弹幕消息
$redis = new Redis();
$redis->connect('127.0.0.1', 6379);

// 模拟用户发送弹幕
$danmu = "用户" . rand(1000, 9999) . "：这波操作太秀了！";
// 发布消息（返回值为接收消息的订阅者数量）
$subscriberCount = $redis->publish('chat:room1', $danmu);

echo "向频道 chat:room1 发布消息：{$danmu}\n";
echo "当前在线订阅者数量：{$subscriberCount}\n";
```

#### 3. 模式订阅示例（订阅所有 chat:* 频道）
```php
<?php
// 模式订阅者：订阅所有以 chat: 开头的频道
$redis = new Redis();
$redis->connect('127.0.0.1', 6379);

// 模式订阅回调函数
function patternMessageHandler($redis, $pattern, $channel, $message) {
    echo "模式 [{$pattern}] 匹配频道 [{$channel}]，收到消息：{$message}\n";
}

// 模式订阅（chat:* 匹配所有 chat: 开头的频道）
$redis->psubscribe(['chat:*'], 'patternMessageHandler');

// 阻塞等待消息
while (true) {
    $redis->psubscribe([], 'patternMessageHandler');
}
```

</details>


## 12. 如何通过 PHP 实现接口请求频率限制（防止恶意刷接口）？常见的限流算法（令牌桶、漏桶）在 PHP 中如何落地？

<details>
<summary>答案与解析</summary>

- 思路要点：接口限流的核心是“控制单位时间内的请求次数”，防止恶意刷接口导致服务过载。需先明确限流的核心指标（如“1分钟内单个IP最多100次请求”），再对比令牌桶、漏桶算法的差异，结合 Redis 实现分布式限流（支持多服务器共享限流状态），避免单机限流的局限性。

### 一、接口限流的核心概念与应用场景
#### 1. 核心概念
- **限流粒度**：按 IP（最常用）、用户 ID、接口路径划分（如“IP=192.168.1.1 访问 /api/login 每分钟最多10次”）；
- **限流指标**：单位时间内的最大请求次数（如 100 次/分钟、10 次/秒）；
- **限流效果**：超出限制时返回 429 Too Many Requests，或提示“请求过于频繁，请稍后再试”。

#### 2. 应用场景
- 登录接口：防止暴力破解密码（如 1分钟内同一IP最多5次登录请求）；
- 短信接口：防止恶意刷短信（如 1小时内同一手机号最多3次发送）；
- 高频查询接口：如商品列表、用户信息，防止爬虫或恶意请求压垮服务。


### 二、常见限流算法对比
| 算法类型                | 核心原理                                  | 优点                                  | 缺点                                  | 适用场景                          |
|-------------------------|-------------------------------------------|---------------------------------------|---------------------------------------|-----------------------------------|
| **固定窗口计数**        | 按固定时间窗口（如1分钟）统计请求次数，超限额则限流 | 实现简单，易理解                      | 窗口切换时可能出现“流量突刺”（如窗口末尾和开头各100次，实际200次/分钟） | 对精度要求不高的场景              |
| **滑动窗口计数**        | 将固定窗口拆分为多个小窗口（如1分钟拆为6个10秒窗口），滑动统计请求次数 | 避免流量突刺，精度高于固定窗口        | 实现略复杂，需存储多个小窗口的计数    | 对精度有一定要求的场景            |
| **令牌桶算法**          | 固定速率向桶中放令牌，请求需获取令牌才能通过，令牌空则限流 | 支持突发流量（桶内有令牌时可快速处理） | 需维护令牌生成速率和桶容量，实现稍复杂 | 允许突发请求的场景（如接口峰值流量） |
| **漏桶算法**            | 请求先进入桶中，桶按固定速率处理请求，桶满则丢弃请求 | 平滑流量输出，避免服务过载            | 不支持突发流量（即使桶空，也需按速率处理） | 要求流量平稳的场景（如数据库访问） |


### 三、PHP 实现限流的关键依赖：Redis
单机限流（如用 PHP 静态变量）无法支持多服务器集群（负载均衡下不同服务器无法共享计数），**推荐用 Redis 存储限流状态**，理由：
- 分布式共享：多服务器可通过 Redis 读写同一限流键，确保计数一致；
- 原子操作：Redis 的 `INCR`、`EXPIRE`、`PIPELINE` 等命令支持原子操作，避免并发计数误差；
- 过期自动清理：Redis 键可设置过期时间，无需手动维护窗口生命周期。


### 四、核心限流算法的 PHP 实现（基于 Redis）
#### 1. 固定窗口计数（最简单，推荐入门）
**原理**：以“IP+接口路径+时间窗口”为 Redis 键（如 `limit:192.168.1.1:/api/login:202405201005`，代表 2024-05-20 10:05 分的窗口），统计该窗口内的请求次数，超限额则限流。

```php
<?php
/**
 * 固定窗口计数限流
 * @param string $key 限流键（如 IP+接口路径）
 * @param int $limit 单位时间内最大请求次数
 * @param int $expire 时间窗口（秒）
 * @return bool true：允许请求，false：限流
 */
function fixedWindowLimit(string $key, int $limit, int $expire): bool {
    $redis = new Redis();
    $redis->connect('127.0.0.1', 6379);

    // 1. 生成限流键（加入时间窗口，如每60秒一个窗口）
    $window = floor(time() / $expire); // 时间窗口标识（如 1693456789 / 60 = 28224279，代表第28224279个60秒窗口）
    $limitKey = "limit:{$key}:{$window}";

    // 2. 原子递增计数（避免并发问题）
    $count = $redis->incr($limitKey);

    // 3. 首次请求时设置过期时间（仅执行一次，避免重复设置）
    if ($count === 1) {
        $redis->expire($limitKey, $expire);
    }

    // 4. 判断是否超出限制
    return $count <= $limit;
}

// 调用示例：限制 192.168.1.1 访问 /api/login 接口，60秒内最多10次请求
$clientIp = $_SERVER['REMOTE_ADDR']; // 获取客户端IP
$apiPath = '/api/login';
$limitKey = "{$clientIp}:{$apiPath}";

if (fixedWindowLimit($limitKey, 10, 60)) {
    echo "请求允许，执行接口逻辑...";
} else {
    http_response_code(429); // 返回 429 状态码
    echo "请求过于频繁，请60秒后再试！";
}
?>
```


#### 2. 令牌桶算法（支持突发流量，推荐生产）
**原理**：
- 桶容量：最多存储 N 个令牌（支持的最大突发流量）；
- 令牌生成速率：每 M 毫秒生成 1 个令牌（如 100ms 生成1个，即每秒10个）；
- 请求处理：每次请求需获取 1 个令牌，令牌不足则限流。

需在 Redis 中存储两个键：`token:bucket:{$key}:tokens`（当前桶内令牌数）、`token:bucket:{$key}:last_refill`（上次令牌填充时间）。

```php
<?php
/**
 * 令牌桶算法限流
 * @param string $key 限流键（如 IP+接口路径）
 * @param int $capacity 桶容量（最大令牌数，支持的突发流量）
 * @param int $rate 令牌生成速率（令牌/秒，如 10 代表每秒生成10个）
 * @return bool true：允许请求，false：限流
 */
function tokenBucketLimit(string $key, int $capacity, int $rate): bool {
    $redis = new Redis();
    $redis->connect('127.0.0.1', 6379);

    // 限流键前缀
    $tokenKey = "token:bucket:{$key}:tokens";
    $lastRefillKey = "token:bucket:{$key}:last_refill";

    // 1. 获取当前令牌数和上次填充时间（Redis 事务确保原子性）
    $redis->multi();
    $redis->get($tokenKey);
    $redis->get($lastRefillKey);
    $result = $redis->exec();

    $currentTokens = (int)$result[0] ?: $capacity; // 初始令牌数=桶容量
    $lastRefillTime = (int)$result[1] ?: time();   // 初始时间=当前时间

    // 2. 计算从上次填充到现在应生成的令牌数
    $now = time();
    $elapsedTime = $now - $lastRefillTime; // 间隔时间（秒）
    $newTokens = (int)($elapsedTime * $rate); // 应生成的令牌数

    // 3. 更新当前令牌数（不超过桶容量）
    $currentTokens = min($currentTokens + $newTokens, $capacity);

    // 4. 判断是否有令牌可用（有则消耗1个，无则限流）
    $allow = false;
    if ($currentTokens > 0) {
        $currentTokens--;
        $allow = true;
    }

    // 5. 保存更新后的令牌数和当前时间（Redis 事务确保原子性）
    $redis->multi();
    $redis->set($tokenKey, $currentTokens);
    $redis->set($lastRefillKey, $now);
    $redis->exec();

    return $allow;
}

// 调用示例：限制 192.168.1.1 访问 /api/product 接口，桶容量20（支持20次突发请求），每秒生成5个令牌
$clientIp = $_SERVER['REMOTE_ADDR'];
$apiPath = '/api/product';
$limitKey = "{$clientIp}:{$apiPath}";

if (tokenBucketLimit($limitKey, 20, 5)) {
    echo "请求允许，执行接口逻辑...";
} else {
    http_response_code(429);
    echo "请求过于频繁，请稍后再试！";
}
?>
```


#### 3. 漏桶算法（平滑流量，适合数据库访问）
**原理**：
- 桶容量：最多存储 N 个请求，超出则丢弃；
- 漏出速率：每 M 毫秒处理 1 个请求（如 200ms 处理1个，即每秒5个）；
- 请求处理：请求先进入桶，桶按固定速率处理，桶满则限流。

```php
<?php
/**
 * 漏桶算法限流
 * @param string $key 限流键（如 IP+接口路径）
 * @param int $capacity 桶容量（最大缓存请求数）
 * @param int $rate 漏出速率（请求/秒，如5代表每秒处理5个）
 * @return bool true：允许请求，false：限流
 */
function leakyBucketLimit(string $key, int $capacity, int $rate): bool {
    $redis = new Redis();
    $redis->connect('127.0.0.1', 6379);

    // 限流键前缀
    $requestCountKey = "leaky:bucket:{$key}:count"; // 当前桶内请求数
    $lastProcessKey = "leaky:bucket:{$key}:last_process"; // 上次处理请求时间

    // 1. 获取当前请求数和上次处理时间
    $redis->multi();
    $redis->get($requestCountKey);
    $redis->get($lastProcessKey);
    $result = $redis->exec();

    $currentCount = (int)$result[0] ?: 0;
    $lastProcessTime = (int)$result[1] ?: time();

    $now = time();
    $leakInterval = 1 / $rate; // 每个请求的处理间隔（秒，如 rate=5 则 0.2秒/个）

    // 2. 计算从上次处理到现在，应漏出的请求数
    $elapsedTime = $now - $lastProcessTime;
    $leakedCount = (int)($elapsedTime / $leakInterval); // 应漏出的请求数

    // 3. 更新当前请求数（减去漏出的，不小于0）
    $currentCount = max($currentCount - $leakedCount, 0);
    // 更新上次处理时间（若有漏出，更新为当前时间；否则保持原时间）
    $lastProcessTime = $leakedCount > 0 ? $now : $lastProcessTime;

    // 4. 判断是否能加入桶（当前请求数 < 桶容量则允许，否则限流）
    $allow = false;
    if ($currentCount < $capacity) {
        $currentCount++;
        $allow = true;
    }

    // 5. 保存更新后的数据
    $redis->multi();
    $redis->set($requestCountKey, $currentCount);
    $redis->set($lastProcessKey, $lastProcessTime);
    // 设置过期时间（避免长期无请求时键残留，过期时间=桶容量/速率 + 10秒）
    $expire = (int)($capacity / $rate) + 10;
    $redis->expire($requestCountKey, $expire);
    $redis->expire($lastProcessKey, $expire);
    $redis->exec();

    return $allow;
}

// 调用示例：限制 192.168.1.1 访问 /api/db/query 接口，桶容量10，每秒处理3个请求
$clientIp = $_SERVER['REMOTE_ADDR'];
$apiPath = '/api/db/query';
$limitKey = "{$clientIp}:{$apiPath}";

if (leakyBucketLimit($limitKey, 10, 3)) {
    echo "请求允许，执行数据库查询...";
} else {
    http_response_code(429);
    echo "请求过于频繁，已限制数据库访问！";
}
?>
```


### 五、限流优化与注意事项
#### 1. 优化建议
- **批量操作减少 Redis 交互**：用 `multi()` + `exec()` 执行 Redis 事务，减少网络往返次数；
- **IP 归属地限流（可选）**：对同一地区（如同一城市）的 IP 统一限流，防止多 IP 刷接口；
- **动态调整限流参数**：根据服务负载（如 CPU 使用率、内存占用）动态调整 `limit` 和 `rate`（如负载高时降低限额）；
- **限流提示优化**：返回 429 状态码时，通过 `Retry-After` 头告知客户端重试时间（如 `header('Retry-After: 60')`）。

#### 2. 注意事项
- **Redis 可用性**：限流依赖 Redis，需确保 Redis 高可用（如主从+哨兵），避免 Redis 宕机导致限流失效；
- **并发安全**：必须用 Redis 原子操作（`INCR`、`MULTI`），避免 PHP 代码层面的并发计数误差；
- **键名规范**：限流键需包含“限流粒度”（如 IP、用户 ID）和“接口标识”，避免键冲突；
- **避免误杀正常请求**：限流参数需合理（如根据接口历史 QPS 设定，避免过严导致正常用户无法访问）。

</details>


## 13. OPcache 作为 PHP 字节码缓存工具，其核心配置参数（如 opcache.memory_consumption、opcache.max_accelerated_files）有什么作用？如何根据项目规模优化 OPcache 配置？

<details>
<summary>答案与解析</summary>

- 思路要点：OPcache 的核心是“预编译 PHP 代码为字节码并缓存”，避免每次请求重新解析、编译源码，需先理解其工作原理，再拆解核心配置参数的作用，最后根据“项目文件数量”“代码体积”“服务器内存”三个维度给出优化方案，确保缓存命中率最大化。

### 一、OPcache 工作原理
PHP 脚本执行默认流程：
1. 读取 PHP 源码文件；
2. 词法分析→语法分析→生成抽象语法树（AST）；
3. 编译 AST 为字节码；
4. Zend 引擎执行字节码，返回结果。

OPcache 优化流程：
- 首次执行脚本：完成“编译→缓存”，将字节码存入内存；
- 后续执行脚本：直接从内存读取字节码执行，跳过“读取源码→编译”步骤，提升性能（通常可提升 50%+ 响应速度）。


### 二、OPcache 核心配置参数解析（php.ini）
OPcache 配置位于 `php.ini` 的 `[opcache]` 段，核心参数按“内存配置”“文件缓存”“有效性检查”“性能优化”分类：

#### 1. 内存配置（控制 OPcache 可用内存）
| 参数名                      | 默认值 | 单位 | 作用说明                                                                 |
|-----------------------------|--------|------|--------------------------------------------------------------------------|
| `opcache.memory_consumption` | 64     | MB   | OPcache 可使用的内存总量，用于存储字节码、编译信息等。需大于项目所有 PHP 文件的字节码总大小，否则会触发缓存淘汰。 |
| `opcache.interned_strings_buffer` | 4    | MB   | 用于存储“内部字符串”（如 PHP 关键字、常量、重复字符串）的内存大小。字符串重复率高的项目（如框架代码）需调大。 |
| `opcache.max_wasted_percentage` | 5    | %    | 允许的“浪费内存比例”，当缓存中碎片内存占比超过该值时，OPcache 会重启并整理内存，避免内存泄漏。 |

#### 2. 文件缓存（控制缓存的文件数量与类型）
| 参数名                          | 默认值 | 单位 | 作用说明                                                                 |
|---------------------------------|--------|------|--------------------------------------------------------------------------|
| `opcache.max_accelerated_files`  | 10000  | 个   | 最多可缓存的 PHP 文件数量。需大于项目中 PHP 文件总数（含框架、依赖库文件），否则未缓存的文件仍需重新编译。 |
| `opcache.blacklist_filename`    | 无     | -    | 黑名单文件路径，列出的文件/目录不会被 OPcache 缓存（如动态生成的 PHP 文件、配置文件）。 |
| `opcache.whitelist_filename`    | 无     | -    | 白名单文件路径，仅列出的文件/目录会被缓存（适合仅需缓存核心代码的场景，优先级高于黑名单）。 |

#### 3. 有效性检查（控制是否重新检查文件修改）
| 参数名                              | 默认值 | 单位 | 作用说明                                                                 |
|-------------------------------------|--------|------|--------------------------------------------------------------------------|
| `opcache.validate_timestamps`       | 1      | -    | 是否检查文件修改时间（1=开启，0=关闭）。生产环境建议设为 0（关闭检查，提升性能），发布代码后需手动重启 OPcache。 |
| `opcache.revalidate_freq`           | 2      | 秒   | 开启 `validate_timestamps=1` 时，检查文件修改的间隔时间（如 60 代表每60秒检查一次），避免频繁 IO。 |
| `opcache.revalidate_path`           | 0      | -    | 是否检查文件的真实路径（1=开启），用于符号链接或文件路径动态变化的场景，默认关闭以提升性能。 |

#### 4. 性能优化（控制编译与执行优化）
| 参数名                          | 默认值 | 单位 | 作用说明                                                                 |
|---------------------------------|--------|------|--------------------------------------------------------------------------|
| `opcache.enable`                | 1      | -    | 是否启用 OPcache（1=启用，0=禁用），生产环境必须设为 1。                 |
| `opcache.enable_cli`            | 0      | -    | 是否为 CLI 模式启用 OPcache（1=启用），仅用于 CLI 脚本频繁执行的场景（如定时任务），默认关闭。 |
| `opcache.optimization_level`    | 0x7FFFBFFF | -  | 优化级别（二进制掩码），控制 OPcache 对字节码的优化程度（如循环优化、常量折叠），默认开启全部优化，无需修改。 |
| `opcache.save_comments`         | 1      | -    | 是否保存代码注释（1=保存），框架（如 Laravel）依赖注释实现注解功能时，需设为 1，否则可设为 0 减少缓存体积。 |


### 三、根据项目规模优化 OPcache 配置
优化核心目标：**最大化缓存命中率（目标 95%+）**，避免“内存不足导致缓存淘汰”“文件数量超限导致未缓存”“频繁检查文件修改导致性能损耗”。

#### 1. 小型项目（文件数 < 1000，代码体积 < 10MB）
- 场景：个人博客、小型企业网站，使用轻量框架（如 Slim、Lumen）；
- 服务器配置：1核2G 内存；
- 优化配置：
  ```ini
  [opcache]
  zend_extension=opcache.so
  opcache.enable=1
  opcache.memory_consumption=64        ; 64MB 足够存储字节码
  opcache.interned_strings_buffer=8    ; 8MB 存储内部字符串
  opcache.max_accelerated_files=2000   ; 大于项目文件数（1000+）
  opcache.validate_timestamps=1        ; 开发/测试环境开启，方便代码更新
  opcache.revalidate_freq=60           ; 每60秒检查一次文件修改
  opcache.save_comments=1              ; 若用框架注解，需开启
  ```

#### 2. 中型项目（文件数 1000-10000，代码体积 10-50MB）
- 场景：电商网站、管理系统，使用主流框架（如 Laravel、ThinkPHP），含依赖库（Composer 包）；
- 服务器配置：2核4G 内存；
- 优化配置：
  ```ini
  [opcache]
  zend_extension=opcache.so
  opcache.enable=1
  opcache.memory_consumption=128       ; 128MB 足够存储框架+业务代码字节码
  opcache.interned_strings_buffer=16   ; 16MB 应对框架大量重复字符串
  opcache.max_accelerated_files=15000  ; 大于文件总数（框架+业务+Composer 包，通常 10000+）
  opcache.validate_timestamps=0        ; 生产环境关闭，发布代码后重启 OPcache
  opcache.max_wasted_percentage=10     ; 允许 10% 浪费内存，减少重启频率
  opcache.save_comments=1              ; Laravel 等框架依赖注释
  opcache.blacklist_filename=/etc/php.d/opcache-blacklist.txt  ; 黑名单（如动态配置文件）
  ```
- 黑名单文件（`opcache-blacklist.txt`）示例：
  ```txt
  ; 不缓存动态生成的文件
  /var/www/project/storage/
  ; 不缓存配置文件（如需实时生效）
  /var/www/project/config/dynamic.php
  ```

#### 3. 大型项目（文件数 > 10000，代码体积 > 50MB）
- 场景：大型电商、微服务，使用微服务框架（如 Hyperf、Swoft），多模块、多依赖；
- 服务器配置：4核8G+ 内存；
- 优化配置：
  ```ini
  [opcache]
  zend_extension=opcache.so
  opcache.enable=1
  opcache.memory_consumption=256       ; 256MB 存储大量字节码
  opcache.interned_strings_buffer=32   ; 32MB 存储海量内部字符串
  opcache.max_accelerated_files=30000  ; 大于文件总数（20000+）
  opcache.validate_timestamps=0        ; 生产环境强制关闭
  opcache.max_wasted_percentage=15     ; 允许更高浪费比例，减少重启
  opcache.enable_cli=1                 ; 若有频繁 CLI 任务（如定时任务、消息消费），开启 CLI 缓存
  opcache.whitelist_filename=/etc/php.d/opcache-whitelist.txt  ; 仅缓存核心代码
  opcache.fast_shutdown=1              ; 启用快速关闭（通过内存映射释放资源，提升脚本结束速度）
  ```
- 白名单文件（`opcache-whitelist.txt`）示例：
  ```txt
  ; 仅缓存核心业务代码和框架核心
  /var/www/project/app/
  /var/www/project/vendor/laravel/
  /var/www/project/vendor/symfony/
  ```


### 四、OPcache 状态监控与问题排查
#### 1. 查看 OPcache 状态
通过 PHP 脚本输出状态信息，核心关注“缓存命中率”“内存使用情况”“文件缓存数量”：
```php
<?php
// opcache-status.php
$status = opcache_get_status();
echo "<pre>";
print_r([
    "缓存命中率" => sprintf("%.2f%%", $status['opcache_statistics']['hit_rate']),
    "已使用内存" => sprintf("%.2f MB / %.2f MB", $status['memory_usage']['used_memory']/1024/1024, $status['memory_usage']['total_memory']/1024/1024),
    "已缓存文件数" => $status['opcache_statistics']['num_cached_files'],
    "最大缓存文件数" => $status['opcache_statistics']['max_cached_files'],
    "未缓存文件数" => $status['opcache_statistics']['num_uncached_files'],
]);
echo "</pre>";
?>
```
- 关键指标：
  - 缓存命中率：需 ≥ 95%，低于则需检查是否“内存不足”或“文件数超限”；
  - 未缓存文件数：需为 0 或极少，否则需调大 `max_accelerated_files`；
  - 内存使用：已使用内存需 < 总内存，避免触发缓存淘汰。

#### 2. 常见问题排查
| 问题现象                  | 可能原因                                  | 解决方案                                  |
|---------------------------|-------------------------------------------|-------------------------------------------|
| 缓存命中率低（<90%）      | 1. 内存不足，字节码被淘汰；2. 文件数超限，部分文件未缓存 | 1. 调大 `memory_consumption`；2. 调大 `max_accelerated_files` |
| 代码更新后不生效          | `validate_timestamps=0`，未重启 OPcache    | 1. 手动重启 OPcache（`opcache_reset()`）；2. 发布脚本中添加 OPcache 重启逻辑 |
| 内存占用持续增长          | 1. `max_wasted_percentage` 过小；2. 存在内存泄漏 | 1. 调大 `max_wasted_percentage`；2. 检查是否有动态生成的 PHP 文件未加入黑名单 |
| CLI 脚本执行慢            | `enable_cli=0`，CLI 模式未启用缓存        | 开启 `enable_cli=1`（仅用于 CLI 脚本频繁执行的场景） |


### 五、生产环境关键操作
1. **发布代码后重启 OPcache**：
   - 方法1：执行 `opcache_reset()`（需通过 Web 请求或 CLI 执行）；
   - 方法2：重启 PHP-FPM（`systemctl restart php-fpm`）；
   - 避免“代码更新后，旧字节码仍被执行”的问题。

2. **定期监控 OPcache 状态**：
   - 集成到监控系统（如 Prometheus + Grafana），监控“命中率”“内存使用”“文件缓存数”；
   - 当命中率低于 95% 或内存使用率超过 90% 时，触发告警。

3. **避免缓存动态文件**：
   - 动态生成的 PHP 文件（如模板编译后的文件）、实时配置文件，需加入黑名单，避免缓存后无法更新。

</details>


## 14. CSRF 防御的进阶方案有哪些？如何通过 PHP 代码实现 SameSite Cookie 配置、Origin/Referer 头严格校验？

<details>
<summary>答案与解析</summary>

- 思路要点：CSRF 攻击的核心是“利用用户登录状态，伪造跨站请求”，基础防御（如 CSRF Token）需结合进阶方案（SameSite Cookie、Origin/Referer 校验、请求方法限制）形成“多层防护”。需先明确各进阶方案的原理，再提供 PHP 代码实现，确保防御无死角。

### 一、CSRF 防御的进阶方案概述
基础防御（CSRF Token）虽有效，但存在“Token 泄露”“前端存储风险”等问题，进阶方案从“请求源头校验”“Cookie 安全属性”“请求合法性判断”三个维度增强防御：

| 进阶方案                | 核心原理                                  | 防御效果                                  | 适用场景                          |
|-------------------------|-------------------------------------------|-------------------------------------------|-----------------------------------|
| **SameSite Cookie**     | 限制 Cookie 仅在“同站请求”中携带，跨站请求不携带 Cookie，直接阻断 CSRF 攻击的“登录状态利用” | 高（从源头阻断，无需后端逻辑）            | 所有使用 Cookie 维持登录状态的场景 |
| **Origin/Referer 校验** | 校验请求头中的 Origin/Referer，仅允许信任域名（如自身域名）发起的请求，阻断第三方网站的伪造请求 | 中（需处理头缺失场景，部分浏览器可能不发送） | 接口请求、表单提交                |
| **请求方法限制**        | 对“写操作”（如 POST、PUT、DELETE）强制校验，GET 请求仅用于查询，不处理业务逻辑 | 辅助（减少 GET 请求被滥用的风险）          | RESTful 接口、表单提交            |
| **双重 Token 校验**     | 同时校验“Cookie 中的 Token”和“请求体中的 Token”，仅当两者一致时允许请求 | 高（防止 Token 被第三方网站窃取后滥用）    | 高安全场景（支付、修改密码）      |


### 二、核心进阶方案的 PHP 实现
#### 1. SameSite Cookie 配置（从源头阻断跨站 Cookie 携带）
##### （1）原理
SameSite 是 Cookie 的属性，用于限制 Cookie 在跨站请求中的携带行为，有三个取值：
- **SameSite=Strict**：仅在“同站请求”（同一域名下的请求）中携带 Cookie，跨站请求（如第三方网站跳转）完全不携带；
- **SameSite=Lax**：默认值，仅在“GET 请求+顶级导航”（如第三方网站链接跳转）中携带 Cookie，POST 等写操作请求不携带；
- **SameSite=None**：允许跨站请求携带 Cookie，但必须配合 `Secure` 属性（仅 HTTPS 传输），用于跨域业务（如第三方登录）。

**防御效果**：CSRF 攻击依赖“跨站请求携带用户 Cookie”，SameSite 配置直接阻止该行为，无需后端额外校验，是最彻底的防御方案之一。

##### （2）PHP 实现
通过 `setcookie()` 函数的第 6、7 个参数（`secure`、`httponly`）配合 `SameSite` 属性实现，需注意：
- PHP 7.3+ 支持直接在 `setcookie()` 中设置 `SameSite`（通过第 8 个参数 `samesite`）；
- 低版本 PHP 需通过 `header()` 函数手动设置 Cookie 头。

```php
<?php
/**
 * PHP 7.3+ 直接设置 SameSite Cookie（推荐）
 * 场景：设置登录 Token Cookie，仅同站请求携带
 */
setcookie(
    'user_token',                  // Cookie 名称
    'abc123xyz789',                // Cookie 值
    time() + 3600 * 24 * 7,        // 有效期（7天）
    '/',                           // 作用域（根目录）
    'yourdomain.com',              // 域名
    true,                          // secure：仅 HTTPS 传输（必须开启，尤其 SameSite=None 时）
    true,                          // httponly：禁止 JS 访问，防 XSS 窃取
    'Strict'                       // SameSite：仅同站请求携带
);

/**
 * PHP 7.3 以下版本：手动设置 Cookie 头（兼容方案）
 */
$cookieValue = 'abc123xyz789';
$expire = time() + 3600 * 24 * 7;
// 手动拼接 Cookie 头，包含 SameSite 属性
header(
    sprintf(
        'Set-Cookie: user_token=%s; expires=%s; path=/; domain=yourdomain.com; secure; HttpOnly; SameSite=Strict',
        urlencode($cookieValue),
        gmdate('D, d M Y H:i:s T', $expire) // GMT 时间格式
    )
);

/**
 * 跨域场景（如第三方登录）：设置 SameSite=None（必须配合 Secure）
 */
setcookie(
    'cross_domain_token',
    'def456ghi789',
    time() + 3600,
    '/',
    'yourdomain.com',
    true, // 必须开启 Secure（HTTPS 传输）
    true,
    'None' // 允许跨站请求携带，但仅 HTTPS 下生效
);
?>
```


#### 2. Origin/Referer 头严格校验（校验请求源头）
##### （1）原理
- **Origin 头**：仅在跨站请求中发送，包含请求的“源域名”（如 `https://yourdomain.com`），不包含路径和参数，优先级高于 Referer；
- **Referer 头**：包含请求的“完整来源 URL”（如 `https://yourdomain.com/login`），同站/跨站请求均可能发送，但部分浏览器或场景（如 HTTPS 跳 HTTP）会省略。

**防御逻辑**：后端校验请求头中的 Origin/Referer，仅当“源域名是信任域名”（如自身域名、合作域名）时，允许请求，否则阻断。

##### （2）PHP 实现
需处理“头缺失”“域名解析”“子域名信任”等边界情况，核心是“提取域名→判断是否在信任列表”：

```php
<?php
/**
 * 提取 URL 中的主域名（如从 https://blog.yourdomain.com 提取 yourdomain.com）
 * 处理子域名场景，支持主流顶级域名（.com/.cn/.co.uk 等）
 */
function extractMainDomain(string $url): ?string {
    // 解析 URL
    $parse = parse_url($url);
    if (empty($parse['host'])) {
        return null;
    }
    $host = $parse['host'];

    // 常见顶级域名后缀列表（可扩展）
    $topLevelDomains = [
        'com', 'cn', 'net', 'org', 'edu', 'gov', 'co.uk', 'co.jp', 'com.cn', 'org.cn'
    ];

    // 分割域名（如 blog.yourdomain.com → [blog, yourdomain, com]）
    $parts = explode('.', $host);
    $partCount = count($parts);

    // 匹配顶级域名后缀（如 co.uk、com.cn）
    foreach ($topLevelDomains as $tld) {
        $tldParts = explode('.', $tld);
        $tldPartCount = count($tldParts);
        // 若域名后缀与顶级域名匹配，提取主域名（如 yourdomain.com）
        if ($tldPartCount <= $partCount && array_slice($parts, -$tldPartCount) === $tldParts) {
            $mainDomain = implode('.', array_slice($parts, -$tldPartCount - 1, $tldPartCount + 1));
            return $mainDomain;
        }
    }

    // 无匹配顶级域名时，取最后两级（如 yourdomain.com）
    return $partCount >= 2 ? implode('.', array_slice($parts, -2)) : $host;
}

/**
 * Origin/Referer 头严格校验
 * @param array $trustedDomains 信任域名列表（如 ['yourdomain.com', 'partner.com']）
 * @return bool true：校验通过，false：校验失败
 */
function validateOriginReferer(array $trustedDomains): bool {
    // 1. 优先校验 Origin 头（跨站请求必带，更可靠）
    $origin = $_SERVER['HTTP_ORIGIN'] ?? '';
    if (!empty($origin)) {
        $mainDomain = extractMainDomain($origin);
        return $mainDomain !== null && in_array($mainDomain, $trustedDomains);
    }

    // 2. Origin 头缺失时，校验 Referer 头
    $referer = $_SERVER['HTTP_REFERER'] ?? '';
    if (empty($referer)) {
        // Referer 头也缺失，高安全场景可直接阻断，或允许 GET 请求（需结合业务）
        return $_SERVER['REQUEST_METHOD'] === 'GET' ? true : false;
    }

    $mainDomain = extractMainDomain($referer);
    return $mainDomain !== null && in_array($mainDomain, $trustedDomains);
}

// 调用示例：仅允许 yourdomain.com 和 partner.com 发起的请求（如修改密码接口）
$trustedDomains = ['yourdomain.com', 'partner.com'];
if (!validateOriginReferer($trustedDomains)) {
    http_response_code(403); // 403 Forbidden
    die("CSRF 防御：请求源头不被信任");
}

// 后续业务逻辑（如修改密码）
// ...
?>
```


#### 3. 双重 Token 校验（高安全场景，防止 Token 泄露）
##### （1）原理
基础 CSRF Token 仅校验“请求体中的 Token”，若 Token 被第三方网站窃取（如 XSS 攻击），仍可能被滥用。双重 Token 校验需同时满足：
- 请求体/URL 中携带的 Token（如 `_csrf_token`）；
- Cookie 中携带的 Token（如 `csrf_cookie_token`）；
- 两者的值一致，且均有效（未过期、未被篡改）。

**防御效果**：即使请求体中的 Token 泄露，第三方网站无法读取用户 Cookie 中的 Token（因 `HttpOnly` 属性），无法通过校验。

##### （2）PHP 实现
```php
<?php
session_start();

/**
 * 生成双重 Token（存储到 Session 和 Cookie）
 */
function generateDoubleCsrfToken(): void {
    // 生成随机 Token（32字节，安全随机数）
    $csrfToken = bin2hex(random_bytes(32));

    // 1. 存储到 Session（用于后续校验）
    $_SESSION['csrf_token'] = $csrfToken;
    $_SESSION['csrf_token_expire'] = time() + 3600; // 1小时过期

    // 2. 存储到 Cookie（HttpOnly，禁止 JS 访问）
    setcookie(
        'csrf_cookie_token',
        $csrfToken,
        time() + 3600,
        '/',
        'yourdomain.com',
        true, // Secure
        true  // HttpOnly
    );
}

/**
 * 双重 Token 校验
 * @return bool true：校验通过，false：校验失败
 */
function validateDoubleCsrfToken(): bool {
    // 1. 检查 Session 和 Cookie 中的 Token 是否存在
    if (empty($_SESSION['csrf_token']) || empty($_COOKIE['csrf_cookie_token'])) {
        return false;
    }

    // 2. 检查 Token 是否过期
    if (time() > $_SESSION['csrf_token_expire']) {
        return false;
    }

    // 3. 检查请求中的 Token（POST 体或 GET 参数）
    $requestToken = $_POST['_csrf_token'] ?? $_GET['_csrf_token'] ?? '';
    if (empty($requestToken)) {
        return false;
    }

    // 4. 校验三者一致（Session Token = Cookie Token = Request Token）
    return hash_equals($_SESSION['csrf_token'], $requestToken) 
        && hash_equals($_SESSION['csrf_token'], $_COOKIE['csrf_cookie_token']);
}

// 调用示例：修改密码接口（高安全场景）
// 首次访问页面时生成 Token
if (empty($_SESSION['csrf_token']) || time() > $_SESSION['csrf_token_expire']) {
    generateDoubleCsrfToken();
}

// 处理表单提交（POST 请求）
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    if (!validateDoubleCsrfToken()) {
        http_response_code(403);
        die("CSRF 防御：双重 Token 校验失败");
    }

    // Token 校验通过，执行修改密码逻辑
    $newPassword = $_POST['new_password'] ?? '';
    // ... 修改密码代码
    echo "密码修改成功！";
}

// 表单中嵌入请求 Token
echo '<form method="post" action="change-password.php">';
echo '<input type="hidden" name="_csrf_token" value="' . $_SESSION['csrf_token'] . '">';
echo '<input type="password" name="new_password" placeholder="新密码" required>';
echo '<button type="submit">提交</button>';
echo '</form>';
?>
```


### 三、CSRF 多层防御最佳实践
1. **优先启用 SameSite Cookie**：从源头阻断跨站 Cookie 携带，是最基础且有效的防御，推荐所有 Cookie 配置 `SameSite=Strict` 或 `Lax`；
2. **Origin/Referer 校验作为补充**：对关键接口（如支付、转账）添加该校验，应对 SameSite 不兼容的旧浏览器；
3. **双重 Token 用于高安全场景**：支付、修改密码、用户信息修改等接口，需结合双重 Token 校验，防止 Token 泄露风险；
4. **限制请求方法**：GET 请求仅用于查询，不处理任何写操作（如创建、修改、删除数据），避免 GET 请求被 CSRF 滥用；
5. **定期刷新 Token**：CSRF Token 设置过期时间（如 1小时），定期刷新，减少 Token 泄露后的滥用窗口。


### 四、常见问题与注意事项
1. **SameSite 兼容性**：IE 浏览器不支持 SameSite 属性，需通过 Origin/Referer 校验兜底；
2. **Origin/Referer 缺失**：部分浏览器在“直接输入 URL”“书签访问”“HTTPS 跳 HTTP”场景下不发送 Referer，需允许 GET 请求通过，或结合其他防御方案；
3. **跨域业务处理**：跨域接口（如第三方登录、API 合作）需设置 `SameSite=None`，但必须配合 `Secure` 属性（仅 HTTPS 传输），同时严格校验 Origin；
4. **Token 存储安全**：请求体中的 Token 需通过 HTTPS 传输，避免明文泄露；Cookie 中的 Token 必须设置 `HttpOnly` 和 `Secure` 属性，防 XSS 窃取。

</details>


## 15. PHP 项目中如何拦截恶意请求（如高频 IP 攻击、爬虫请求）？如何结合 Redis 实现 IP 黑名单与请求频率阈值控制？

<details>
<summary>答案与解析</summary>

- 思路要点：恶意请求拦截的核心是“识别恶意行为→阻断请求”，需从“IP 维度”（高频请求、黑名单）和“行为维度”（爬虫特征、异常请求格式）双管齐下。结合 Redis 实现分布式拦截（支持多服务器共享状态），避免单机拦截的局限性，确保拦截规则一致。

### 一、恶意请求的常见类型与识别特征
| 恶意请求类型        | 识别特征                                  | 拦截目标                                  |
|---------------------|-------------------------------------------|-------------------------------------------|
| **高频 IP 攻击**    | 同一 IP 单位时间内请求次数远超正常用户（如 1秒100次） | 限制 IP 请求频率，超限额则临时封禁          |
| **IP 黑名单攻击**   | IP 已被标记为恶意（如之前发起过攻击、属于已知攻击 IP 段） | 直接阻断黑名单 IP 的所有请求                |
| **爬虫请求**        | 1. User-Agent 异常（如无 UA、含“spider”“bot”）；2. 无 Cookie/Session（非人类操作）；3. 请求间隔固定（机器特征） | 识别爬虫特征，返回 403 或虚假数据          |
| **异常请求格式**    | 1. 参数缺失/格式错误（如必填参数为空）；2. 请求体篡改（如签名校验失败）；3. 访问不存在的接口路径 | 校验请求合法性，非法请求直接阻断            |


### 二、结合 Redis 实现 IP 黑名单与请求频率控制
#### 1. IP 黑名单拦截（直接阻断已知恶意 IP）
**原理**：
- 维护 Redis 黑名单集合（如 `blacklist:ip`），存储恶意 IP；
- 每个请求进来时，先检查 IP 是否在黑名单中，是则直接返回 403；
- 支持手动添加（如后台管理系统操作）和自动添加（如高频请求触发后加入黑名单）。

**PHP 实现**：
```php
<?php
/**
 * IP 黑名单拦截
 * @param string $ip 客户端 IP
 * @param int $expire 黑名单过期时间（秒，0 代表永久）
 * @return bool true：IP 在黑名单中（阻断），false：允许请求
 */
function isIpInBlacklist(string $ip, int $expire = 0): bool {
    $redis = new Redis();
    $redis->connect('127.0.0.1', 6379);

    $blacklistKey = 'blacklist:ip';

    // 1. 检查 IP 是否在黑名单中
    if ($redis->sIsMember($blacklistKey, $ip)) {
        return true;
    }

    // 2. （可选）自动添加高频 IP 到黑名单（结合请求频率控制）
    $freqKey = "freq:ip:{$ip}";
    $requestCount = $redis->get($freqKey) ?: 0;
    if ($requestCount > 1000) { // 若 1小时内请求超 1000 次，自动加入黑名单
        if ($expire > 0) {
            $redis->sAdd($blacklistKey, $ip);
            // 为黑名单 IP 设置过期时间（如 24小时后自动解除）
            $redis->expire($blacklistKey, $expire);
        } else {
            $redis->sAdd($blacklistKey, $ip); // 永久黑名单
        }
        return true;
    }

    return false;
}

// 调用示例：拦截黑名单 IP，临时黑名单 24小时过期
$clientIp = getClientIp(); // 获取客户端真实 IP
if (isIpInBlacklist($clientIp, 24 * 3600)) {
    http_response_code(403);
    echo "您的 IP 因异常行为已被限制访问，请24小时后重试！";
    exit();
}

// 辅助函数：获取客户端真实 IP（支持反向代理，如 Nginx、Cloudflare）
function getClientIp(): string {
    $ipHeaders = [
        'HTTP_X_FORWARDED_FOR',
        'HTTP_X_REAL_IP',
        'HTTP_CLIENT_IP',
        'REMOTE_ADDR'
    ];
    foreach ($ipHeaders as $header) {
        $ip = $_SERVER[$header] ?? '';
        if (!empty($ip) && filter_var($ip, FILTER_VALIDATE_IP)) {
            // 处理 X-Forwarded-For 多 IP 情况（取第一个有效 IP）
            if ($header === 'HTTP_X_FORWARDED_FOR') {
                $ips = explode(',', $ip);
                $ip = trim($ips[0]);
            }
            return $ip;
        }
    }
    return '0.0.0.0';
}
?>
```

#### 2. IP 请求频率控制（限制高频请求）
**原理**：
- 以“IP+时间窗口”为 Redis 键（如 `freq:ip:192.168.1.1:2024052010`，代表 2024-05-20 10 分钟窗口）；
- 统计该窗口内的请求次数，超限额则临时限制（如 10 分钟内不允许请求）；
- 结合 Redis 原子操作（`INCR`、`EXPIRE`）确保计数准确，支持多服务器共享状态。

**PHP 实现**（基于滑动窗口计数，避免固定窗口的流量突刺）：
```php
<?php
/**
 * IP 请求频率控制（滑动窗口计数）
 * @param string $ip 客户端 IP
 * @param int $limit 单位时间内最大请求次数
 * @param int $window 时间窗口（秒）
 * @return bool true：允许请求，false：超限额（临时限制）
 */
function ipRateLimit(string $ip, int $limit, int $window): bool {
    $redis = new Redis();
    $redis->connect('127.0.0.1', 6379);

    $prefix = "freq:ip:{$ip}";
    $now = time();
    $currentWindow = $now - ($now % $window); // 当前窗口起始时间（如 window=60，取整到分钟）
    $currentKey = "{$prefix}:{$currentWindow}";

    // 1. 原子递增当前窗口请求数
    $count = $redis->incr($currentKey);
    // 2. 首次请求时设置窗口过期时间（窗口结束后自动清理）
    if ($count === 1) {
        $redis->expire($currentKey, $window * 2); // 过期时间设为 2 个窗口，确保滑动时数据不丢失
    }

    // 3. （滑动窗口核心）统计当前窗口 + 上一个窗口的总请求数（避免窗口切换时的流量突刺）
    $prevWindow = $currentWindow - $window;
    $prevKey = "{$prefix}:{$prevWindow}";
    $prevCount = $redis->get($prevKey) ?: 0;
    $totalCount = $count + $prevCount;

    // 4. 判断是否超限额
    if ($totalCount > $limit) {
        // 超限额时，记录临时限制（如 10 分钟内不允许请求）
        $blockKey = "block:ip:{$ip}";
        $redis->setex($blockKey, 600, 1); // 10分钟后自动解除
        return false;
    }

    // 5. 检查是否处于临时限制中
    $blockKey = "block:ip:{$ip}";
    if ($redis->exists($blockKey)) {
        return false;
    }

    return true;
}

// 调用示例：限制 192.168.1.1 1分钟内最多 60 次请求（正常用户约 1次/秒）
$clientIp = getClientIp();
if (!ipRateLimit($clientIp, 60, 60)) {
    http_response_code(429);
    echo "请求过于频繁，请10分钟后再试！";
    exit();
}

// 后续业务逻辑
// ...
?>
```


### 三、爬虫请求拦截（识别机器行为）
#### 1. 基于 User-Agent 识别
**原理**：正常浏览器有规范的 User-Agent（如 `Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36`），爬虫通常无 UA 或含“spider”“bot”“crawler”等关键词。

**PHP 实现**：
```php
<?php
/**
 * 基于 User-Agent 识别爬虫
 * @return bool true：疑似爬虫，false：正常浏览器
 */
function isCrawlerByUA(): bool {
    $userAgent = $_SERVER['HTTP_USER_AGENT'] ?? '';
    if (empty($userAgent)) {
        return true; // 无 User-Agent 大概率是爬虫
    }

    // 爬虫关键词列表（可扩展）
    $crawlerKeywords = [
        'spider', 'bot', 'crawler', 'slurp', 'search', 'worm', 
        'fetch', 'curl', 'wget', 'python', 'java', 'scrapy'
    ];

    // 检查 UA 中是否包含爬虫关键词（不区分大小写）
    foreach ($crawlerKeywords as $keyword) {
        if (stripos($userAgent, $keyword) !== false) {
            return true;
        }
    }

    // 合法爬虫白名单（如搜索引擎爬虫，需允许访问）
    $whitelistUA = [
        'Googlebot', 'Baiduspider', 'Bingbot', 'Sogou Spider', 
        'Yahoo! Slurp', 'DuckDuckBot', 'YandexBot'
    ];
    foreach ($whitelistUA as $ua) {
        if (stripos($userAgent, $ua) !== false) {
            return false; // 合法爬虫，不拦截
        }
    }

    return false;
}
?>
```

#### 2. 基于 Cookie/会话识别（区分人类与机器）
**原理**：人类用户会在浏览器中保留Cookie（如会话ID），且请求会随页面交互自然产生；爬虫通常无Cookie，或会话生命周期极短（一次性请求）。  
**实现逻辑**：首次访问时植入“追踪Cookie”，后续请求检查Cookie是否存在，结合会话时长判断是否为机器行为。

```php
<?php
/**
 * 基于 Cookie/会话识别爬虫
 * @return bool true：疑似爬虫，false：正常用户
 */
function isCrawlerByCookie(): bool {
    session_start();
    $trackCookie = 'user_track';

    // 1. 首次访问：植入追踪 Cookie（有效期 7 天）
    if (!isset($_COOKIE[$trackCookie])) {
        setcookie(
            $trackCookie,
            bin2hex(random_bytes(16)), // 随机追踪标识
            time() + 3600 * 24 * 7,
            '/',
            '',
            false, // 非 HTTPS 环境可选 false
            false  // 允许 JS 读取（不影响安全性）
        );
        // 记录首次访问时间
        $_SESSION['first_visit_time'] = time();
        return false; // 首次访问暂不判定为爬虫
    }

    // 2. 检查会话时长（正常用户会话会持续一段时间）
    $firstVisit = $_SESSION['first_visit_time'] ?? time();
    $sessionDuration = time() - $firstVisit;
    if ($sessionDuration < 5 && $_SERVER['REQUEST_METHOD'] === 'POST') {
        // 会话 <5 秒就发起 POST 请求，疑似爬虫（人类操作需浏览页面）
        return true;
    }

    // 3. 检查请求间隔（机器请求间隔通常固定，人类操作有随机性）
    $lastRequestTime = $_SESSION['last_request_time'] ?? 0;
    $currentTime = time();
    $interval = $currentTime - $lastRequestTime;
    $_SESSION['last_request_time'] = $currentTime;

    // 连续 5 次请求间隔均 <1 秒，疑似爬虫
    if ($lastRequestTime > 0 && $interval < 1) {
        $_SESSION['fast_request_count'] = ($_SESSION['fast_request_count'] ?? 0) + 1;
        if ($_SESSION['fast_request_count'] >= 5) {
            return true;
        }
    } else {
        $_SESSION['fast_request_count'] = 0; // 重置计数
    }

    return false;
}
?>
```

#### 3. 基于请求内容识别（异常参数/路径）
**原理**：爬虫常访问不存在的路径（如 `/admin.php`、`/config.ini`），或提交异常参数（如超长字符串、SQL注入特征）。  
**实现逻辑**：校验请求路径是否在白名单内，检查参数格式与长度，拦截异常请求。

```php
<?php
/**
 * 基于请求内容识别恶意请求
 * @return bool true：请求异常，false：正常请求
 */
function isMaliciousByContent(): bool {
    // 1. 校验请求路径（仅允许白名单内的接口/页面）
    $allowedPaths = [
        '/', '/index.php', '/login.php', '/api/user', '/api/product'
    ];
    $currentPath = parse_url($_SERVER['REQUEST_URI'], PHP_URL_PATH) ?? '';
    if (!in_array($currentPath, $allowedPaths)) {
        return true; // 访问未授权路径
    }

    // 2. 检查参数是否含恶意特征（SQL注入、XSS 等）
    $maliciousPatterns = [
        '/union\s+select/i', '/insert\s+into/i', '/delete\s+from/i', // SQL注入
        '/<script>/i', '/alert\(/i', '/onerror=/i', // XSS
        '/\.\.\/', '/etc\/passwd', '/proc\/self\/environ' // 路径遍历
    ];
    $params = array_merge($_GET, $_POST);
    foreach ($params as $key => $value) {
        $value = (string)$value;
        foreach ($maliciousPatterns as $pattern) {
            if (preg_match($pattern, $value)) {
                return true;
            }
        }
        // 检查参数长度（超长参数可能是攻击载荷）
        if (strlen($value) > 1000) {
            return true;
        }
    }

    return false;
}
?>
```


### 四、综合拦截策略（多层防御）
单一策略可能存在误判，需结合多层防御形成闭环，流程如下：  
1. **IP 黑名单检查** → 2. **请求频率控制** → 3. **爬虫特征识别** → 4. **请求内容校验** → 5.（可选）验证码验证  

```php
<?php
// 综合拦截入口
function interceptMaliciousRequest(): void {
    $clientIp = getClientIp();

    // 1. IP 黑名单拦截
    if (isIpInBlacklist($clientIp, 24 * 3600)) {
        http_response_code(403);
        logMaliciousRequest($clientIp, 'IP 在黑名单中');
        exit('访问被拒绝：您的 IP 存在异常行为');
    }

    // 2. 请求频率控制（1分钟最多60次，约1次/秒）
    if (!ipRateLimit($clientIp, 60, 60)) {
        http_response_code(429);
        logMaliciousRequest($clientIp, '请求频率超限');
        exit('请求过于频繁，请10分钟后再试');
    }

    // 3. 爬虫特征识别（UA + Cookie 双重校验）
    if (isCrawlerByUA() || isCrawlerByCookie()) {
        // 对疑似爬虫，先尝试验证码验证
        if (!verifyCaptcha()) {
            http_response_code(403);
            logMaliciousRequest($clientIp, '疑似爬虫且验证码未通过');
            exit('请完成验证码验证');
        }
    }

    // 4. 请求内容校验（拦截异常参数/路径）
    if (isMaliciousByContent()) {
        http_response_code(400);
        logMaliciousRequest($clientIp, '请求内容异常');
        exit('无效的请求参数');
    }
}

// 验证码验证（集成第三方验证码如极验、腾讯云验证码）
function verifyCaptcha(): bool {
    // 简单示例：实际项目需对接真实验证码服务
    if ($_SERVER['REQUEST_METHOD'] === 'POST' && !empty($_POST['captcha'])) {
        return $_POST['captcha'] === '1234'; // 仅为示例，真实场景需校验验证码票据
    }
    // 输出验证码表单（对疑似爬虫的 GET 请求）
    echo '<form method="post">';
    echo '请输入验证码：<input type="text" name="captcha" required>';
    echo '<button type="submit">提交</button>';
    echo '</form>';
    exit();
}

// 恶意请求日志记录（存储到 Redis 或文件）
function logMaliciousRequest(string $ip, string $reason): void {
    $redis = new Redis();
    $redis->connect('127.0.0.1', 6379);
    $logKey = 'malicious:log:' . date('Ymd');
    $logData = [
        'ip' => $ip,
        'time' => date('Y-m-d H:i:s'),
        'url' => $_SERVER['REQUEST_URI'] ?? '',
        'reason' => $reason
    ];
    $redis->rPush($logKey, json_encode($logData));
    $redis->expire($logKey, 30 * 86400); // 日志保留30天
}

// 执行拦截逻辑（放在项目入口文件，如 index.php 最顶部）
interceptMaliciousRequest();

// 正常业务逻辑（通过所有拦截后执行）
echo "请求正常，执行业务逻辑...";
?>
```


### 五、运维与优化建议
1. **动态调整策略**：  
   - 根据日志分析正常用户与恶意请求的特征，定期更新 `crawlerKeywords`、`maliciousPatterns` 等规则；  
   - 对误判的合法 IP（如搜索引擎爬虫），手动添加到白名单（`whitelist:ip` 集合）。

2. **Redis 性能优化**：  
   - 为所有键设置合理的过期时间（如频率控制键 2 个窗口周期，日志键 30 天），避免内存溢出；  
   - 对高频访问的 IP，使用 Redis Pipeline 批量执行命令，减少网络往返。

3. **可视化监控**：  
   - 集成监控工具（如 Grafana），实时展示“拦截量”“TOP 恶意 IP”“爬虫识别率”等指标；  
   - 当拦截量突增时（可能是攻击），自动触发告警（如邮件、短信）。

4. **白名单机制补充**：  
   对已知的合法来源（如合作平台 IP、内部办公 IP），添加到白名单直接放行：  
   ```php
   function isIpInWhitelist(string $ip): bool {
       $redis = new Redis();
       $redis->connect('127.0.0.1', 6379);
       return $redis->sIsMember('whitelist:ip', $ip);
   }
   ```


### 总结
拦截恶意请求需结合“IP 管控”“行为分析”“内容校验”多层策略，核心是通过 Redis 实现分布式状态共享（确保多服务器规则一致），同时兼顾“拦截准确性”与“用户体验”（避免误判正常请求）。实际项目中需根据业务场景调整阈值（如高频请求限制、爬虫识别规则），并通过日志持续优化策略。

</details>