---
title: assert-2020.01.01
date: 2020-01-01 20:50:53
tags: [node]
---

## assert 断言

`assert` 模块提供了一组简单的断言测试，可用于测试不变量。 该模块提供了建议的 **_严格模式_** 和更宽松的 **_遗留模式_**。

## assert.AssertionError 类

`Error` 的子类，表明断言的失败。 `assert` 模块抛出的所有错误都将是 `AssertionError` 类的实例。

## new assert.AssertionError(options)

- options `<Object>`
  - message `<string>` 如果提供，则将错误消息设置为此值。
  - actual `<any>` 错误实例上的 `actual` 属性将包含此值。
  - expected `<any>` 错误实例上的 `expected` 属性将包含此值。
  - operator `<string>` 错误实例上的 `operator` 属性将包含此值。
  - stackStartFn `<Function>` 如果提供，则生成的堆栈跟踪将移除所有帧直到提供的函数。

所有实例都包含内置的 `Error` 属性（`message` 和 `name`）以及：

- actual `<any>` 设置方法的 actual 参数，例如 `assert.strictEqual()`。
- expected `<any>` 设置方法的 expected 参数，例如 `assert.strictEqual()`。
- generatedMessage `<boolean>` 表明消息是否是自动生成的。
- code `<string>` 始终设置为字符串 **ERR_ASSERTION** 以表明错误实际上是断言错误。
- operator `<string>` 设置为传入的运算符值。

## strict 严格模式

```javascript
const assert = require("assert").strict;
```

## legacy 遗留模式

```javascript
const assert = require("assert");
```

尽量使用 **_严格模式_**，**_遗留模式_**的 **抽象的相等性比较** 可能会有意外结果。
eg：

```javascript
// 注意：这不会抛出 AssertionError！
assert.deepEqual(/a/gi, new Date());
```

## assert(value[, message])

```javascript
assert(false, "结果为假");
```

## assert.deepStrictEqual(actual, expected[, message])

```javascript
assert.deepStrictEqual(1, 2, "1和2不相等");
```
