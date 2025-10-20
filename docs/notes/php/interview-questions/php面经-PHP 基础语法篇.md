---
title: php面经-PHP 基础语法篇
createTime: 2025/08/29 10:46:11
permalink: /php/面试题/basic-grammar/
---
### 1. 请解释 PHP 中 == 和 === 的区别，并举例说明在什么场景下两者结果不同（如类型转换导致的问题）

<details>
<summary>思考与答案（点击展开）</summary>

- 思考：
  - `==` 比较值，允许类型转换；`===` 同时比较值与类型，不做类型转换。
  - 容易踩坑：`"0" == 0`、`false == "0"`、`null == 0` 等宽松比较。
  - 思路要点：== 是弱等于，比较时会先进行自动类型转换，仅判断转换后的值是否相等，不关注原始类型是否一致； === 是全等于，比较时不进行类型转换，需同时判断值和原始类型是否完全相同； 两者结果不同的典型场景：当比较的两个变量值在类型转换后相等，但原始类型不同时，== 返回 true，而 === 返回 false（如字符串与数字、0 与 false、null 与空字符串等）。
- 答案与示例：

```php
<?php
// 场景1：字符串与数字（值相等，类型不同）
echo "1. 字符串与数字的比较：\n";
var_dump("4" == 4);   // bool(true) （弱比较：字符串"4"转为数字4，值相等）
var_dump("4" === 4);  // bool(false)（强比较：string类型 vs int类型，类型不同）

// 场景2：0与false（弱比较中值等价，类型不同）
echo "\n2. 0与false的比较：\n";
var_dump(0 == false);  // bool(true) （弱比较：0和false都视为"假值"，等价）
var_dump(0 === false); // bool(false)（强比较：int类型 vs bool类型，类型不同）

// 场景3：null与空字符串（弱比较中值等价，类型不同）
echo "\n3. null与空字符串的比较：\n";
var_dump(null == '');  // bool(true) （弱比较：都视为"假值"，等价）
var_dump(null === ''); // bool(false)（强比较：null类型 vs string类型，类型不同）

// 场景4：true与数字1（弱比较中值等价，类型不同）
echo "\n4. true与1的比较：\n";
var_dump(true == 1);  // bool(true) （弱比较：true转为数字1，值相等）
var_dump(true === 1); // bool(false)（强比较：bool类型 vs int类型，类型不同）

// 场景5：空数组与false（弱比较中值等价，类型不同）
echo "\n5. 空数组与false的比较：\n";
var_dump([] == false);  // bool(true) （弱比较：空数组视为"假值"，与false等价）
var_dump([] === false); // bool(false)（强比较：array类型 vs bool类型，类型不同）
?>

```

- 实战建议：
  - 业务判断优先用 `===`；只在确实需要“宽松比较”时用 `==`；布尔/数字/字符串交叉比较更应使用 `===`。

</details>

### 2. PHP 的变量作用域有哪些（全局、局部、静态、超全局）？如何在函数内部访问全局变量？global 关键字和 $GLOBALS 数组有什么区别？

<details>
<summary>答案与思考</summary>

- 思路要点：

  1. PHP 变量作用域分为4类：
     - 全局作用域（Global）：在函数外部定义的变量，默认仅在函数外部可直接访问。
     - 局部作用域（Local）：在函数内部定义的变量，仅在该函数内部有效，函数执行结束后销毁。
     - 静态作用域（Static）：在函数内部用 `static` 声明的变量，函数执行后不销毁，保留值供下次调用使用（仅初始化一次）。
     - 超全局作用域（Superglobal）：PHP 预定义的数组（如 `$_GET`、`$_POST`、`$GLOBALS`、`$_SESSION` 等），在脚本任何位置（包括函数、类方法内）可直接访问，无需特殊声明。
  2. 函数内部访问全局变量的两种方式：
     - 使用 `global` 关键字：在函数内声明变量为全局变量，将外部全局变量引入函数局部作用域。
     - 使用 `$GLOBALS` 超全局数组：通过 `$GLOBALS['变量名']` 直接访问，`$GLOBALS` 本身在任何作用域均可直接使用。
  3. `global` 与 `$GLOBALS` 的区别：
     - 工作机制：`global` 是创建局部变量引用全局变量；`$GLOBALS` 是直接访问全局变量的内存地址。
     - 销毁影响：`unset(global声明的变量)` 仅销毁局部引用，全局变量仍存在；`unset($GLOBALS['变量名'])` 直接销毁全局变量。
     - 使用场景：`$GLOBALS` 在类方法中更通用；`global` 不能用于类属性声明，需注意上下文限制。
- 示例代码：

  ```php
  <?php
  // 1. 全局变量（全局作用域）
  $site = "PHP教程";

  // 2. 局部变量与全局变量访问
  function testScope() {
      // 局部变量（仅函数内可见）
      $local = "局部变量";
      echo "函数内访问局部变量：{$local}<br>";

      // 用global访问全局变量
      global $site;
      echo "global访问全局变量：{$site}<br>";

      // 用$GLOBALS访问全局变量
      echo "$GLOBALS访问全局变量：{$GLOBALS['site']}<br>";
  }
  testScope();
  echo "函数外访问全局变量：{$site}<br>"; // 直接访问
  // echo $local; // 报错：局部变量在外部不可访问


  // 3. 静态变量示例
  function testStatic() {
      static $count = 0; // 静态变量，仅初始化一次
      $count++;
      echo "静态变量当前值：{$count}<br>";
  }
  testStatic(); // 输出：1
  testStatic(); // 输出：2（保留上次结果）
  testStatic(); // 输出：3


  // 4. global与$GLOBALS的区别（销毁测试）
  function testDiff() {
      global $site;
      $site = "被global修改";
      echo "global修改后：{$site}<br>";

      $GLOBALS['site'] = "被\$GLOBALS修改";
      echo "$GLOBALS修改后：{$GLOBALS['site']}<br>";

      // 销毁global声明的变量（仅销毁局部引用）
      unset($site);
      echo "unset(global变量)后是否存在：" . (isset($site) ? "是" : "否") . "<br>";
  }
  testDiff();
  echo "函数外全局变量：{$site}<br>"; // 仍为"被$GLOBALS修改"

  // 销毁$GLOBALS中的全局变量
  unset($GLOBALS['site']);
  echo "unset(\$GLOBALS变量)后是否存在：" . (isset($site) ? "是" : "否") . "<br>"; // 否


  // 5. 超全局变量示例
  function testSuperglobal() {
      echo "超全局变量\$_SERVER['PHP_SELF']：{$_SERVER['PHP_SELF']}<br>";
      echo "超全局变量\$_GET：" . (is_array($_GET) ? "是数组" : "不是") . "<br>";
  }
  testSuperglobal();
  ?>
  ```

</details>

### 3. 什么是可变变量（Variable variables）？用代码示例说明其用法（如 $$var ），并说明在实际开发中应注意什么（如安全风险）

<details>
<summary>答案与思考</summary>

- 思路要点：

  1. 可变变量的定义：指变量名可以通过另一个变量的值动态确定的变量机制。在PHP中，通过 `$$var`语法实现——当 `$var`的值为字符串 `"name"`时，`$$var`等价于 `$name`，即通过 `$var`的取值动态引用了名为 `$name`的变量。
  2. 核心特征：允许在运行时动态生成和访问变量名，增加了代码的灵活性，但同时也带来了复杂性和风险。
  3. 使用注意：对数组使用可变变量时，需注意PHP的解析优先级（例如 `$$arr[0]`与 `${$arr}[0]`的区别），避免逻辑混淆。
- 示例代码：

  ```php
  <?php
  // 基础用法：通过变量值动态生成变量名
  $key = "username";
  $username = "kkt";
  var_dump($$key); // 输出：string(3) "kkt"（等价于$username）

  $prefix = "msg";
  $msg_welcome = "欢迎访问";
  $dynamicVar = $prefix . "_welcome"; // 动态拼接变量名
  var_dump($$dynamicVar); //输出：string(6) "欢迎访问"（等价于$msg_welcome）


  // 数组相关的可变变量（重点注意解析优先级）
  $data = ["apple", "banana", "cherry"];
  $ref = "data"; // $ref的值为数组变量名

  // 1. ${$ref[0]}：先取$ref的第0个字符（"d"），再引用名为$d的变量
  $d = "通过字符'd'访问的变量";
  var_dump(${$ref[0]}); // 输出：string(21) "通过字符'd'访问的变量"

  // 2. $$ref[0]：PHP解析优先级为「先解析$$ref，再取索引0」，等价于${$ref}[0]
  var_dump($$ref[0]); // 输出：string(5) "apple"（等价于$data[0]）

  // 3. ${$ref}[0]：显式指定先解析$ref得到数组名，再访问数组索引0
  var_dump(${$ref}[0]); // 输出：string(5) "apple"（等价于$data[0]）


  $a = "kkt";
  $kkt = '你好，kkt';
  var_dump($$a); // 输出：string(9) "你好，kkt"
  ?>
  ```
