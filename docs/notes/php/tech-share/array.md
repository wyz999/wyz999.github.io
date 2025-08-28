---
title: PHP 数组：易错点、性能与原理
createTime: 2025/08/19 12:30:00
permalink: /php/技术分享/array/
---
> 这是一篇面向实践的数组专题分享，聚焦常见易错点、性能考量与底层原理，帮助在真实业务中写出更稳健高效的 PHP 代码。

## 1. array_merge 与 + 的区别

- `array_merge`：合并数组，遇到相同的字符串键会被后者覆盖，数字键会被重新索引。
- `+`（数组并集）：保留左侧数组已有键，右侧相同键的值会被忽略，数字键不重新索引。

示例：

```php
$a = ['a' => 1, 10 => 'x'];
$b = ['a' => 2, 20 => 'y'];
var_dump(array_merge($a, $b)); // ['a' => 2, 0 => 'x', 1 => 'y']
var_dump($a + $b);             // ['a' => 1, 10 => 'x', 20 => 'y']
```

## 2. in_array 与 array_search 的差异（严格比较）

- `in_array($needle, $haystack, true)` 第三个参数为 true 时启用全等比较，避免 '0' 与 0、false 混淆。
- `array_search($needle, $haystack, true)` 返回匹配到的键；若返回 `0` 别误判为未找到，需全等判断 `!== false`。

```php
$xs = ['0', 1, false];
var_dump(in_array(0, $xs));          // true（非严格）
var_dump(in_array(0, $xs, true));    // false（严格）
var_dump(array_search(false, $xs));   // 2
```

## 3. 数组拷贝与引用

- `$b = $a` 是值拷贝（写时复制），但对嵌套对象/引用要谨慎。
- `&$` 引用会联动修改，不熟悉时尽量避免对临时变量取引用。

```php
$a = [1, 2];
$b = &$a;
$b[0] = 100;
var_dump($a[0]); // 100
```

## 4. 排序函数的稳定性与键保持

- `sort/rsort` 会重建数字索引；`asort/arsort` 会保持键；`usort` 使用自定义比较。
- 自定义比较函数需返回负/零/正整型，避免返回布尔值导致未定义行为。

```php
$xs = ['b' => 2, 'a' => 1];
asort($xs); // 保持键：['a'=>1,'b'=>2]
```

## 5. 数组指针函数的「隐形陷阱」

PHP 数组指针（`current()`/`next()`/`prev()`/`end()`/`reset()`）操作后会改变内部指针位置，易导致后续遍历异常。

- 遍历后未重置指针，可能导致二次遍历失败
- `each()` 已废弃（PHP 7.2+），需用 `foreach` 替代
- `end()` 会移动指针到最后元素，影响后续 `current()` 取值

```php
$arr = ['a', 'b', 'c'];
next($arr); // 指针移到 'b'
echo current($arr); // 输出 'b'

// 未重置指针直接遍历，会从 'b' 开始
foreach ($arr as $v) {
  echo $v; // 输出 'bc'（遗漏 'a'）
}

reset($arr); // 重置指针到开头（修复关键）
```

## 6. 数组键的「类型自动转换」

PHP 数组键会自动转换类型，可能导致意外覆盖：

- 字符串键若为纯数字（如 `'123'`）会转为整数 `123`
- 浮点数键会截断为整数（如 `5.9` → `5`）
- 布尔值键会转为整数（`true`→`1`，`false`→`0`）
- `null` 键转为空字符串 `''`

```php
$arr = [
  '123' => 'a',
  123 => 'b',       // 覆盖上面的 '123' 键
  5.9 => 'c',
  5 => 'd',         // 覆盖 5.9 键
  true => 'e',
  1 => 'f'          // 覆盖 true 键
];
var_dump(count($arr)); // 输出 3（实际保留 123、5、1 三个键）
```

## 7. isset() 与 array_key_exists() 的「存在性判断」

- `isset($arr[$key])`：检查键存在且值不为 `null`（值为 `null` 时返回 `false`）
- `array_key_exists($key, $arr)`：仅检查键是否存在（值为 `null` 也返回 `true`）

```php
$arr = ['name' => null, 'age' => 20];
var_dump(isset($arr['name'])); // false（值为 null）
var_dump(array_key_exists('name', $arr)); // true（键存在）
```

## 8. array_slice 与 array_splice 的「增删差异」

