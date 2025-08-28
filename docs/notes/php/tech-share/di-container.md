---
title: 原生 PHP 实现一个最小化依赖注入容器（支持自动注入与单例）
tags:
  - php
  - di
  - ioc
  - container
createTime: 2025/08/19 15:30:00
permalink: /php/技术分享/di-container/
---

本文用原生 PHP 从零实现一个最小化的依赖注入（DI）容器，支持：
- 自动解析构造函数依赖（基于反射）
- 绑定接口到实现（interface -> class / 闭包）
- 单例 / 多例 两种生命周期

适合用作学习 IoC/DI 基础、快速验证想法或在小型项目中简单使用。

## 1. 为什么需要依赖注入容器？
- 解耦：把“如何创建依赖”的细节从业务代码中拿出来，统一交给容器管理。
- 可测试：通过绑定与替身替换（如将接口绑定到 Mock 实现），更容易进行单元测试。
- 可维护/可扩展：当实现类变化时，无需修改业务构造处的 new 语句，只需调整绑定。

## 2. 设计目标与能力
- bind(id, concrete, shared)：绑定接口/类到实现，concrete 可为类名或闭包，shared 表示是否单例。
- singleton(id, concrete)：单例绑定的快捷方式。
- make(id, params)：解析并创建实例，支持手动传参覆盖基本类型/默认值。
- build(class, params)：通过反射构建实例，自动递归解析类依赖。

## 3. 完整实现

```php
<?php

/**
 * 依赖注入容器实现
 * 用原生 PHP 实现一个最小化的 DI 容器，支持自动解析类构造函数依赖、绑定接口到实现、单例/多例两种生命周期。
 */

class Container
{
    /** @var array 存储所有绑定关系 [id => [concrete, shared]] */
    private array $bindings = [];
    /** @var array 存储单例实例缓存 [id => instance] */
    private array $singletons = [];

    /**
     * 绑定接口/类到具体实现
     * @param string $id 接口名或类名
     * @param mixed $concrete 具体实现类名或闭包，null时使用$id本身
     * @param bool $shared 是否为单例模式
     */
    public function bind(string $id, mixed $concrete = null, bool $shared = false): void
    {
        $this->bindings[$id] = ['concrete' => $concrete ?? $id, 'shared' => $shared];
    }

    /**
     * 注册单例绑定（shared=true的快捷方法）
     * @param string $id 接口名或类名
     * @param mixed $concrete 具体实现
     */
    public function singleton(string $id, mixed $concrete = null): void
    {
        $this->bind($id, $concrete, true);
    }

    /**
     * 从容器中解析并创建实例
     * @param string $id 要解析的接口/类名
     * @param array $params 手动传入的构造参数
     * @return mixed 解析后的实例
     */
    public function make(string $id, array $params = [])
    {
        // 如果是单例且已实例化，直接返回缓存
        if (isset($this->singletons[$id])) {
            return $this->singletons[$id];
        }

        // 获取绑定信息，未绑定时使用原始类名
        $binding = $this->bindings[$id] ?? ['concrete' => $id, 'shared' => false];
        $concrete = $binding['concrete'];
        $shared   = $binding['shared'];

        // 根据concrete类型决定实例化方式
        if ($concrete instanceof Closure) {
            // 如果是闭包，直接调用
            $object = $concrete($this, $params);
        } else {
            if (!is_string($concrete)) {
                throw new RuntimeException("无效的绑定：concrete 必须是类名字符串或闭包");
            }
            // 如果是类名，通过反射构建
            $object = $this->build($concrete, $params);
        }

        // 如果是单例模式，缓存实例
        if ($shared) {
            $this->singletons[$id] = $object;
        }
        return $object;
    }

    /**
     * 通过反射构建类实例，自动解析构造函数依赖
     * @param string $class 要构建的类名
     * @param array $params 手动传入的参数
     * @return object 构建的实例
     */
    private function build(string $class, array $params)
    {
        $ref = new ReflectionClass($class);
        if (!$ref->isInstantiable()) {
            throw new RuntimeException("$class 不可实例化");
        }

        $ctor = $ref->getConstructor();
        // 无构造函数，直接new
        if (!$ctor) return new $class;

        $args = [];
        // 遍历构造函数参数，自动解析依赖
        foreach ($ctor->getParameters() as $p) {
            $type = $p->getType();
            if (!$type || $type->isBuiltin()) {
                // 基本类型或无类型提示：使用手动参数或默认值
                $args[] = $params[$p->name] ?? ($p->isDefaultValueAvailable() ? $p->getDefaultValue() : throw new RuntimeException("缺少参数 {$p->name}"));
            } else {
                // 类类型：递归从容器解析
                $args[] = $this->make($type->getName());
            }
        }
        return $ref->newInstanceArgs($args);
    }
}
```

## 4. 使用示例

```php
<?php

// ========== 测试用例 ==========
interface Mailer {}
class SmtpMailer implements Mailer {}
class UserService { public function __construct(public Mailer $mailer) {} }

$c = new Container();
$c->bind(Mailer::class, SmtpMailer::class);          // 绑定接口到实现
$c->singleton(UserService::class);                   // 注册为单例

$u1 = $c->make(UserService::class);                  // 第一次创建
$u2 = $c->make(UserService::class);                  // 第二次获取

assert($u1 === $u2, '单例');                          // 验证单例模式
assert($u1->mailer instanceof SmtpMailer, '自动注入'); // 验证依赖注入

echo "DI container CLI test passed\n";
```

- 自动注入：UserService 构造函数中声明了 Mailer 类型，容器在解析时会先根据绑定规则把接口 Mailer 解析为 SmtpMailer，再实例化并注入。
- 单例：对 UserService 使用 singleton 绑定，首个 make() 实例化后会缓存，后续多次 make() 返回同一实例。

## 5. 生命周期：单例 vs 多例
- 多例（默认）：每次 make(id) 都会重新创建一个实例。
- 单例（shared）：通过 singleton() 或 bind(id, concrete, true) 声明，容器只创建一次，并在 singletons 池中复用。

## 6. 自动注入机制细节
- 通过 ReflectionClass + 构造函数参数反射获取依赖列表。
- 对“类类型”参数：递归调用 make() 实现自动解析。
- 对“基本类型/无类型提示”参数：优先使用 make() 调用时传入的 $params，其次使用形参默认值，否则抛出“缺少参数 xxx”。

## 7. 常见错误与提示
- 绑定为非字符串且非闭包：抛出 “无效的绑定：concrete 必须是类名字符串或闭包”。
- 类不可实例化（如抽象类、接口或构造受保护）：抛出 “Xxx 不可实例化”。
- 基本类型/无默认值参数未提供：抛出 “缺少参数 name”。

## 8. 可扩展方向
- 别名与上下文绑定：根据不同上下文为同一接口绑定不同实现。
- 方法注入 / 属性注入：除构造函数外的更多注入方式。
- 作用域：请求作用域、协程作用域等更细粒度的生命周期控制。
- 延迟解析：支持代理/延迟加载以提升性能。
- 配合配置文件/注解：统一管理绑定定义。

## 9. 小结
本文实现了一个精简而完整的 DI 容器，包括绑定、自动解析与生命周期控制。你可以据此快速在项目中使用，或作为学习 IoC/DI 的起点，再逐步演化出更强的功能。