- 实际开发注意事项：

  1. **安全风险**：若将用户输入直接作为可变变量的基础（如 `$$_GET['param']`），恶意用户可能构造输入覆盖系统关键变量（如 `$GLOBALS`、`$_SESSION`），甚至注入恶意代码，导致安全漏洞。
  2. **代码可读性差**：动态生成的变量名会使代码逻辑晦涩，增加维护难度，其他开发者难以快速追踪变量的引用关系。
  3. **调试困难**：可变变量的名称在运行时动态生成，调试时无法通过固定变量名直接定位，增加问题排查成本。
  4. **替代方案**：多数场景下，使用数组（如 `$arr['dynamic_key']`）或对象属性（如 `$obj->$dynamicProp`）可更安全、清晰地实现动态数据访问，应优先选择。
  5. **性能影响**：
     可变变量的解析需要额外的内存和计算资源，大规模使用可能导致性能下降。

</details>

### 4. PHP8 在类型声明和代码声明方面有哪些增强特性（如联合类型、构造函数属性提升等）？这些特性的作用是什么？declare (strict_types=1) 在 PHP8 中的作用有何变化？

<details>
<summary>答案与思考</summary>

- 思路要点：

PHP8在类型声明和代码声明机制上进行了突破性增强，进一步强化了类型安全，简化了代码编写，并提升了开发灵活性，核心特性包括：

1. **联合类型（Union Types）**：允许参数、返回值或属性接受多种显式声明的类型（如 `int|string|null`），替代了PHP7中 `?type`的有限 nullable 支持，能更精准地描述多类型场景。
2. **构造函数属性提升（Constructor Property Promotion）**：将类属性的声明、访问控制（public/protected/private）与构造函数参数合并，大幅减少冗余代码，直接在构造函数参数中完成属性定义。
3. **命名参数（Named Arguments）**：调用函数时可通过 `参数名: 值`的形式指定参数，无需严格遵循参数顺序，尤其适合参数多、有默认值的函数，提升代码可读性和可维护性。
4. **nullsafe运算符（Nullsafe Operator）**：通过 `?->`简化对可能为 `null`的对象属性/方法的链式访问，自动在中间值为 `null`时返回 `null`，替代冗长的 `isset()`或 `if`判断。
5. **严格的类型检查强化**：对内部函数强制类型声明，严格模式（`strict_types=1`）下字符串含非数字字符（如 `"123a"`）无法转为数字，避免隐式转换导致的逻辑错误。
6. **返回类型协变与参数类型逆变**：支持子类方法的返回类型比父类更具体（协变），参数类型比父类更宽泛（逆变），增强面向对象的类型灵活性与兼容性。
7. **纯交集类型（PHP8.1+）**：通过 `&`声明多个接口的交集（如 `A&B`），要求参数/返回值同时实现多个接口，进一步细化类型约束。

- 实际开发注意事项：

1. 联合类型中 `null`需显式包含（如 `int|null`），PHP7的 `?int`仍兼容但建议统一为 `int|null`以保持一致性。
2. 构造函数属性提升必须指定访问控制符（public/protected/private），否则会被视为普通参数而非类属性，且无法直接通过 `$this`访问。
3. 命名参数可与位置参数混合使用，但命名参数必须放在位置参数之后，且不可重复指定同一参数（如 `func(1, a: 2)`合法，`func(a: 1, 2)`不合法）。
4. `?->`仅用于对象的属性/方法访问，不能用于数组（如 `$arr?[0]`不合法）或静态方法，且中间值为 `null`时不会抛出错误，需注意业务逻辑对 `null`的处理。
5. 升级到PHP8时，需处理严格模式下的类型转换差异：非数字字符串（如 `"abc"`）无法转为 `int/float`，可能导致依赖隐式转换的旧代码报错。
6. 协变与逆变仅适用于对象类型（类/接口），标量类型（`int`/`string`等）不支持，且子类方法的类型必须与父类兼容（如父类返回 `Animal`，子类可返回 `Dog`；父类参数为 `Dog`，子类可接受 `Animal`）。
7. 纯交集类型（`A&B`）只能用于接口组合，不能包含类或标量类型，且需确保类型组合的合理性（如 `Countable&Iterator`）。

- 示例代码：

```php
<?php
declare(strict_types=1);

// 1. 联合类型（Union Types）
function formatValue(int|string|bool|null $value): string {
    if ($value === null) return "null";
    return "类型: " . gettype($value) . ", 值: {$value}";
}
echo formatValue(100) . "<br>";    // 正常（int）
echo formatValue("hello") . "<br>"; // 正常（string）
echo formatValue(null) . "<br>";    // 正常（null）
// echo formatValue(3.14); // 报错：TypeError（float不在声明范围内）


// 2. 构造函数属性提升
class Product {
    // 直接在构造函数参数中声明属性（类型+访问控制符）
    public function __construct(
        private int $id,
        public string $name,
        protected float $price,
        private ?string $desc = null
    ) {}

    public function getPrice(): string {
        return "¥" . number_format($this->price, 2);
    }
}
$product = new Product(1, "PHP8指南", 59.9);
echo $product->name . " 价格: " . $product->getPrice(); // 输出：PHP8指南 价格: ¥59.90


// 3. 命名参数
function sendMessage(
    string $to,
    string $content,
    string $subject = "通知",
    bool $isHtml = false
) {
    return [
        'to' => $to,
        'subject' => $subject,
        'content' => $content,
        'html' => $isHtml
    ];
}
// 不按参数顺序，通过名称指定
$msg = sendMessage(
    content: "PHP8类型声明新特性",
    to: "user@example.com",
    isHtml: true
);
var_dump($msg['subject']); // 输出：string(2) "通知"（使用默认值）


// 4. nullsafe运算符
class Profile {
    public ?User $user = null;
}
class User {
    public ?Address $address = null;
}
class Address {
    public function getZipCode(): string {
        return "100000";
    }
}

$profile = new Profile();
// 无需嵌套isset()，中间为null时直接返回null
$zip = $profile->user?->address?->getZipCode();
var_dump($zip); // 输出：NULL

$profile->user = new User();
$profile->user->address = new Address();
$zip = $profile->user?->address?->getZipCode();
var_dump($zip); // 输出：string(6) "100000"


// 5. 协变与逆变
interface Shape {}
class Circle implements Shape {}
class Square implements Shape {}

class ShapeDrawer {
    public function draw(Shape $shape): Shape {
        return new Circle();
    }
}

class CircleDrawer extends ShapeDrawer {
    // 参数类型逆变：父类接受Shape，子类可接受更宽泛的object（实际开发通常更具体）
    public function draw(object $shape): Circle { // 返回类型协变：比父类更具体（Circle实现Shape）
        return new Circle();
    }
}
$drawer = new CircleDrawer();
var_dump($drawer->draw(new Square())); // 输出：object(Circle)


// 6. 纯交集类型（PHP8.1+）
interface Logger {}
interface Formatter {}
class FileLogger implements Logger, Formatter {}

function processLogger(Logger&Formatter $logger): void {
    echo "处理同时实现Logger和Formatter的对象";
}
processLogger(new FileLogger()); // 正常（FileLogger同时实现两个接口）
// processLogger(new class implements Logger {}); // 报错：未实现Formatter接口
?>
```

</details>

### 5. 解释 PHP 的垃圾回收机制（GC）的基本原理，什么情况下会触发垃圾回收？如何手动触发垃圾回收？

<details>
<summary>答案与思考</summary>

- 思路要点：

PHP 8.1+ 的垃圾回收机制（GC）核心基于「引用计数」+「循环引用收集器」，在保持低开销的同时解决了循环引用导致的内存泄漏问题，其核心原理与触发逻辑如下：

1. **基本原理**：

   - **引用计数机制**：PHP 中每个变量（Zval 结构）都包含 `refcount`（引用计数）和 `is_ref`（是否为显式引用）。当变量被赋值、传递给函数参数或作为数组/对象属性时，`refcount` 加 1；当变量被 `unset()`、超出作用域或覆盖时，`refcount` 减 1。当 `refcount` 降至 0 时，PHP 会立即释放该变量占用的内存。
   - **循环引用处理**：若两个/多个对象/数组互相引用（如 A 引用 B，B 引用 A），它们的 `refcount` 始终为 1，无法通过引用计数自动释放。PHP 8.1+ 保留了「根缓冲区 + 标记-清除」机制：将可能产生循环引用的「根节点」（如数组/对象）存入根缓冲区，当缓冲区满（默认 1000 个根节点）或满足触发条件时，GC 会标记出循环引用的节点，清除无效引用并释放内存。
   - **PHP 8.1+ 优化**：对 Zval 结构进行紧凑存储优化，减少内存占用；提升循环引用收集器的扫描效率，降低 GC 运行时的性能损耗；同时增强对 `WeakMap`、`WeakReference` 等弱引用类型的支持，帮助开发者主动避免循环引用。
2. **触发机制**：

   - **自动触发**：① 根缓冲区存储的「根节点」数量达到阈值（默认 1000，可通过 `gc_mem_caches_count()` 查看）；② 脚本执行中内存占用达到 `php.ini` 中 `gc_memory_cutoff` 配置的阈值（默认无限制，需手动配置）；③ 脚本结束时，PHP 会自动触发一次完整 GC，释放所有未回收内存。
   - **手动触发**：通过调用 `gc_collect_cycles()` 函数主动触发 GC，函数返回本次收集到的循环引用内存块数量（PHP 8.1+ 中返回值更精准，包含释放的内存字节数相关统计）。
