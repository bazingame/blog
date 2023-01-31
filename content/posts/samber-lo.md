---
title: "基于泛型的 Golang lodash 库 — samber/lo"
summary: 基于泛型的 Golang lodash 库 — samber/lo
date: 2022-03-28
weight: 1
tags: ["golang"]
---


近日，Go 核心开发团队终于宣布了 Go 1.18 正式版本的发布！这是一个大家期待很久的版本！Go 1.18 包含大量新功能：模糊测试、性能改进、工作区等，以及 Go 语言开源以来最大的一次语法特性变更 —— 支持泛型！

支持泛型后，我们便不再需要写如下冗余的代码：

![image.png](https://cdn.gocn.vip/forum-user-images/20220318/810e554336b642359a253dba7579c319.jpg)

现在只需要简单的一行即可：

```go
func Min[T constraints.Ordered](a, b T) T {	if a < b {	return a };	return b}
```

## lodash

在 JavaScript 的世界里，`lodash.js` 是一个一致性、模块化、高性能的 JavaScript 实用工具库，其通过降低 array、number、objects、string 等等的使用难度让 JavaScript 变得更简单。并且其不需要引入其他第三方依赖。

我们可以直接调用其中封装好的方法，比如数组去重，防抖函数等等，简化很多代码。

比如去重：

```javascript
import _ from 'lodash'
_.uniq([2, 1, 2]);
// => [2, 1]
```

比如过滤掉数组中不符合规则的元素：

```javascript
var users = [
  { 'user': 'barney', 'age': 36, 'active': true },
  { 'user': 'fred',   'age': 40, 'active': false }
];
 
_.filter(users, function(o) { return !o.active; });
// => objects for ['fred']
 
// The `_.matches` iteratee shorthand.
_.filter(users, { 'age': 36, 'active': true });
// => objects for ['barney']
```

## samber/lo

在 Golang 支持泛型之前，实现像 `lodash.js` 这样一套适配多种数据类型的完整的工具库是非常不容易的。有一些开源库通过其他方式实现了部分功能，大致有三种方案：

- **纯手撸** - 毫无疑问，这种方式是最不优雅的，需要对每种类型进行开发，需要做很多无聊的工作。
- **代码生成** - 通过脚本辅助生成针对不同类型的工具函数，比如 [go-dash/slice](https://github.com/go-dash/slice)。
- **使用反射** - 这种方式可以实现目的，但是反射会带来较大复杂度和造成运行时性能的下降。[go-funk](https://github.com/thoas/go-funk) 和[robpike/filter](https://github.com/robpike/filter)都是通过该种方式实现的工具库。

**`samber/lo`** 是一个基于 Golang 泛型实现的的 lodash 风格工具库，比较好的避免了上面的问题。

`samber/lo` 包含了非常多的方法，主要可以划分为以下几类：

- slice 辅助方法
- map 辅助方法
- tuples 辅助方法
- 多个集合之间计算辅助方法
- 搜索查询辅助方法
- 其他函数式编程辅助方法等

以切片去重举例：

```go
names := lo.Uniq[string]([]string{"Samuel", "Marc", "Samuel"})
// []string{"Samuel", "Marc"}
```

调用非常简单，并且在大多数情况下，我们可以省略类型的指定:

```go
names := lo.Uniq([]string{"Samuel", "Marc", "Samuel"})
// []string{"Samuel", "Marc"}
```

再比如过滤掉切片中不符合规则的元素：

```go
even := lo.Filter([]int{1, 2, 3, 4}, func(x int, _ int) bool {
    return x%2 == 0
})
// []int{2, 4}
```

## Summary

`samber/lo` 基于泛型包装了非常多的工具方法，可以大大节省我们的开发时间，避免重复开发，提升效率。但是该库开源至今才两周，可能会有一些问题缺陷存在其中，线上使用还需要谨慎一些。

## Reference

[Go 1.18 is released! - The Go Programming Language](https://go.dev/blog/go1.18)

[The benefits of using Lodash in the Go language without reflection (freecodecamp.org)](https://www.freecodecamp.org/news/the-benefits-of-using-lodash-in-the-go-language-without-reflection-1d64b5115486)

[samber/lo: 💥 A Lodash-style Go library based on Go 1.18+ Generics (map, filter, contains, find...) (github.com)](https://github.com/samber/lo)
