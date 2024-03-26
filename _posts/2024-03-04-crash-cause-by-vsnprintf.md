---
layout: post
title: "crash caused by vsprintf"
description: "crash caused by vsprintf"
date: 2024-03-04
tags: [vsnprintf, crash, coredump]
---

我们总会有这样的印象：平常的测试都好好的，该写的用例也都写了，但是在上线前的最后一刻就出问题了，好像是总有一个恶魔在跟我们作对。

这次遇到的是一个进程崩溃的问题 (`Segmentation fault (core dumped)`)，而代码却看起来一点问题也没有，百思不得其解。就是下面的一小段代码，你能找打原因吗？


```CXX
#ifndef MAXSTRINGLEN
#define MAXSTRINGLEN 8192
#endif


std::string
cxx_sprintf(const char *fmt, ...) 
{
    char buf[MAXSTRINGLEN];
    va_list ap;

    va_start(ap, fmt);

    int n = vsnprintf(buf, sizeof(buf), fmt, ap); 

    va_end(ap);

    if (n < 0) { 
        throw std::runtime_error(string("invalid fmt string: ") + string(fmt));
    }    

    return std::string(buf, n);
}
```

乍一看上面的代码错误处理也做了，从 `char []` 转换为 `string` 还限制了字符串的长度了，怎么会崩溃呢？

其实是以为上面的错误处理并不完善， `vsnprintf` 的返回值大有文章，这个一不小心就容易掉坑里。
开源代码中大部分应该也是没有处理的，这就容易照猫画虎，以讹传讹。
当然有些情况其实也没有处理的必要，比如将数值转换为字符串，在 `buf` 足够大，`fmt` 参数正确的情况下。

我们看看 `vsnprintf`到底返回了什么？

```text
RETURN VALUE
       Upon successful return, these functions return the number of characters printed (excluding the null  byte  used  to  end
       output to strings).

       The functions snprintf() and vsnprintf() do not write more than size bytes (including the terminating null byte ('\0')).
       If the output was truncated due to this limit, then the return value is the number of characters (excluding  the  termi‐
       nating  null byte) which would have been written to the final string if enough space had been available.  Thus, a return
       value of size or more means that the output was truncated.  (See also below under NOTES.)

       If an output error is encountered, a negative value is returned.
```

这里的重点就是:

 “If the output was truncated due to this limit, then the return
 value is the number of characters (excluding  the  termi‐nating  null byte) 
 which would have been written to the final string if enough space had been available. ”

也就是说，返回值是这个字符串格式化的实际长度，不受缓存大小的影响。为什么这个接口要怎么设计呢？
这是为了能够在缓存不足的情况下，重新申请一个更大的缓存来重现格式化字符串。
这样就不必提供两个接口，一个用于获取实际的长度，一个用于格式化字符串。

因此上面的代码应该修改成：

```CXX
std::string
cxx_sprintf(const char *fmt, ...) 
{
    char buf[MAXSTRINGLEN];
    va_list ap;

    va_start(ap, fmt);

    int n = vsnprintf(buf, sizeof(buf), fmt, ap); 

    va_end(ap);

    if (n < 0) { 
        throw std::runtime_error(string("invalid fmt string: ") + string(fmt));
    }    

    if (n >= sizeof(buf)) {
        n = sizeof(buf) - 1;
    }

    return std::string(buf, n);
}
```

注意上面的判断条件是 `>=` 而不是 `>`。这是因为 `vsnprintf` 总是会在字符串的末尾增加一个 `\0`。
这样实际上可用的空间就只有  `sizeof(buf) - 1`。

至于为什么会导致进程崩溃，我想这个可能是访问内存越界，访问到了没有加载的内存页导致的。

比如上面的代码 `buff` 是在栈上的，那么如果访问超过了栈的最大值，那么就会访问到未分配的内存页。
像下面的 stack 的访问是 `7ffffffde000-7ffffffff000` 的范围。如果读取该范围外的内存将会导致 `segment fault`。

```shell
$ sudo cat /proc/5756/maps
555555554000-555555560000 r--p 00000000 fd:00 67537410                   /usr/sbin/sshd
555555560000-5555555ef000 r-xp 0000c000 fd:00 67537410                   /usr/sbin/sshd
5555555ef000-55555563a000 r--p 0009b000 fd:00 67537410                   /usr/sbin/sshd
55555563a000-55555563e000 r--p 000e5000 fd:00 67537410                   /usr/sbin/sshd
55555563e000-55555563f000 rw-p 000e9000 fd:00 67537410                   /usr/sbin/sshd
55555563f000-55555570e000 rw-p 00000000 00:00 0                          [heap]
7ffff6f1d000-7ffff6f24000 r--p 00000000 fd:00 67466762                   /usr/lib64/libnss_systemd.so.2
7ffff6f24000-7ffff6f5c000 r-xp 00007000 fd:00 67466762                   /usr/lib64/libnss_systemd.so.2
7ffff6f5c000-7ffff6f6e000 r--p 0003f000 fd:00 67466762                   /usr/lib64/libnss_systemd.so.2
...
7ffff7fc7000-7ffff7fc9000 r--p 00000000 fd:00 67442267                   /usr/lib64/ld-linux-x86-64.so.2
7ffff7fc9000-7ffff7fef000 r-xp 00002000 fd:00 67442267                   /usr/lib64/ld-linux-x86-64.so.2
7ffff7fef000-7ffff7ffa000 r--p 00028000 fd:00 67442267                   /usr/lib64/ld-linux-x86-64.so.2
7ffff7ffb000-7ffff7ffd000 r--p 00033000 fd:00 67442267                   /usr/lib64/ld-linux-x86-64.so.2
7ffff7ffd000-7ffff7fff000 rw-p 00035000 fd:00 67442267                   /usr/lib64/ld-linux-x86-64.so.2
7ffffffde000-7ffffffff000 rw-p 00000000 00:00 0                          [stack]
ffffffffff600000-ffffffffff601000 --xp 00000000 00:00 0                  [vsyscall]
```