3. **PHP 8.1+ 特性适配**：

   - 对匿名类、`readonly` 属性的对象，GC 能正确识别其引用关系，避免因属性不可修改导致的引用计数误判；
   - 支持 `WeakMap`（弱引用映射）：`WeakMap` 的键为对象的弱引用，不会增加对象的 `refcount`，当键对象被回收时，对应的键值对会自动从 `WeakMap` 中移除，从根源上避免循环引用。

- 实际开发注意事项：

1. **无需滥用手动 GC**：PHP 自动 GC 已足够高效，仅在内存敏感场景（如长时间运行的 CLI 脚本、批量处理大量对象）中，可在批量操作后手动触发 `gc_collect_cycles()`，避免内存堆积；常规 Web 脚本因请求生命周期短，自动 GC 即可满足需求。
2. **警惕循环引用场景**：常见于「双向关联对象」（如 `User` 类引用 `Order` 类，`Order` 类同时引用 `User` 类）、「对象数组自引用」（数组中元素引用数组本身），此类场景需通过 `WeakMap` 替代强引用（如 `class Order { private WeakMap $user; }`），减少 GC 负担。
3. **关注 `WeakMap`/`WeakReference` 用法**：PHP 8.1+ 中 `WeakMap` 支持更完善，适用于缓存、关联映射等场景，避免因强引用导致的内存泄漏；但需注意 `WeakMap` 的键必须是对象，且不可迭代（需通过 `foreach` 遍历前判断键是否存在）。
4. **调试 GC 状态**：可通过 `gc_status()` 函数（PHP 8.1+ 返回更详细的数组，包含 `runs`（GC 运行次数）、`collected`（回收的循环引用数）、`buffer_size`（当前根缓冲区大小）等）查看 GC 运行状态，定位内存泄漏问题。
5. **避免 `unset()` 误解**：`unset()` 仅减少变量的 `refcount`，并非直接释放内存；若存在其他引用（如数组/对象中仍引用该变量），`refcount` 未降至 0 时，内存不会释放。

- 示例代码（PHP 8.1+ 适配）：

```php
<?php
declare(strict_types=1);

// 1. 引用计数机制演示
function refcountDemo(): void {
    $a = "PHP 8.1 GC"; // $a 的 refcount = 1
    $b = $a; // 赋值：$a 和 $b 指向同一 Zval，refcount = 2
    $c = &$a; // 显式引用：is_ref = 1，refcount = 2（显式引用不增加 refcount，仅标记关联）

    // 查看引用计数（需安装 xdebug 扩展，生产环境可通过 gc_status() 间接判断）
    var_dump(xdebug_debug_zval('a')); // 输出：a: (refcount=2, is_ref=1)='PHP 8.1 GC'

    unset($b); // 销毁 $b：refcount = 1（仅 $a 和 $c 关联）
    unset($c); // 销毁显式引用：is_ref = 0，refcount = 1（仅 $a 引用）
    unset($a); // 销毁 $a：refcount = 0，内存自动释放
    echo "引用计数演示完成<br>";
}
refcountDemo();


// 2. 循环引用与 GC 自动处理
class CycleA {
    public ?CycleB $b = null;
}
class CycleB {
    public ?CycleA $a = null;
}

function cycleRefDemo(): void {
    // 创建循环引用：A 引用 B，B 引用 A
    $a = new CycleA();
    $b = new CycleB();
    $a->b = $b;
    $b->a = $a;

    // 销毁外部引用：$a 和 $b 被 unset，但 A 和 B 互相引用，refcount 仍为 1
    unset($a, $b);

    // 查看 GC 状态（此时根缓冲区已存入循环引用的根节点）
    $gcStatus = gc_status();
    echo "循环引用前根缓冲区大小：{$gcStatus['buffer_size']}<br>"; // 输出：1（或更多，取决于其他操作）

    // 自动触发 GC（若根缓冲区未满，可手动触发）
    $collected = gc_collect_cycles(); // 手动触发 GC，返回回收的循环引用数
    echo "GC 回收的循环引用数：{$collected}<br>"; // 输出：1（回收 A 和 B 组成的循环引用）

    $gcStatusAfter = gc_status();
    echo "GC 后根缓冲区大小：{$gcStatusAfter['buffer_size']}<br>"; // 输出：0（缓冲区已清空）
}
cycleRefDemo();


// 3. 手动触发 GC 演示
function manualGCDemo(): void {
    // 批量创建对象（模拟内存占用场景）
    $objects = [];
    for ($i = 0; $i < 1500; $i++) {
        $obj = new stdClass();
        $obj->data = str_repeat('x', 1024); // 每个对象占用约 1KB 内存
        $objects[] = $obj;
    }

    // 销毁外部引用，但可能存在隐性循环引用（如数组内部关联）
    unset($objects);

    // 手动触发 GC 并查看效果
    echo "手动 GC 前内存使用：" . memory_get_usage(true) . " 字节<br>";
    $collected = gc_collect_cycles();
    echo "手动 GC 回收的循环引用数：{$collected}<br>";
    echo "手动 GC 后内存使用：" . memory_get_usage(true) . " 字节<br>"; // 内存明显下降
}
manualGCDemo();


// 4. WeakMap 解决循环引用（PHP 8.1+ 推荐用法）
class User {
    public string $name;
    public function __construct(string $name) {
        $this->name = $name;
    }
}
class Order {
    // 用 WeakMap 存储 User 弱引用，避免循环引用
    private WeakMap $userMap;

    public function __construct() {
        $this->userMap = new WeakMap();
    }

    public function bindUser(User $user, string $orderNo): void {
        $this->userMap[$user] = $orderNo; // 键为 User 弱引用，不增加 User 的 refcount
    }

    public function getOrderNo(User $user): ?string {
        return $this->userMap[$user] ?? null;
    }
}

function weakMapDemo(): void {
    $user = new User("张三");
    $order = new Order();
    $order->bindUser($user, "ORDER123");

    echo "绑定的订单号：" . $order->getOrderNo($user) . "<br>"; // 输出：ORDER123

    // 销毁 User 外部引用：User 的 refcount 降至 0，内存被释放
    unset($user);

    // WeakMap 中对应的键值对自动移除（因 User 已被回收）
    echo "User 回收后订单号：" . $order->getOrderNo(new User("张三")) . "<br>"; // 输出：null（新 User 是不同对象）
    echo "WeakMap 解决循环引用演示完成<br>";
}
weakMapDemo();
?>
```

</details>

### 6. PHP 中的闭包（Closure）是什么？闭包有哪些实际使用场景？use 关键字在闭包中起到什么作用？

<details>
<summary>思考与答案（点击展开）</summary>

- 思路要点：

  1. **闭包（Closure）的定义**：闭包是PHP中的匿名函数，它可以像变量一样被传递、赋值或作为参数使用，且能访问其创建时所在父作用域中的变量（即使父作用域已销毁）。闭包本质是 `Closure`类的实例，支持面向对象的方法（如 `bindTo()`）。
  2. **闭包的实际使用场景**：

     - **回调函数**：在 `array_map()`、`usort()`等函数中作为匿名回调，简化临时逻辑（如数组过滤、排序）。
     - **延迟执行**：封装一段代码，在需要时再执行（如事件触发、条件判断后执行）。
     - **状态保持**：通过 `use`关键字捕获父作用域变量，实现类似“函数工厂”的功能（动态生成带特定状态的函数）。
     - **框架中的路由/中间件**：如Laravel的闭包路由（`Route::get('/', function () { ... })`），直接在路由中定义处理逻辑。
     - **依赖注入**：在依赖容器中用闭包延迟实例化对象（如 `$container->bind('db', function () { return new Database(); })`）。
  3. **use关键字的作用**：
     `use`用于将闭包外部（父作用域）的变量“引入”闭包内部，使闭包能访问这些变量。默认情况下，`use`引入的是变量的副本；若需修改父作用域的变量，需通过 `&`声明为引用传递（`use (&$var)`）。`use`仅能捕获父作用域中已存在的变量，且变量作用域在闭包创建时确定（而非执行时）。
- 实际开发注意事项：

  1. **变量作用域陷阱**：`use`引入的变量是闭包创建时的快照（值传递），若需实时获取父作用域变量的最新值，需使用引用（`&`），但需谨慎处理引用导致的副作用（如意外修改父变量）。
  2. **性能考量**：闭包比普通函数稍占内存，大规模使用（如循环中创建闭包）可能影响性能，需合理控制使用场景。
  3. **短闭包语法**：PHP 7.4+支持 `fn()`短闭包（`fn($x) => $x*2`），语法更简洁，但仅支持单表达式，且 `use`关键字不可用（自动继承父作用域的变量）。
  4. **对象绑定**：通过 `Closure::bindTo()`或 `bind()`方法，可将闭包绑定到特定对象或类，使其能访问对象的私有/保护成员（类似类方法）。
  5. **避免过度使用**：闭包逻辑复杂时会降低代码可读性，建议复杂逻辑抽离为命名函数或方法。
