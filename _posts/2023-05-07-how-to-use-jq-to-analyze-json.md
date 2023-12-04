---
layout: post
title: "使用 jq 分析 json 文件"
description: "使用 jq 分析 json 文件"
date: 2023-05-07
tags: [tools, jq]
---

经常要分析 json 文件，如果用肉眼查看，实在是太累了。
使用 jq 来过滤需要的值将大大增加工作效率。

# 安装 jq

```shell
yum install -y jq
```

```shell
apt-get install -y jq
```

# jq 的命令介绍

## 基本概念

jq 的语法是清晰的:

| 语法 |  描述  |
| --------| :------------:|
| , | 由逗号分割的多个部分将会产生多个独立的输出 |
| ? | 如果这个类型错误将会被忽略 |
| [] | 构建数组 |
| {} | 构建对象 |
| + | 拼接或者加法 |
| - | 差集或者减法 |
| length | 数组的长度 |
| &#124; | 将多个 jq 命令拼接，类似于 shell 管道 |


## 处理 json 对象

| 描述 | 命令 |
| ------------| :-----: |
| 提取所有的键值 | `jq 'keys'` |
| 给所有的键值加 1 | `jq 'map_values(.+1)'` |
| 删除一个键值 | `jq 'del(.foo)'` |
| 将一个对象转换为一个数组 | `to_entries &#124; map([.key, .value])` |
| 提取一个对象下面的数组的所有 instruction 成员 | `cat docker.json \| jq '.[0].layers[].instruction'`, 特别注意，访问layers对象下面的数组，不需要加上点号 |

## 处理成员

| 描述 | 命令 |
| ------------| :-----: |
| 拼接两个成员 | `fieldNew=.field1+" "+.field2` |


## 处理 json 数组

### 分片和过滤

| 描述 | 命令 |
| ------------| :-----: |
| All | `jq .[]` |
| First |	`jq '.[0]'` |
| Range | `jq '.[2:4]'` |
| First 3 | `jq '.[:3]'` |
| Last 2 | `jq '.[-2:]'` |
| Before Last | `jq '.[-2]'`|
| Select array of int by value | `jq 'map(select(. >= 2))'` |
| Select array of objects by value | jq '.[] &#124; select(.id == "second")' |
| Select by type | jq '.[] &#124; numbers' <br>with type been arrays, objects, iterables, booleans, numbers, normals, finites, strings, nulls, values, scalars |

### 映射和转换

| 描述 | 命令 |
| ------------| :-----: |
| Add + 1 to all items | `jq 'map(.+1)'` |
| Delete 2 items| `jq 'del(.[1, 2])'` |
| Concatenate arrays | `jq 'add'` |
| Flatten an array | `jq 'flatten'` |
| Create a range of numbers | `jq '[range(2;4)]'` |
| Display the type of each item| `jq 'map(type)'` |
| Sort an array of basic type| `jq 'sort'` |
| Sort an array of objects | `jq 'sort_by(.foo)'` |
| Group by a key - opposite to flatten | `jq 'group_by(.foo)'` |
| Minimun value of an array| `jq 'min'` .See also  min, max, min_by(path_exp), max_by(path_exp) |
| Remove duplicates| `jq 'unique'` or `jq 'unique_by(.foo)'` or `jq 'unique_by(length)'` |
| Reverse an array | `jq 'reverse'` |

# 通过一份文件学习 jq 的操作

## 测试数据

https://github.com/joseluisq/json-datasets/blob/master/json/programming-languages/programming_languages_keywords.json


## 操作实践

我们先要了解一个 json 对象的整体结构，那么就需要获取这个对象有哪些键值。

```shell
$ jq "keys" programming_languages_keywords.json
[
  "data"
]
```

获取某一个 json 对象的键值对应的值。

```shell
$ jq ".data" programming_languages_keywords.json

[
  {
    "name": "Lua",
    "version": 5.3,
    "summary": "Lua is a powerful, efficient, lightweight, embeddable scripting language. It supports procedural programming, object-oriented programming, functional programming, data-driven programming and data description.",
    "extensions": [
      "lua"
    ],
    "keywords": [
      "and",
...
]
```

发现上面的输出太多了，实际上我们只要数组中的一个成员