- `array_slice($arr, $offset, $length)`：返回子数组，**不修改原数组**，保留键（数字键会重置，关联键保留）
- `array_splice($arr, $offset, $length, $replacement)`：**修改原数组**（删除指定元素并插入替换内容），返回被删除的元素

```php
$arr = ['a', 'b', 'c', 'd'];
// 取子数组，原数组不变
var_dump(array_slice($arr, 1, 2)); // ['b', 'c']
var_dump($arr); // 仍为 ['a', 'b', 'c', 'd']

// 修改原数组，删除并替换
var_dump(array_splice($arr, 1, 2, ['x', 'y'])); // 返回被删的 ['b', 'c']
var_dump($arr); // 变为 ['a', 'x', 'y', 'd']
```

## 9. 多维数组的「引用传递」

处理多维数组时，引用传递容易导致意外修改原数组，尤其是在循环中。

```php
$users = [
  ['name' => 'Alice', 'age' => 20],
  ['name' => 'Bob', 'age' => 22]
];

// 错误：未解除引用，导致后续操作影响原数组
foreach ($users as &$user) {
  $user['age']++;
}

// 此时 $user 仍引用最后一个元素
$user = ['name' => 'Charlie'];
var_dump($users[1]); // 被修改为 ['name' => 'Charlie']（意外！）

// 修复：遍历后 unset 引用
unset($user);
```

## 10. 数组内存优化：写时复制（Copy-on-Write）机制

PHP 数组采用写时复制机制，理解这一点对大数据处理至关重要：

```php
$large_array = range(1, 1000000);
$copy = $large_array;  // 此时未真正复制，两者共享内存

// 只有在修改时才会触发真正的复制
$copy[0] = 'modified'; // 触发写时复制，此时内存翻倍

// 利用引用避免不必要的复制
function process_array(&$arr) {
    foreach ($arr as $key => $value) {
        // 只读操作，不会触发复制
        if ($value > 500000) break;
    }
}
```

**性能影响**：大数组传参时，使用引用 `&$arr` 比值传递性能更优，避免不必要的内存分配。

## 11. 数组哈希冲突与性能退化

PHP 数组底层是哈希表实现，特定键模式可能导致哈希冲突，性能从 O(1) 退化到 O(n)：

```php
// 避免连续整数键产生的哈希冲突（极端情况）
$inefficient = [];
for ($i = 1000000; $i < 1000100; $i += 7) {
    $inefficient[$i] = "value_$i"; // 某些步长可能导致冲突
}

// 更优做法：预分配或使用 SplFixedArray（纯数字索引）
$efficient = new SplFixedArray(100);
for ($i = 0; $i < 100; $i++) {
    $efficient[$i] = "value_$i";
}
```

## 12. 数组序列化的「精度丢失」与类型还原

`serialize/unserialize` 和 `json_encode/json_decode` 对数据类型处理有差异：

```php
$original = [
    'float' => 3.14159265359,
    'large_int' => PHP_INT_MAX,
    'resource' => fopen('php://memory', 'r')
];

// JSON 序列化：精度丢失，资源类型丢失
$json = json_encode($original);
$from_json = json_decode($json, true);
// float 精度丢失，resource 变为 null

// PHP 原生序列化：保持类型，但资源无法序列化
$serialized = serialize($original);
// 警告：无法序列化 resource

// 最佳实践：序列化前预处理数据
$safe_data = array_filter($original, function($value) {
    return !is_resource($value);
});
```

## 13. 数组键顺序的「插入顺序保持」特性

PHP 数组会保持插入顺序（从 PHP 7.0 开始保证），这影响 `foreach` 遍历顺序：

```php
$arr = [];
$arr['c'] = 3;
$arr['a'] = 1;
$arr['b'] = 2;

foreach ($arr as $key => $value) {
    echo "$key => $value\n"; // 输出：c => 3, a => 1, b => 2（插入顺序）
}

// 若需要按键排序，必须显式调用
ksort($arr);
foreach ($arr as $key => $value) {
    echo "$key => $value\n"; // 输出：a => 1, b => 2, c => 3（字典序）
}
```

**实际意义**：JSON 对象序列化时，键顺序得以保持，与前端交互更可预测。

---

面试提示：回答数组相关问题时，重点展示对 PHP「弱类型特性」（如键类型转换、松散比较）的理解，以及对「函数副作用」（如修改原数组的函数）的掌握。高级话题如写时复制机制、哈希冲突性能退化等，体现了对 PHP 数组底层实现的深度理解。