- 示例代码（PHP 8.1+）：

```php
<?php
// 1. 闭包基础用法（匿名函数特性）
$greet = function (string $name): string {
    return "Hello, {$name}!";
};
echo $greet("PHP"); // 输出：Hello, PHP!
var_dump($greet instanceof \Closure); // 输出：bool(true)（闭包是Closure类实例）


// 2. use关键字的作用（引入外部变量）
$prefix = "Result: ";
$calculate = function (int $a, int $b) use ($prefix): string {
    $sum = $a + $b;
    return $prefix . $sum; // 使用use引入的$prefix
};
echo $calculate(3, 5); // 输出：Result: 8

// use引用传递（修改外部变量）
$count = 0;
$increment = function () use (&$count): void {
    $count++; // 修改的是外部$count的引用
};
$increment();
$increment();
echo $count; // 输出：2（外部变量被修改）


// 3. 闭包作为回调函数（场景：数组处理）
$numbers = [1, 2, 3, 4, 5];
// 用闭包过滤偶数
$evenNumbers = array_filter($numbers, function (int $n): bool {
    return $n % 2 === 0;
});
var_dump($evenNumbers); // 输出：array(2) { [1]=> int(2) [3]=> int(4) }

// 用闭包排序（按字符串长度）
$words = ["apple", "banana", "cherry", "date"];
usort($words, function (string $a, string $b): int {
    return strlen($a) - strlen($b);
});
var_dump($words); // 输出：array(4) { [0]=> string(4) "date" [1]=> string(5) "apple" ... }


// 4. 闭包实现状态保持（函数工厂）
function createMultiplier(int $factor): \Closure {
    // 闭包捕获$factor，每次调用保持该状态
    return function (int $num) use ($factor): int {
        return $num * $factor;
    };
}

$double = createMultiplier(2); // 生成"乘以2"的闭包
$triple = createMultiplier(3); // 生成"乘以3"的闭包
echo $double(5); // 输出：10
echo $triple(5); // 输出：15


// 5. PHP 7.4+ 短闭包（fn()）
$square = fn(int $x): int => $x * $x;
echo $square(4); // 输出：16

// 短闭包自动继承父作用域变量（无需use）
$base = 10;
$addBase = fn(int $x): int => $x + $base;
echo $addBase(5); // 输出：15


// 6. 闭包绑定到对象（访问私有成员）
class User {
    private string $name = "Alice";
}

$user = new User();
// 创建闭包并绑定到$user对象，使其能访问私有成员
$getPrivateName = function () {
    return $this->name;
};
$boundClosure = $getPrivateName->bindTo($user, User::class);
echo $boundClosure(); // 输出：Alice
?>
```

</details>


### 7. PHP 中值传递和引用传递的区别是什么？处理数据时选择哪种传递方式更合适？为什么？

<details>
<summary>思考与答案（点击展开）</summary>

- 思路要点：

  1. **核心区别**：

     - **值传递**：函数接收变量的副本，修改参数不会影响原变量（基础类型如 `int`、`string`默认值传递）。
     - **引用传递**：函数接收变量的内存地址，修改参数会直接影响原变量（通过 `&`符号声明，如 `function func(&$var) { ... }`）。
  2. **选择依据**：

     - 优先使用**值传递**：适用于基础类型（小数据）、无需修改原变量的场景，避免意外副作用，代码更安全。
     - 谨慎使用**引用传递**：适用于大型数据（如大数组、对象）以减少内存复制开销，或需在函数内修改原变量的场景（如交换变量值、批量修改数据）。
- 实际开发注意事项：

  1. 对象默认是“引用传递的效果”：对象变量存储的是指针，传递时复制指针而非对象本身，修改对象属性会影响原对象（但重新赋值变量不会影响原变量）。
  2. 引用传递可能导致不可预期的副作用：多人协作时，修改引用参数可能意外改变外部变量，增加调试难度。
  3. 对大型数组使用引用传递可优化性能：值传递会复制整个数组，内存开销大；引用传递仅传递地址，适合处理大数据集。
  4. PHP 中函数参数的引用传递必须显式声明（`&`），否则即使变量本身是引用，传递时仍为值传递。
- 示例代码（PHP 8.1+）：

```php
<?php
// 1. 值传递示例（基础类型）
$num = 10;
function addValue($n) {
    $n += 5;
    echo "函数内：{$n}<br>"; // 输出：15
}
addValue($num);
echo "函数外：{$num}<br>"; // 输出：10（原变量未变）


// 2. 引用传递示例（基础类型）
$num2 = 10;
function addRef(&$n) { // 显式声明引用传递
    $n += 5;
    echo "函数内：{$n}<br>"; // 输出：15
}
addRef($num2);
echo "函数外：{$num2}<br>"; // 输出：15（原变量被修改）


// 3. 数组的值传递 vs 引用传递
$largeArray = range(1, 100000); // 大型数组

// 值传递：复制数组，内存开销大
function modifyArrayByValue($arr) {
    $arr[0] = 999;
}
modifyArrayByValue($largeArray);
echo "值传递后第一个元素：{$largeArray[0]}<br>"; // 输出：1（原数组未变）

// 引用传递：不复制数组，直接修改原数组
function modifyArrayByRef(&$arr) {
    $arr[0] = 999;
}
modifyArrayByRef($largeArray);
echo "引用传递后第一个元素：{$largeArray[0]}<br>"; // 输出：999（原数组被修改）


// 4. 对象的传递特性（类似引用传递，但本质是指针复制）
class User {
    public $name = "Alice";
}
$user = new User();

function changeName($obj) { // 未声明&，但传递的是对象指针
    $obj->name = "Bob"; // 修改属性会影响原对象
    $obj = new User(); // 重新赋值变量，不影响原对象
    $obj->name = "Charlie";
}
changeName($user);
echo $user->name; // 输出：Bob（原对象属性被修改）
?>
```

</details>

### 8. PHP 中抽象类（abstract class）和接口（interface）有什么核心区别？如何选择使用抽象类或接口定义通用行为？

<details>
<summary>思考与答案（点击展开）</summary>

- 思路要点：

  1. **核心区别**：  

     | 特性       | 抽象类（abstract class）                                          | 接口（interface）                                                                               |
     | ---------- | ----------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
     | 定义关键字 | `abstract class`                                                | `interface`                                                                                   |
     | 方法实现   | 可包含抽象方法（无实现）和具体方法                                | PHP 7.4前：仅抽象方法；PHP 8.0+：可包含私有方法、常量和默认实现的方法（`public`/`private`） |
     | 继承方式   | 类通过 `extends`单继承                                          | 类通过 `implements`多实现                                                                     |
     | 属性声明   | 可声明任意访问控制的属性                                          | 仅可声明常量（`const`），不可声明变量                                                         |
     | 访问控制   | 方法可声明为 `public`/`protected`（不能 `private`抽象方法） | 方法默认 `public`（PHP 8.0+可加 `private`）                                                 |
     | 设计目的   | 代码复用（提供基础实现）                                          | 行为规范（定义必须实现的方法）                                                                  |
  2. **选择依据**：

     - 用**抽象类**：当需要为子类提供通用实现（代码复用），且子类属于同一“家族”（如 `Animal`→`Dog`/`Cat`），逻辑上是“is-a”关系。
     - 用**接口**：当需要规范不同类的共同行为（不关心实现），且类可能属于不同家族（如 `Flyable`接口可被 `Bird`和 `Plane`实现），逻辑上是“can-do”关系。
- 实际开发注意事项：

  1. 抽象类不能实例化，接口也不能实例化，均需通过子类/实现类使用。
  2. PHP 8.0+ 接口支持 `private`方法（仅接口内部调用）和默认实现（`public function method() { ... }`），但抽象方法仍需实现类重写。
  3. 一个类可实现多个接口（多继承的替代方案），但只能继承一个抽象类（PHP单继承限制）。
  4. 接口常量必须被子类继承且不可修改，抽象类的属性可被子类继承并修改（依访问控制）。
  5. 过度使用接口可能导致类实现冗余，过度使用抽象类可能限制灵活性（单继承）。
- 示例代码（PHP 8.1+）：