```shell
$ jq ".data[0]" programming_languages_keywords.json
{
  "name": "Lua",
  "version": 5.3,
  "summary": "Lua is a powerful, efficient, lightweight, embeddable scripting language. It supports procedural programming, object-oriented programming, functional programming, data-driven programming and data description.",
  "extensions": [
    "lua"
  ],
  "keywords": [
    "and",
    "break",
    "do",
    "else",
    "elseif",
    "end",
    "false",
    "for",
    "function",
    "goto",
    "if",
    "in",
    "local",
    "nil",
    "not",
    "or",
    "repeat",
    "return",
    "then",
    "true",
    "until",
    "while"
  ],
  "sources": [
    "https://www.lua.org/manual/5.3/manual.html#3.1"
  ]
}
```

如果更进一步，可以得到语言的名称，那么可以使用如下命令

```shell
$ jq ".data[0].name" programming_languages_keywords.json
"Lua"
```

如果我们想要得到数组的所有的编程语言，那么使用如下命令

```shell
$ jq ".data[].name" programming_languages_keywords.json
"Lua"
"Go"
"C"
"Python"
"Python"
"C"
"Java"
"C++"
"C#"
"PHP"
"R"
"JS"
"Ruby"
"Swift"
"Scala"
"Erlang"
"Rust"
"Kotlin"
"Elixir"
"Dart"
"Fortran"
```

我们可能希望得到的是编程语言和它对应的描述，那么使用如下的命令

```shell
$ jq ".data[] | .name, .summary" programming_languages_keywords.json
"Lua"
"Lua is a powerful, efficient, lightweight, embeddable scripting language. It supports procedural programming, object-oriented programming, functional programming, data-driven programming and data description."
"Go"
"Go is an open source programming language that makes it easy to build simple, reliable, and efficient software."
"C"
"C18 is the informal name for ISO/IEC 9899:2018, the most recent standard for the C programming language."
```

如果我们希望输出的是一个 json 格式的数据，那么可以使用如下的命令。
这里要注意最外面我们加了一个中括号，你以去掉比较一下效果。

```shell
jq "[ .data[] | [ .name, .summary ] ]" programming_languages_keywords.json
```

可以比较一下下面这个命令加深对 jq 选择器的了解

```shell
jq "[ .data[].name, .data[].summary ]" programming_languages_keywords.json
```

虽然上面得到的是一个数组，但是我们希望是 KV 格式, 因此使用下面的格式。
注意作为 KEY 的 .name 要加上圆括号。

```shell
jq "[ .data[] | { (.name):.summary } ]" programming_languages_keywords.json

# or

jq ".data | map({(.name): .summary})" programming_languages_keywords.json
```

我们想要知道有多少个编程语言，可以是使用如下的命令:

```shell
jq ".data | length" programming_languages_keywords.json
```

选择指定范围的数组成员, 比如我们想选择前 10 名的编程语言

```shell
jq ".data[0:9]" programming_languages_keywords.json
```

或者选择最后 10 名编程语言

```shell
jq ".data[-10:]" programming_languages_keywords.json
```

我们可以想要根据一定的规则选出数组中的成员，这个时候可以用 select。
比如这里选择的是关键词数量超过 100 的编程语言。

```shell
jq ".data | map(select(.keywords | length > 100))" programming_languages_keywords.json
```

如果我们要选择的是包含 submodule 这个关键词的编程语言又应该怎么选择呢？
注意下面的命令中外层引号使用的是单引号，内层使用的的双引号。
这个是以为 jq 的语法要求使用双引号表示字符串。外层的使用单引号是为了让
shell 识别这是一个字符串。如果没有外层的单引号，那么就会被shell识别成好多个不同的命令行参数。

```shell
jq '.data[] | select(.keywords[] | contains ("submodule")) | .name' programming_languag_languages_keywords.json
```

如果我们要选择的是 go 这个关键词，我们用上面的语法会发现 contains 是不对的。我们应该用 == 比较操作符。

```shell
jq '.data[] | select(.keywords[] == "go") | .name' programming_languag_languages_keywords.json
```

如果我们想要对数组进行排序，可以使用 sort 这个函数对基本类型进行排序，也可以使用 sort_by 这个函数对对象进行排序。

```shell
jq '.data | sort_by(.version)' programming_languages_keywords.json
```

# 参考文档

执行 `man jq` 查询帮助是最快捷的方式

此外还有如下文档

https://stackoverflow.com/questions/tagged/jq
https://stedolan.github.io/jq/manual/
