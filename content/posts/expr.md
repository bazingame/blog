---
title: "表达式解析器规则引擎库 — expr"
summary: 表达式解析器规则引擎库 — expr
date: 2022-08-19
weight: 1
tags: ["golang"]
---
expr 提供了类似 JavaScript 中 `eval` 功能，提供了解析、执行表达式的功能。可以用于计算任意表达式的值。类似功能函数在 JavaScript 、Python 等动态语言中比较常见。`expr`让 Golang 个编译型语言也具备了该种能力。

## 基本使用

计算两个值的和:

```go
package main

import (
	"fmt"
	"github.com/antonmedv/expr"
)

func main() {
	env := map[string]interface{}{
		"foo": 1,
		"bar": 2,
	}

	out, err := expr.Eval("foo + bar", env)

	if err != nil {
		panic(err)
	}
	fmt.Print(out)
}
```

调用复杂函数：

```go
package main

import (
	"fmt"

	"github.com/antonmedv/expr"
)

func main() {
	env := map[string]interface{}{
		"greet":   "Hello, %v!",
		"names":   []string{"world", "you"},
		"sprintf": fmt.Sprintf, // You can pass any functions.
	}

	code := `sprintf(greet, names[0])`

	// Compile code into bytecode. This step can be done once and program may be reused.
	// Specify environment for type check.
	program, err := expr.Compile(code, expr.Env(env))
	if err != nil {
		panic(err)
	}

	output, err := expr.Run(program, env)
	if err != nil {
		panic(err)
	}

	fmt.Print(output)
}
```

引入结构体:

```go
package main

import (
	"fmt"
	"time"

	"github.com/antonmedv/expr"
)

type Env struct {
	Tweets []Tweet
}

// Methods defined on such struct will be functions.
func (Env) Format(t time.Time) string { return t.Format(time.RFC822) }

type Tweet struct {
	Text string
	Date time.Time
}

func main() {
	code := `map(filter(Tweets, {len(.Text) > 0}), {.Text + Format(.Date)})`

	// We can use an empty instance of the struct as an environment.
	program, err := expr.Compile(code, expr.Env(Env{}))
	if err != nil {
		panic(err)
	}

	env := Env{
		Tweets: []Tweet{{"Oh My God!", time.Now()}, {"How you doin?", time.Now()}, {"Could I be wearing any more clothes?", time.Now()}},
	}

	output, err := expr.Run(program, env)
	if err != nil {
		panic(err)
	}

	fmt.Print(output)
}
```

更多详细的使用及支持的语法可见: [expr/Language-Definition.md at master · antonmedv/expr (github.com)](https://github.com/antonmedv/expr/blob/master/docs/Language-Definition.md)

## Summary



## Reference

[antonmedv/expr: Expression language for Go (github.com)](https://github.com/antonmedv/expr)