```php
<?php
// 1. 抽象类示例（提供基础实现）
abstract class Animal {
    protected string $name;

    public function __construct(string $name) {
        $this->name = $name;
    }

    // 抽象方法（子类必须实现）
    abstract public function makeSound(): string;

    // 具体方法（子类可直接使用）
    public function getName(): string {
        return $this->name;
    }
}

class Dog extends Animal {
    public function makeSound(): string {
        return "汪汪";
    }
}

class Cat extends Animal {
    public function makeSound(): string {
        return "喵喵";
    }
}


// 2. 接口示例（规范行为）
interface Flyable {
    // 抽象方法（实现类必须重写）
    public function fly(): string;

    // PHP 8.0+：默认实现的方法
    public function land(): string {
        return "开始降落";
    }

    // PHP 8.0+：私有方法（仅接口内部调用）
    private function checkWind(): bool {
        return true; // 模拟检查风力
    }
}

// 类实现接口（可多实现）
class Bird extends Animal implements Flyable {
    public function makeSound(): string {
        return "叽叽";
    }

    public function fly(): string {
        return "鸟儿在飞";
    }
}

class Plane implements Flyable {
    public function fly(): string {
        return "飞机在飞";
    }

    // 可选：重写接口的默认方法
    public function land(): string {
        return "飞机开始降落";
    }
}


// 使用示例
$dog = new Dog("阿黄");
echo $dog->getName() . "叫：" . $dog->makeSound() . "<br>"; // 输出：阿黄叫：汪汪

$sparrow = new Bird("麻雀");
echo $sparrow->getName() . "叫：" . $sparrow->makeSound() . "，" . $sparrow->fly() . "<br>"; // 输出：麻雀叫：叽叽，鸟儿在飞

$boeing = new Plane();
echo $boeing->fly() . "，" . $boeing->land() . "<br>"; // 输出：飞机在飞，飞机开始降落
?>
```

</details>

### 9. PHP 的 JSON 扩展函数（json_encode、json_decode）有哪些常用关键参数？处理 JSON 数据时需要解决哪些常见问题？

<details>
<summary>思考与答案（点击展开）</summary>

- 思路要点：

  1. **常用关键参数**：

     - **json_encode**：

       - `JSON_UNESCAPED_UNICODE`：保留中文不转义（避免 `\uXXXX`格式）。
       - `JSON_PRETTY_PRINT`：格式化输出（带缩进和换行，便于阅读）。
       - `JSON_NUMERIC_CHECK`：自动将字符串类型的数字转为数值类型。
       - `JSON_FORCE_OBJECT`：强制数组转为JSON对象（即使是空数组）。
       - `JSON_THROW_ON_ERROR`：PHP 7.3+，错误时抛出异常（替代 `json_last_error()`）。
     - **json_decode**：

       - 第二个参数（`$assoc`）：`true`时返回关联数组，`false`（默认）返回对象。
       - 第三个参数（`$depth`）：限制嵌套深度（默认512，超过抛出错误）。
       - 第四个参数（`$flags`）：`JSON_BIGINT_AS_STRING`：将大整数转为字符串（避免精度丢失）。
  2. **常见问题及解决方案**：

     - **中文乱码**：用 `JSON_UNESCAPED_UNICODE`参数保留中文。
     - **循环引用**：对象互相引用时 `json_encode`失败，需先解除引用或用 `JSON_PARTIAL_OUTPUT_ON_ERROR`。
     - **大整数精度丢失**：`json_decode`时用 `JSON_BIGINT_AS_STRING`转为字符串。
     - **错误处理**：结合 `json_last_error()`或 `JSON_THROW_ON_ERROR`捕获编码/解码错误（如语法错误、深度超限）。
     - **数据类型转换**：`json_encode`会将PHP的 `null`转为JSON `null`，资源类型（如 `fopen`句柄）无法编码。
- 实际开发注意事项：

  1. 编码前确保数据可序列化：资源类型（如 `$file = fopen(...)`）无法被 `json_encode`处理，需过滤或转换。
  2. 解码时明确数据类型：根据需求选择返回对象（默认）或关联数组（`$assoc=true`），避免类型混淆。
  3. 生产环境建议开启 `JSON_THROW_ON_ERROR`：通过异常捕获错误，比 `json_last_error()`更直观，便于调试。
  4. 处理第三方JSON数据时，先验证格式合法性：可结合 `json_last_error()`检查是否为有效JSON。
  5. 避免过度依赖 `JSON_NUMERIC_CHECK`：可能误将字符串“0123”转为数字123（丢失前导零），需根据业务场景判断。
- 示例代码（PHP 8.1+）：

```php
<?php
// 1. json_encode 常用参数
$data = [
    'name' => '张三',
    'age' => '25', // 字符串类型的数字
    'hobbies' => ['篮球', '编程'],
    'is_student' => false,
    'score' => null
];

// 保留中文+格式化输出
$json1 = json_encode($data, JSON_UNESCAPED_UNICODE | JSON_PRETTY_PRINT);
echo "JSON1：<pre>{$json1}</pre>";

// 自动将字符串数字转为数值类型
$json2 = json_encode($data, JSON_UNESCAPED_UNICODE | JSON_NUMERIC_CHECK);
echo "JSON2：{$json2}<br>"; // age变为数值25


// 2. json_decode 常用参数
$jsonStr = '{"id": 1234567890123456789, "name": "PHP", "tags": ["lang", "web"]}';

// 返回对象（默认）
$obj = json_decode($jsonStr);
echo "对象访问：{$obj->name}<br>"; // 输出：PHP

// 返回关联数组
$arr = json_decode($jsonStr, true); // $assoc=true
echo "数组访问：{$arr['tags'][0]}<br>"; // 输出：lang

// 处理大整数（避免精度丢失）
$bigIntJson = '{"big_num": 9223372036854775807}';
$objBig = json_decode($bigIntJson);
echo "大整数默认处理：{$objBig->big_num}<br>"; // 可能丢失精度

$arrBig = json_decode($bigIntJson, true, 512, JSON_BIGINT_AS_STRING);
echo "大整数转为字符串：{$arrBig['big_num']}<br>"; // 正确保留完整数值


// 3. 错误处理（JSON_THROW_ON_ERROR）
try {
    // 无效JSON字符串
    json_decode('{invalid}', flags: JSON_THROW_ON_ERROR);
} catch (JsonException $e) {
    echo "解码错误：{$e->getMessage()}<br>"; // 输出：Syntax error
}

try {
    // 循环引用的对象
    $a = new stdClass();
    $b = new stdClass();
    $a->b = $b;
    $b->a = $a;
    json_encode($a, flags: JSON_THROW_ON_ERROR);
} catch (JsonException $e) {
    echo "编码错误：{$e->getMessage()}<br>"; // 输出：Recursion detected
}
?>
```

</details>

### 10. PHP 中的静态变量（static）有什么独特特性？使用静态变量需要注意哪些潜在问题？

<details>
<summary>思考与答案（点击展开）</summary>

- 思路要点：

  1. **静态变量的独特特性**：

     - **函数内静态变量**：在函数多次调用间保留值（仅初始化一次），生命周期与脚本一致（非函数执行期）。
     - **类中静态成员**：属于类本身而非实例，所有实例共享同一静态属性；通过 `self::`或类名访问，无需实例化即可调用。
     - **访问限制**：静态方法不能直接访问非静态属性（需通过实例），非静态方法可访问静态属性。
     - **继承行为**：子类可继承父类静态成员，通过 `parent::`访问；重写静态方法时需保持兼容的访问控制。
  2. **潜在问题与注意事项**：

     - **状态污染**：静态变量在脚本生命周期内持续存在，多次调用可能累积状态（如计数器未重置），导致逻辑错误。
     - **测试困难**：依赖静态变量的代码难以单元测试（静态状态全局共享，无法隔离测试环境）。
     - **线程安全风险**：多线程环境（如PHP-FPM多进程模型下的共享内存）中，静态变量可能引发竞争条件（数据不一致）。
     - **耦合性高**：过度使用静态方法会导致代码耦合（硬依赖类名），不利于依赖注入和扩展。
     - **自动加载问题**：访问未加载类的静态成员可能导致“类未找到”错误，需确保类已加载。
- 实际开发注意事项：

  1. 函数内静态变量适合存储“仅初始化一次”的资源（如数据库连接配置），但需避免存储可变状态（如用户会话数据）。
  2. 类中静态属性适合存储全局共享配置（如 `App::version()`），但不适合存储实例相关数据（应使用非静态属性）。
  3. 优先使用依赖注入（DI）替代静态方法调用：如用 `$logger->log()`而非 `Logger::log()`，降低耦合性。
  4. 静态变量在闭包中需显式通过 `use`引入（函数内静态变量与闭包是不同作用域）。
  5. 重置静态变量状态可通过类方法实现（如 `public static function reset() { self::$count = 0; }`），便于测试。
- 示例代码（PHP 8.1+）：

```php
<?php
// 1. 函数内静态变量（保留状态）
function counter(): int {
    static $count = 0; // 仅初始化一次
    $count++;
    return $count;
}

echo counter() . "<br>"; // 输出：1
echo counter() . "<br>"; // 输出：2（保留上次值）
echo counter() . "<br>"; // 输出：3


// 2. 类中的静态成员
class MathUtil {
    public static int $pi = 3; // 静态属性（共享值）
    private static int $calcCount = 0; // 静态计数器

    // 静态方法
    public static function add(int $a, int $b): int {
        self::$calcCount++; // 访问静态属性
        return $a + $b;
    }

    public static function getCalcCount(): int {
        return self::$calcCount;
    }

    // 重置静态状态（便于测试）
    public static function reset(): void {
        self::$calcCount = 0;
    }
}

// 访问静态属性和方法（无需实例化）
echo "π = " . MathUtil::$pi . "<br>"; // 输出：3
echo "1+2 = " . MathUtil::add(1, 2) . "<br>"; // 输出：3
echo "计算次数：" . MathUtil::getCalcCount() . "<br>"; // 输出：1

// 所有实例共享静态属性
$obj1 = new MathUtil();
$obj2 = new MathUtil();
$obj1::add(3, 4); // 等价于MathUtil::add()
echo "计算次数：" . $obj2::getCalcCount() . "<br>"; // 输出：2

MathUtil::reset();
echo "重置后次数：" . MathUtil::getCalcCount() . "<br>"; // 输出：0


// 3. 静态变量的潜在问题（状态污染）
function getUserRole(): string {
    static $role = 'guest';
    return $role;
}

echo "初始角色：" . getUserRole() . "<br>"; // 输出：guest

// 意外修改静态变量（可能在代码其他位置）
function setAdminRole(): void {
    static $role; // 与getUserRole()的$role是不同的静态变量（函数作用域隔离）
    $role = 'admin'; // 此修改不影响getUserRole()
}
setAdminRole();
echo "修改后角色：" . getUserRole() . "<br>"; // 仍输出：guest（函数内静态变量作用域隔离）
?>
```

