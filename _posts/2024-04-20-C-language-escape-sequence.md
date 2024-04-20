---
layout: post
title: "再论 C 语言的字符转义"
description: "一次 Bug 重新认识字符转义"
date: 2024-04-20
tags: [C, Escape Sequence]
---

# 问题来了

虽然 `C` 语言的语法特性已经是最简单的了，但是作为一个 `C` 语言的老鸟，免不了还是要栽在 `C` 语言的大坑里。

最近在做一个项目，需要将多个字符串拼接为 `LV` 格式的字符串。
这里的 `LV` 是代表 `Length Value` 格式，而不是奢侈品的 `LV` 包包的 `LV`。

因为子字符串的长度只能小于 256 字节, 因此 `L` 用一个字节来表示。
比如 "abc" "com" 两个字符串要表示成 `LV` 格式就是 "\x03abc\x03com"。 

可以看到我们上面用 16 进制的转义系列。

但是，问题就出在 16 进制的转义系列。乍一看还以为很完美，我们一直都是这么搞 16 进制转义系列的。

比如常见的 `to_hex_str()` 这样的函数将字符串转义为16进制，而 `from_hex_str()` 将 16 进制字符串转换成字节序列。

像 Shell 的 hexdump 就像将字节系列转换为 16 进制。下面就是 `Hello world` 的 16 进制表示。

```shell
echo -n "Hello world" | hexdump -C
00000000  48 65 6c 6c 6f 20 77 6f  72 6c 64                 |Hello world|
0000000b
```

在 C 语言中我们可以直接这样表示：

```C
char s[] = "Hello world";
```

也可以用 16 进制表示：

```C
char s[] = "\x48\x65\x6c\x6c\x6f\x20\x77\x6f\x72\x6c\x64";
```

那么到底问题出现在哪里呢？这就要从 16 进制的语法表示说起。

# 转义字符的语法表示

- escape-sequence:
  - simple-escape-sequence
  - octal-escape-sequence
  - hexadecimal-escape-sequence
  - universal-character-name
- simple-escape-sequence: one of
  - \' \" \? \\
  - \a \b \f \n \r \t \v
- octal-escape-sequence:
  - \ octal-digit
  - \ octal-digit octal-digit
  - \ octal-digit octal-digit octal-digit

- hexadecimal-escape-sequence:
  - \x hexadecimal-digit
  - hexadecimal-escape-sequence hexadecimal-digit

从上面可以看到，8 进制可以用 1~3 个数字表示，而 16 进制则是递归的方式。
因此从语法上讲，`\xa`, `\xab`, `\xabc` 都是 16 进制表示。

然而这就是一个坑，或许当初专家为了节省储存空间，理所当然认为大家都知道这些门门道道。
没有想到世界发展如此之快，电脑变得唾手可得，普通人都能成为一个程序员。

专家挖坑，普通人踩坑，大家都有收获。 :joy: :boom:

讲这么多，还是不容易理解，那么只能 `show me the fucking code`。

# C 代码片段

```C
#include <stdio.h>

int main(int argc, char **argv)
{
    unsigned char s[] = "\x0123";
    for (int i = 0; i < sizeof(s); i++) {
        printf("%02x\n", s[i]);
    }

    return 0;
}
```

将上面的代码保持为 t.c, 然后执行编译得到结果如下：

```shell
$ gcc -Wall t.c
t.c: In function ‘main’:
t.c:5:33: warning: hex escape sequence out of range
    5 |     unsigned char s[] = "\x0123";
      |                                 ^
```

程序编译通过了，但是有警告，这就是该死的 16 进制转义。。

同时也应该记住，-Wall -Werror 是良好的实践：死在前面总比不知道怎么死的来得好。


# 解决问题

那么如何解决 16 进制转义呢？其中一个方法就是使用像 `Helllo world` 那样的方式，将所有的字节转义。
比如 "abc" "com" 的 `LV` 的 Hex 表示是： "\x03\x61\x62\x63\0x03\x63\x6f\x6d"。

但是这种转义方式存在的问题就是代码阅读很不方便。我们的目标就是要混合转义和非转义字符。
这种情况下最好的方式就是使用 3 个八进制的转义字符了。因此上面的 "abc" "com" 两个字符串
的 `C` 转义系列就可以表示成 "\003abc\003com"。

当然，如果你属于艺高人胆大的，还是可以通过判断 16 进制后面跟的是不是合法的 16 进制字符来实现
混合 16 进制转义， 8 进制转义来达到目标的。但是 16 进制需要多一个 x 字符，所以其实多数情况下还是用 4 个字节的转义系列表示一个字节，并没有比 8 进制来得少。

欢迎入坑 `C` 语言。
