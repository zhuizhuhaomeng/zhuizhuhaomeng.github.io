---
layout: post
title: "如何安全拷贝字符串"
description: "strncpy vs snprintf"
date: 2024-01-23
tags: [strncpy, memset, stpncpy, mempcpy, snprintf]
---

# 安全拷贝字符串

很多面谈题喜欢让人回答如何才能安全拷贝字符串。要比较 `strcpy` 和 `strncpy` 的区别。
事实上，大多数的面试官是不知道 `strncpy` 具体是怎么实现，有什么副作用的。

我们通过 `man strncpy` 可以清楚的看到，`strncpy` 其实执行了一个 `bzero` 的动作，也就是将
目标地址空间的内存清 0 了。

```C
char *
strncpy(char *restrict dst, const char *restrict src, size_t sz)
{
    stpncpy(dst, src, sz);
    return dst;
}

char *
stpncpy(char *restrict dst, const char *restrict src, size_t sz)
{
    bzero(dst, sz);
    return mempcpy(dst, src, strnlen(src, sz));
}
```

上面有个比较不常见的函数 `mempcpy`。还是通过 man 手册，我们知道该还是基本等同于 `memcpy`，
只不过返回值指向了拷贝的数据的末尾的后一个字节的。这种返回值方便级联操作，不需要自己再执行类似
`dst += len` 的操作。

```man
The  mempcpy()  function  is nearly identical to the memcpy(3) function.  It copies n bytes from the object beginning at src into the object
pointed to by dest.  But instead of returning the value of dest it returns a pointer to the byte following the last written byte.
```

当然，面试的重点在于知道 `strncpy` 拷贝 `n` 个字节, 如果源字符串超过 `n` 字节，那么拷贝的内容是不会以 `\0` 结尾的。

但是，我想问一个问题： 如果源字符串中间包含 `\0`, 那么拷贝是否会在中间的 `\0` 结尾呢？

比如字符串 "Hello world!\0Welcome to the hell of strncpy!"

如果你认真阅读了上面的参考实现，那么你应该知道是会把中间的 `\0` 以及 `\0` 之后的字符给拷贝过去的。

另外一个问题是 `strncpy` 是否高效呢？ 其实因为有了 `bzero` 的操作，这个并不见得高效。比如你的 `dst` 空间是
10M， 你的 `src` 字符串一般情况下只有 100 字节左右，但是少数情况下可能很长。那么使用 strncpy 显然不是一个好的选择。

所以，面试官再提这种问题，就好好的教育教育他吧！

# 一个函数调用就生成正确的 C 字符串

上面提到，strncpy 在源字符串长度大于等于参数 n 的时候，拷贝的目标字符串是不带 \0 结尾的。那么如果想要更加优雅的方法实现一个函数调用就实现有什么办法呢？

1. 封装 strncpy函数，末尾加上 \0.
2. 使用 snprintf 函数。

通过 man 手册再查看一下 snprintf函数，重点关注末尾的 \0 和 返回值。

```text
$ man snprintf
int snprintf(char str[restrict .size], size_t size,
             const char *restrict format, ...);

int vsnprintf(char str[restrict .size], size_t size,
              const char *restrict format, va_list ap);
...
The functions snprintf() and vsnprintf() write at most size bytes (including the  terminating null byte ('\0')) to str.
...

The  functions  snprintf() and vsnprintf() do not write more than size bytes (including the
terminating null byte ('\0')).  If the output was truncated due to this limit, then the return
value is  the number of characters (excluding the terminating null byte) which would
have been written to the final string if enough space had been available. Thus, a  return
value of size or more means that the output was truncated.

If an output error is encountered, a negative value is returned.
```

从上面可以清楚的看到，snprintf() 一定会在字符串的末尾写入 \0.
但是如果你想知道写入的字符串是不是被截断了，那么得通过返回参数是否大于等于参数 `size` 来判断。

```C
int len = snprintf(buf, sizeof(buf), "%s", msg);
if (len < 0) {
    // performance error handle;
    return;
}

if (len >= sizeof(buf)) {
    // performance error handle;
    fprintf(stderr, "msg truncated");
    len = sizeof(buf) - 1;
}
```

如果不执行错误处理，那么在把 buf 和 len 往下传的过程中就会出错了。
后面的出入函数会出现使用了超过 `sizeof(buf)` 的数据, 比如将 len 长度写入文件。

我们再看看将多个数据拼接起来的情况。

可以看到这一版本的代码你能看出多少问题。

```C
// msgs is an array of string

int len = 0;
char buf[1024];

for (int i = 0; i < length; i++) {
    if (i > 0) {
        len += snprintf(" %s", msgs[i]);
    } else {
        len += snprintf("%s", msgs[i]);
    }
}
```

可以看到是没有对 len 是否超过总长度做错误处理的，因此增加一个返回值的处理。

```C
// msgs is an array of string

int len = 0;
char buf[1024];

for (int i = 0; i < length; i++) {
    if (i > 0) {
        len += snprintf(" %s", msgs[i]);
    } else {
        len += snprintf("%s", msgs[i]);
    }

    if (len >= sizeof(buf)) {
        len = sizeof(buf) - 1;
        break;
    }
}
```

上面还存在另一个错误就是没有对每一个单独的返回值做判断。
因此改成下面的代码：


```C
// msgs is an array of string

int len;
int total_len = 0;
char buf[1024];

for (int i = 0; i < length; i++) {
    if (i > 0) {
        len = snprintf(" %s", msgs[i]);
    } else {
        len = snprintf("%s", msgs[i]);
    }

    if (len < 0) {
        // maybe log here
        return;
    }

    total_len += len;
    if (total_len >= sizeof(buf)) {
        // maybe log here
        total_len = sizeof(buf) - 1;
        break;
    }
}
```

可以看到，要拷贝个字符串真的很不容易。snprintf 虽然自动写入了 \0, 但是判断错误也是非常麻烦的事情,并且snprintf 的性能比起strncpy来说也会更差，所以不是一个很好的选择。


上面的例子只是为了展示 snprintf，当然拷贝拼接还是可以使用 strncat, strlcpy, strlcat 等。
不过没有一个顺手的。

```text
// Catenate a null-padded character sequence into a string.
char *strncat(char *restrict dst, const char src[restrict .sz],
                      size_t sz);


char *
strncat(char *restrict dst, const char *restrict src, size_t sz)
{
    int   len;
    char  *p;

    len = strnlen(src, sz);
    p = dst + strlen(dst);
    p = mempcpy(p, src, len);
    *p = '\0';

    return dst;
}

// Copy/catenate a string with truncation.
size_t strlcpy(char dst[restrict .sz], const char *restrict src,
               size_t sz);
size_t strlcat(char dst[restrict .sz], const char *restrict src,
               size_t sz);

strlcpy(3bsd)
strlcat(3bsd)
       if (strlcpy(buf, "Hello ", sizeof(buf)) >= sizeof(buf))
           goto toolong;
       if (strlcat(buf, "world", sizeof(buf)) >= sizeof(buf))
           goto toolong;
       len = strlcat(buf, "!", sizeof(buf));
       if (len >= sizeof(buf))
           goto toolong;
       puts(buf);
```