</details>

### 11. PHP 中的错误（Error）和异常（Exception）有什么本质区别？如何区分并统一收集这两类问题？

<details>
<summary>思考与答案（点击展开）</summary>

- 思路要点：

  1. **本质区别**：

     - **错误（Error）**：PHP 自身运行时的问题，通常由非法操作或环境限制导致（如语法错误、内存溢出、调用未定义函数），属于“不可预期的系统级问题”。PHP 7 前错误会直接终止脚本，PHP 7+ 多数错误被封装为 `Error`类（继承自 `Throwable`），可被捕获。
     - **异常（Exception）**：程序逻辑中的预期问题（如无效用户输入、数据库连接失败），由开发者主动通过 `throw`抛出，属于“可预期的应用级问题”，必须被捕获否则终止脚本。
  2. **区分与统一收集**：

     - **区分方式**：

       - 错误通常是 PHP 内核触发（如 `TypeError`、`ParseError`），异常是开发者主动抛出（如 `Exception`、自定义异常）。
       - 错误侧重“系统无法处理的状态”，异常侧重“程序可处理的异常流程”。
     - **统一收集方案**：

       - **错误处理**：用 `set_error_handler()`捕获非致命错误（如 `E_WARNING`），`register_shutdown_function()`捕获致命错误（如 `E_ERROR`）。
       - **异常处理**：用 `set_exception_handler()`捕获未被 `try-catch`处理的异常。
       - **PHP 7+ 统一捕获**：因 `Error`和 `Exception`均实现 `Throwable`接口，可通过 `try-catch(Throwable $e)`统一捕获所有错误和异常。
- 实际开发注意事项：

  1. 语法错误（`ParseError`）无法被 `set_error_handler`捕获，因为解析阶段脚本尚未执行，需在开发阶段通过语法检查工具避免。
  2. 生产环境应禁用错误显示（`display_errors = Off`），启用错误日志（`log_errors = On`），避免敏感信息泄露。
  3. 自定义异常类可继承 `Exception`，便于按业务类型分类处理（如 `ValidationException`、`DatabaseException`）。
  4. 捕获错误/异常后，需记录详细上下文（如时间、请求URL、堆栈跟踪），便于排查问题，但避免记录敏感信息（如密码）。
  5. 非致命错误（如 `E_NOTICE`）可通过 `error_reporting()`调整报告级别，但不应忽略（可能隐藏潜在问题）。
- 示例代码（PHP 8.1+）：

```php
<?php
// 1. 错误与异常的本质区别演示
try {
    // 异常：开发者主动抛出
    throw new Exception("这是一个自定义异常");
} catch (Exception $e) {
    echo "捕获异常：{$e->getMessage()}<br>";
}

try {
    // 错误：PHP内核触发（类型错误）
    $num = "string" + 10; // PHP 8+ 会抛出TypeError
} catch (TypeError $e) {
    echo "捕获错误：{$e->getMessage()}<br>"; // 输出：Unsupported operand types: string + int
}


// 2. 统一捕获所有Throwable（错误和异常）
try {
    // 异常
    throw new RuntimeException("运行时异常");
    // 错误（注释掉上面的异常，测试错误）
    // call_user_func('undefined_function'); // 调用未定义函数，抛出Error
} catch (Throwable $e) { // 同时捕获Error和Exception
    echo "统一捕获：{$e->getMessage()}，类型：" . get_class($e) . "<br>";
}


// 3. 全局错误/异常处理注册
// 注册异常处理器
set_exception_handler(function (Throwable $e) {
    $log = "[".date('Y-m-d H:i:s')."] 异常：{$e->getMessage()}，文件：{$e->getFile()}:{$e->getLine()}\n";
    error_log($log, 3, 'error.log'); // 写入日志文件
    echo "系统异常，请联系管理员<br>"; // 向用户显示友好信息
});

// 注册错误处理器（处理非致命错误）
set_error_handler(function (int $errno, string $errstr, string $errfile, int $errline) {
    $log = "[".date('Y-m-d H:i:s')."] 错误：{$errstr}，文件：{$errfile}:{$errline}，级别：{$errno}\n";
    error_log($log, 3, 'error.log');
    echo "系统错误，请联系管理员<br>";
    return true; // 阻止PHP默认处理
});

// 注册脚本终止处理器（捕获致命错误）
register_shutdown_function(function () {
    $error = error_get_last();
    if ($error && in_array($error['type'], [E_ERROR, E_PARSE, E_CORE_ERROR])) {
        $log = "[".date('Y-m-d H:i:s')."] 致命错误：{$error['message']}，文件：{$error['file']}:{$error['line']}\n";
        error_log($log, 3, 'error.log');
        echo "系统致命错误，请联系管理员<br>";
    }
});


// 测试全局处理器
// 触发异常（会被set_exception_handler捕获）
// throw new Exception("全局异常测试");

// 触发非致命错误（会被set_error_handler捕获）
// $undefinedVar; // E_NOTICE

// 触发致命错误（会被register_shutdown_function捕获）
// call_user_func('fatal_error_function');
?>
```

</details>

### 12. PHP 命名空间（Namespace）的主要作用是什么？如何解决不同模块间的类名冲突问题？命名空间的导入（use）和别名（as）用法如何举例说明？

<details>
<summary>思考与答案（点击展开）</summary>

- 思路要点：

  1. **命名空间的主要作用**：

     - **解决命名冲突**：避免不同模块/库中同名的类、函数、常量冲突（如 `App\User`与 `Lib\User`可共存）。
     - **代码组织**：按功能或模块划分命名空间（如 `App\Controller`、`App\Model`），提高代码可读性和可维护性。
     - **访问控制**：通过命名空间层级隔离代码，明确类的归属（如 `Illuminate\Database\Model`属于Laravel的数据库模块）。
  2. **解决类名冲突的方案**：

     - **命名空间隔离**：为不同模块的类定义不同命名空间（如 `Payment\Wechat\Order`和 `Payment\Alipay\Order`）。
     - **完全限定名称**：通过完整命名空间访问类（如 `new \OtherNamespace\SameClassName()`），避免歧义。
     - **导入与别名**：用 `use`导入类并通过 `as`取别名（如 `use OtherNamespace\SameClassName as AliasName`），简化访问并避免冲突。
  3. **导入（use）和别名（as）的用法**：

     - **导入类**：`use Namespace\ClassName;` 后可直接用 `ClassName`访问，无需完整命名空间。
     - **导入命名空间**：`use Namespace\SubNamespace;` 后可通过 `SubNamespace\ClassName`访问子空间类。
     - **别名**：`use Namespace\ClassName as Alias;` 为类指定别名，解决同名冲突（如 `use A\User as AUser; use B\User as BUser;`）。
     - **PHP 7+ 批量导入**：`use Namespace\{ClassA, ClassB as B};` 一次性导入多个类。
- 实际开发注意事项：

  1. 命名空间通常与文件目录结构一致（PSR-4规范）：如 `App\Controller\UserController`对应文件 `app/Controller/UserController.php`，便于自动加载。
  2. 全局空间的类需用 `\`开头（如 `\DateTime`），否则会被解析为当前命名空间下的类。
  3. 避免过度嵌套命名空间（如 `A\B\C\D\Class`），会增加代码复杂度，建议层级控制在3层以内。
  4. 函数和常量的命名空间导入需显式声明（`use function Namespace\func; use const Namespace\CONSTANT;`），且优先级低于当前命名空间的函数/常量。
  5. 第三方库通常遵循特定命名空间规范（如Symfony的 `Symfony\Component\...`），使用时需按其规范导入。
- 示例代码（PHP 8.1+）：

```php
<?php
// 1. 定义命名空间（文件：app/Model/User.php）
namespace App\Model;

class User {
    public function getName(): string {
        return "App\Model\User";
    }
}


// 2. 另一个命名空间的同名类（文件：lib/User.php）
namespace Lib;

class User {
    public function getName(): string {
        return "Lib\User";
    }
}


// 3. 使用命名空间解决冲突（文件：index.php）
namespace App;

// 导入类并使用
use App\Model\User; // 导入App\Model\User
use Lib\User as LibUser; // 导入Lib\User并取别名LibUser

// 创建实例
$appUser = new User();
echo $appUser->getName() . "<br>"; // 输出：App\Model\User

$libUser = new LibUser();
echo $libUser->getName() . "<br>"; // 输出：Lib\User


// 4. 完全限定名称访问（不导入直接使用）
$globalUser = new \Lib\User(); // 用\开头访问全局空间下的Lib\User
echo $globalUser->getName() . "<br>"; // 输出：Lib\User


// 5. 批量导入（PHP 7+）
use App\Model\{Article, Comment}; // 批量导入App\Model下的Article和Comment
// 假设Article和Comment已定义
$article = new Article();
$comment = new Comment();


// 6. 导入函数和常量
namespace Tools;

const MAX_SIZE = 1024;

function formatTime(string $time): string {
    return date('Y-m-d', strtotime($time));
}


// 在其他命名空间中使用
namespace App;

use function Tools\formatTime; // 导入函数
use const Tools\MAX_SIZE; // 导入常量

echo "格式化时间：" . formatTime('2023-10-01') . "<br>"; // 输出：2023-10-01
echo "最大尺寸：" . MAX_SIZE . "<br>"; // 输出：1024


// 7. 命名空间与自动加载（PSR-4规范示例）
// composer.json 中配置：
/*
{
    "autoload": {
        "psr-4": {
            "App\\": "app/",
            "Lib\\": "lib/"
        }
    }
}
*/
// 执行composer dump-autoload后，可通过use自动加载类，无需require
?>
```

</details>

### 13. 什么是 PHP 生成器（Generator）？生成器相比传统数组存储数据有什么优势？如何定义简单的生成器函数实现“滚动加载”数据功能？

<details>
<summary>思考与答案（点击展开）</summary>

- 思路要点：

  1. **生成器（Generator）的定义**：生成器是一种特殊函数，通过 `yield`关键字逐个返回值（而非 `return`一次性返回），执行时返回 `Generator`对象（可迭代）。每次迭代生成一个值，函数暂停执行，下次迭代从暂停处继续，直到函数结束。
  2. **相比传统数组的优势**：

     - **内存高效**：传统数组需一次性加载所有数据到内存（如 `range(1, 1000000)`占用大量内存）；生成器按需生成值，内存占用恒定（与数据量无关）。
     - **延迟执行**：数据在迭代时动态生成，适合处理大型数据集（如数据库查询结果、日志文件），避免一次性加载导致的内存溢出。
     - **简化代码**：无需手动实现 `Iterator`接口，用 `yield`即可创建迭代器，代码更简洁。
  3. **实现“滚动加载”数据功能**：
     模拟从数据库分批查询数据（如每次查10条），通过生成器逐个返回记录，实现类似“滚动加载”的效果（每次迭代获取一批新数据）。
- 实际开发注意事项：

  1. 生成器函数不能返回值（`return`只能不带参数，用于终止生成器），否则会抛出 `RuntimeException`。
  2. 生成器是一次性的：迭代结束后无法重置，需重新调用函数创建新生成器。
  3. 适合处理“流式数据”：如读取大文件、处理API分页数据、生成无限序列（如斐波那契数列）。
  4. 性能略低于数组：因每次迭代涉及函数暂停/恢复，对小型数据集，数组可能更快；对大型数据集，生成器的内存优势远超性能差异。
  5. 可通过 `yield from`语法嵌套生成器（PHP 7+），简化多层迭代逻辑（如 `yield from subGenerator();`）。
- 示例代码（PHP 8.1+）：

```php
<?php
// 1. 基础生成器示例（对比数组）
// 生成器：内存占用低
function numberGenerator(int $start, int $end): \Generator {
    for ($i = $start; $i <= $end; $i++) {
        yield $i; // 逐个返回值
        // 每次yield后函数暂停，下次迭代从这里继续
    }
}

// 传统数组：内存占用高（大数据时）
function numberArray(int $start, int $end): array {
    $arr = [];
    for ($i = $start; $i <= $end; $i++) {
        $arr[] = $i;
    }
    return $arr;
}

// 测试内存占用（生成100万个数字）
echo "生成器内存使用：" . memory_get_usage(true) . " 字节<br>";
foreach (numberGenerator(1, 1000000) as $num) {
    // 仅处理前10个，模拟部分数据使用
    if ($num > 10) break;
}
echo "生成器处理后内存：" . memory_get_usage(true) . " 字节<br>";

echo "数组内存使用：" . memory_get_usage(true) . " 字节<br>";
$array = numberArray(1, 1000000);
foreach ($array as $num) {
    if ($num > 10) break;
}
echo "数组处理后内存：" . memory_get_usage(true) . " 字节<br>"; // 内存显著增加


// 2. 实现“滚动加载”数据功能（模拟数据库分批查询）
/**
 * 模拟数据库查询：每次查询$limit条，从$offset开始
 */
function fetchFromDb(int $offset, int $limit): array {
    // 模拟数据库数据（实际中是SQL查询：SELECT * FROM table LIMIT $offset, $limit）
    $data = [];
    for ($i = 0; $i < $limit; $i++) {
        $id = $offset + $i + 1;
        $data[] = [
            'id' => $id,
            'name' => "用户{$id}",
            'email' => "user{$id}@example.com"
        ];
        // 模拟数据量有限（共35条）
        if ($id >= 35) break;
    }
    return $data;
}

/**
 * 生成器：滚动加载数据（每次10条）
 */
function scrollLoader(int $batchSize = 10): \Generator {
    $offset = 0;
    while (true) {
        $batch = fetchFromDb($offset, $batchSize);
        if (empty($batch)) break; // 无数据时终止
      
        // 逐个返回批次中的数据（或yield from直接返回整个批次）
        yield from $batch; // yield from 简化批量yield
      
        $offset += $batchSize;
    }
}

// 使用滚动加载生成器
echo "<br>滚动加载数据：<br>";
$loader = scrollLoader(10); // 每次加载10条
foreach ($loader as $user) {
    echo "ID: {$user['id']}, 姓名: {$user['name']}<br>";
    // 模拟加载到20条时停止
    if ($user['id'] >= 20) break;
}
// 如需继续加载，可再次迭代$loader（从上次中断处继续）
?>
```

</details>

### 14. PHP 中如何定义和使用匿名类？匿名类相比普通类有什么优势和局限性？

<details>
<summary>思考与答案（点击展开）</summary>

- 思路要点：

  1. **匿名类的定义与使用**：匿名类是没有类名的类，通过 `new class`语法直接实例化，可用于创建一次性使用的简单类。支持继承父类、实现接口、声明构造函数和方法，语法与普通类类似但无需类名。
  2. **优势**：

     - **减少类名污染**：无需为仅使用一次的简单类定义名称（如临时实现接口的类），简化代码结构。
     - **增强内聚性**：类的定义与使用在同一位置，避免跳转到其他文件查看类实现，提高代码可读性。
     - **动态灵活性**：可在运行时根据条件创建不同实现的匿名类（如动态修改父类方法）。
  3. **局限性**：

     - **不可复用**：匿名类没有类名，无法二次实例化或在其他地方引用，仅适合一次性使用。
     - **调试困难**：堆栈跟踪中匿名类显示为 `class@anonymous`，难以定位具体类定义。
     - **功能受限**：不能定义静态方法/属性（PHP 7.1+ 支持）、不能被继承、无法使用 `self`引用自身（需用 `parent`或 `static`）。
     - **文档生成困难**：自动文档工具（如PHPDoc）难以识别匿名类，降低代码可维护性。
- 实际开发注意事项：

  1. 匿名类适合简单场景：如临时实现接口（如 `Iterator`）、回调类、测试中的mock对象，避免创建冗余的命名类。
  2. 复杂逻辑慎用：当类需要多个方法、属性或可能被复用，应使用普通命名类。
  3. 匿名类可访问父作用域变量：通过构造函数传递外部变量（类似闭包的 `use`，但需显式传递）。
  4. PHP 7.1+ 支持匿名类的静态成员：但因无法复用，实际意义有限。
  5. 结合接口使用更规范：匿名类实现接口可确保方法签名正确（如 `new class implements Logger { ... }`）。
- 示例代码（PHP 8.1+）：

```php
<?php
// 1. 基础匿名类示例
$greet = new class {
    public function sayHello(): string {
        return "Hello from anonymous class";
    }
};
echo $greet->sayHello() . "<br>"; // 输出：Hello from anonymous class


// 2. 匿名类实现接口
interface Logger {
    public function log(string $message): void;
}

// 匿名类实现Logger接口（一次性日志处理器）
$fileLogger = new class('/tmp/log.txt') implements Logger {
    private string $logFile;

    public function __construct(string $file) {
        $this->logFile = $file;
    }

    public function log(string $message): void {
        $log = "[" . date('Y-m-d H:i:s') . "] {$message}\n";
        file_put_contents($this->logFile, $log, FILE_APPEND);
    }
};

$fileLogger->log("系统启动"); // 写入日志到/tmp/log.txt
echo "日志已写入<br>";


// 3. 匿名类继承父类
class ParentClass {
    protected string $name;

    public function __construct(string $name) {
        $this->name = $name;
    }

    public function getName(): string {
        return $this->name;
    }
}

// 匿名类继承父类并重写方法
$child = new class("匿名子类") extends ParentClass {
    public function getName(): string {
        return "子类名称：" . parent::getName(); // 调用父类方法
    }
};
echo $child->getName() . "<br>"; // 输出：子类名称：匿名子类


// 4. 匿名类访问外部变量（通过构造函数）
$prefix = "用户：";
$user = new class($prefix) {
    private string $prefix;

    public function __construct(string $prefix) {
        $this->prefix = $prefix;
    }

    public function getUsername(string $name): string {
        return $this->prefix . $name;
    }
};
echo $user->getUsername("张三") . "<br>"; // 输出：用户：张三


// 5. 匿名类的局限性（不可复用）
// 无法再次实例化同一个匿名类（无类名）
// $anotherUser = new class($prefix); // 这是一个新的匿名类（与上面的$user不同）
?>
```

</details>

### 15. PHP 中的魔术方法（如 __get、__set、__call）有什么作用？如何合理使用魔术方法简化代码？

<details>
<summary>思考与答案（点击展开）</summary>

- 思路要点：

  1. **魔术方法的定义与作用**：  
     魔术方法是PHP预定义的特殊方法（以`__`开头），在特定场景下自动触发，无需手动调用，用于处理动态操作（如访问未定义成员、对象序列化等），增强类的灵活性和动态性。PHP中所有魔术方法及具体作用如下：  

     - `__construct([$params...])`：类的构造函数，在对象实例化（`new Class()`）时自动调用，用于初始化对象（如赋值属性、建立连接）。  
     - `__destruct()`：类的析构函数，在对象被销毁（脚本结束或`unset()`）时自动调用，用于释放资源（如关闭文件、断开数据库连接）。  
     - `__get(string $name)`：当访问对象的**未定义属性**或**不可访问属性**（如`private`/`protected`）时触发，需返回属性值。  
     - `__set(string $name, mixed $value)`：当为对象的**未定义属性**或**不可访问属性**赋值时触发，用于处理属性赋值逻辑。  
     - `__isset(string $name)`：当对对象的**未定义属性**或**不可访问属性**使用`isset()`或`empty()`时触发，返回`bool`表示属性是否存在。  
     - `__unset(string $name)`：当对对象的**未定义属性**或**不可访问属性**使用`unset()`时触发，用于销毁属性或处理清理逻辑。  
     - `__call(string $name, array $arguments)`：当调用对象的**未定义方法**或**不可访问方法**时触发，`$name`为方法名，`$arguments`为参数数组，用于动态处理方法调用。  
     - `__callStatic(string $name, array $arguments)`：当调用类的**未定义静态方法**或**不可访问静态方法**时触发，作用类似`__call`但针对静态方法。  
     - `__toString()`：当对象被当作字符串使用时（如`echo $obj`、`print $obj`）触发，必须返回字符串，否则会抛出`E_RECOVERABLE_ERROR`。  
     - `__invoke([$params...])`：当对象被当作函数调用时（如`$obj()`）触发，允许对象像函数一样被调用，参数和返回值可自定义。  
     - `__clone()`：当使用`clone`关键字复制对象时（如`$newObj = clone $obj`）触发，用于处理对象克隆时的自定义逻辑（如深拷贝属性，避免浅拷贝导致的引用问题）。  
     - `__sleep()`：当对象被序列化（`serialize($obj)`）时触发，返回需要序列化的属性名数组，用于指定哪些属性参与序列化（过滤无需保存的临时数据）。  
     - `__wakeup()`：当对象被反序列化（`unserialize($str)`）时触发，用于重新初始化对象（如重建数据库连接、恢复资源类型属性）。  
     - `__serialize()`：PHP 7.4+ 新增，序列化时优先于`__sleep()`调用，返回待序列化的键值对数组（更灵活的序列化控制）。  
     - `__unserialize(array $data)`：PHP 7.4+ 新增，反序列化时优先于`__wakeup()`调用，接收序列化数据数组并恢复对象状态（更直观的反序列化控制）。  
     - `__set_state(array $properties)`：当使用`var_export($obj)`导出对象并通过其返回值重建对象时（如`$obj = eval('return ' . var_export($obj, true) . ';')`）触发，`$properties`为属性数组，需返回新对象。  
     - `__debugInfo()`：当使用`var_dump($obj)`打印对象时触发，返回用于调试的属性数组，可自定义调试输出内容（隐藏敏感信息或简化输出）。  


  2. **合理使用简化代码的场景**：  
     - **动态属性管理**：用`__get`/`__set`统一处理大量动态属性（如ORM模型映射数据库字段），避免声明冗余私有属性。  
     - **方法动态路由**：用`__call`/`__callStatic`实现“魔术方法”（如`$obj->getUser(1)`自动路由到`get('user', 1)`），简化代码逻辑。  
     - **对象字符串表示**：用`__toString`定义对象的可读性字符串（如打印实体对象时输出关键信息），便于调试和日志记录。  
     - **序列化控制**：用`__sleep`/`__wakeup`或`__serialize`/`__unserialize`过滤敏感数据（如密码），确保序列化安全。  
     - **克隆定制**：用`__clone`处理对象克隆时的深拷贝（如克隆包含子对象的复杂对象），避免原对象与克隆对象的属性引用冲突。  
     - **简化ORM映射**：将数据库字段映射为对象属性，通过魔术方法自动处理读写。

- 实际开发注意事项：

  1. **性能影响**：魔术方法比直接访问属性/方法慢（需额外函数调用），高频操作场景应避免过度使用。
  2. **可读性降低**：过度依赖魔术方法会隐藏实际逻辑（如属性未显式声明），增加维护难度。
  3. **调试困难**：动态属性/方法在IDE中无代码提示，且错误信息可能不明确（如拼写错误导致调用 `__call`）。
  4. **明确边界**：用魔术方法处理“例外情况”（如动态属性），常规属性/方法仍应显式声明。
  5. **配合文档注释**：用 `@property`/`@method`标注动态属性/方法，帮助IDE识别（如 `/** @property string $name */`）。
- 示例代码（PHP 8.1+）：

```php
<?php
// 1. __get/__set：动态属性管理
/**
 * @property string $name
 * @property int $age
 */
class User {
    private array $data = []; // 存储动态属性

    // 访问未定义属性时触发
    public function __get(string $name): mixed {
        if (isset($this->data[$name])) {
            return $this->data[$name];
        }
        throw new InvalidArgumentException("属性{$name}不存在");
    }

    // 设置未定义属性时触发
    public function __set(string $name, mixed $value): void {
        // 对属性值进行验证
        switch ($name) {
            case 'age':
                if (!is_int($value) || $value < 0) {
                    throw new InvalidArgumentException("年龄必须是正整数");
                }
                break;
            case 'name':
                if (!is_string($value) || trim($value) === '') {
                    throw new InvalidArgumentException("姓名不能为空字符串");
                }
                break;
        }
        $this->data[$name] = $value;
    }

    // 检查属性是否存在
    public function __isset(string $name): bool {
        return isset($this->data[$name]);
    }
}

$user = new User();
$user->name = "张三"; // 触发__set
$user->age = 25; // 触发__set
echo "姓名：{$user->name}，年龄：{$user->age}<br>"; // 触发__get
var_dump(isset($user->name)); // 触发__isset，输出：bool(true)


// 2. __call：动态方法处理
class DataProcessor {
    // 调用未定义方法时触发
    public function __call(string $name, array $args): mixed {
        // 解析方法名：如formatDate → 处理日期格式化
        if (str_starts_with($name, 'format')) {
            $type = strtolower(substr($name, 6)); // 提取"Date"部分
            $value = $args[0] ?? null;

            switch ($type) {
                case 'date':
                    return date('Y-m-d', strtotime($value));
                case 'time':
                    return date('H:i:s', strtotime($value));
                default:
                    throw new BadMethodCallException("方法{$name}不存在");
            }
        }
        throw new BadMethodCallException("方法{$name}不存在");
    }

    // 静态版本：__callStatic
    public static function __callStatic(string $name, array $args): mixed {
        return "静态方法{$name}被调用，参数：" . implode(',', $args);
    }
}

$processor = new DataProcessor();
echo "格式化日期：" . $processor->formatDate('2023-10-01 15:30:00') . "<br>"; // 输出：2023-10-01
echo "格式化时间：" . $processor->formatTime('2023-10-01 15:30:00') . "<br>"; // 输出：15:30:00
echo DataProcessor::staticMethod('param1', 'param2') . "<br>"; // 触发__callStatic


// 3. __toString：对象字符串表示
class Product {
    private string $name;
    private float $price;

    public function __construct(string $name, float $price) {
        $this->name = $name;
        $this->price = $price;
    }

    // 对象被当作字符串时触发
    public function __toString(): string {
        return "商品：{$this->name}，价格：¥" . number_format($this->price, 2);
    }
}

$product = new Product("PHP编程指南", 59.9);
echo $product . "<br>"; // 触发__toString，输出：商品：PHP编程指南，价格：¥59.90
?>
```

</details>
