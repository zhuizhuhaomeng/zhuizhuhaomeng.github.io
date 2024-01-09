---
layout: post
title: "Perl notes"
description: "Perl study notes"
date: 2024-01-07
tags: [Perl]
---

# 文件开头

添加 strict 和 warnings 的检查，减少不必要的出错。

```perl
#!/usr/bin/env perl

use v5.10.1;
use strict;
use warnings;
#no warnings 'recursion';
#no warnings 'portable';
```


# 正则表达式

1.  \U 将所有匹配转换为大写字母

    ```perl
    s/(abc)/\U$1/g
    ```

1. 去除空格

    ```perl
    my $t = " abc ";
    $t =~ s/(^\s+|\s+$)//g;
    ```

1. 模式分组

    ```perl
    my $t = "abba";
    if ($t =~ /(.)(.)\2\1/) {
        print "match\n";
    }

    可以使用 g{idx} 来替代 \1 这种形式。

    my $t = "abba";
    if ($t =~ /(.)(.)\g{2}\g{1}/) {
        print "match\n";
    }
    ```

1. 匹配定界符

    ```perl
    my $txt = "https://www.badiu.com";
    if ($text =~ m#https://\w+\w+\w+#) {
       print "match\n";
    }
    ```
    这里选择 # 作为定界符，可以防止和模式字符串里面的 // 冲突。

1. 其它修复符号

    - i: 大小写无关匹配
    - s: .号可以匹配换行
    - x: 让正则表达式里面可以加上任意的空格，可以让正则表达式看起来更清晰。注意在这两种情况下，如果要匹配空格需要使用\s.

1. 锚位

    - ^: 匹配行首
    - $: 匹配行尾
    - \b: 匹配单词边界

1. 模式内插

    ```perl
    my $word = "larry";

    while (<>) {
        if (/($word)/) {
            print;
        }
    }
    ```

    这种内插需要注意内插变量是否存在元字符。可以用下面的形式将元字符替换。

    ```perl
    my $word = "abc(def";
    $txt = "abc(";
    if ($txt =~ /(\Q$word\E)/) {
        print "match\n";
    }
    ```

1. 命名分组

    ```perl
    my $word = "larry";

    while (<>) {
        if (/(?<name1>\w+)/) {
            print;
        }
    }
    ```

    如果有多个分组，搞不清楚对应那个分组，或者模式修改在中间插入分组，那么使用命令分组可以减少代码的修改。


1. 自动匹配变量
    - $&: 正则表达式匹配的部分
    - $`: 正则表达式匹配之前的部分
    - $': 正则表达式匹配后续的部分



# 算术运算

1. 求模运算

    ```perl
    10 % 3 == 1
    10.3 % 3.9 == 1
    ```
    这是因为求模运算会先转换成整数在计算。

1. 幂运算

    ```shell
    2**3 == 8
    ```
    幂运算使用两个星号

1. 其它

# 字符串

1. 单引号字符串

    1.1 perl 不会对 单引号字符串做转义，同时 perl 的单引号字符串可以包含换行符。

    ```perl
    my $a = '
    a
    d
    ';
    print($a);
    ```

    1.2 perl 的单引号字符串对反斜杠和单引号做转义

    ```perl
    my $a = 'I\'am \\ \n.';
    print($a);
    ```

    可以看到，上面的 `\n` 不会被转义

1. 双引号字符串

    双引号字符串和 C 语言的字符串转义是一样的。当然，`\l, \L, \u, \U, \Q， \E` 这几个需要特别注意。

    `\l, \u` 分别将下一个字母转换为小写，大写。`\L,\U` 将接下来的字符转换为小写，大写知道遇到 \E。

    而 `\Q` 是将非单词字符添加反斜杠直到遇到  \E。

    ```perl
    print("\lA\n");  #a
    print("\ua\n");  #A
    print("\LABC\n"); #abc
    print("\LABC\EABC\n"); #abcABC
    print("\Uabc\n");  #ABC
    print("\Uabc\Eabc\n"); #ABCabc
    print("\Qabc, abc\E\n"); #abc\,\ abc
    ```

1. 字符串拼接

    Perl 使用的是 `.`拼接字符串，而 Lua 则使用 `..`,  其它更多的语言使用运算符重载 `+`。

     如果要让字符串重复， perl 使用的是 x 字母。比如 `my $a = "a" x 10;`

1. 数字和字符串自动转换

    ```perl
    my $a = "z" . 32 * 2; # 得到的是 "z64"
    ```

# 比较运算

1. Perl 没有专门的布尔值。其中 0, "0", "" ，undef 是假值，其它非标量值转换为数值再进行判断。比如空的列表，哈希表。

# 读取数据

```perl
my $line = <STDIN>;  # 从终端读取一行，一般会有换行符，因此要把换行符去掉，用chomp。注意，chomp 只会移除行尾的一个换行符，如果有多个需要多次调用。
chomp $line;

#or
chomp($line = <STDIN>);

# 如果要读取说有的行，可以使用

my @lines = <STDIN>;  #列表上下文会自动将所有行读取存入列表

#上面的行包含换行符，因此可以这样去掉换行符

chomp(my @lines = <STDIN>);
```

#  列表

## qw 列表

```perl
qw (abc def hello world)
qw /abc def hello world/
qw #abc def hello world#
qw {abc def hello world}
qw !abc def hello world!
qw [abc def hello world]
qw <abc def hello world>

qw {/usr/local/
    /opt/
    /var
}
```

## 数值列表

```perl
my @arr = 1..6;
print @arr, "\n";
print "@arr\n";
```

注意，上面的内存会在数组元素之间添加空格分隔符。



# 变量内插

变量内插的时候，经常会遇到 $@这些运算符导致的问题。如果存在需要区分变量，可以首映大括号做分割。比如 `"${var}s"` 内插 了 $var 跟着一个 s。

如果不像内插，那么应该做转义，比如 `"\$var"`。



# 变量上下文

要的是标量上下文还是列表上下文，这个对于理解 perl 代码很重要。

有时候需要明确是标量上下文，可以使用 `scalar` 得到列表元素的格式。

```perl
my @rocks = (talc quartz jade);
print "I have", scalar @rocks, "rocks!\n";

# 这个是标量上下文，因此每次读入一行
while (defined ($_ = <STDIN>)) {
    print $_;
}

# 这个是列表上下文，因此读取整个文件
foreach (<STDIN>) {
    print $_;
}
```

# 变量声明

```perl
my $a, $b;  # 声明变量 a
my ($a, $b) # 声明变量 a, b。所有这个括号千万不要丢了。
```



# 文件操作

- chmod
- chown
- close
- closedir
- link
- mkdir
- open
- opendir
- rename
- rmdir
- stat
- symlink
- unlink
- utime: 修改文件的时间戳

## 开关读写文件

```perl
# 打开文件
open my $config, "file.txt" or die "can not open file: $!"; # open for reading
open my $config, "<file.txt" or die "can not open file: $!"; # open for reading
open my $config, ">file.txt" or die "can not open file: $!"; # open for writing
open my $config, ">>file.txt" or die "can not open file: $!"; # open for appending

# 关闭文件
close $config;

# 读取文件
binmode $in;
my $json = do { local $/; <$in> };

# 写入文件。 注意文件句柄和输出内容之间没有空格
print $out YAML::XS::Dump($units);

# 默认打印的到 stdout
print "hello world"

# 打印到文件句柄 $in
select $in;
print "hello world";
```

## 获取目录下的文件

```perl
# get all files end with .txt
my @files = glob "*.txt";

# or
# get all files end with .png or .jpeg
my @files = glob "*.png *.jpeg";

```

使用更高效的方式，不需要另外启动子进程

```perl
my $dir_to_process = "/etc";
opendir my $dh, $dir_to_process or die "Cannot open $dir_to_process: $!";
foreach $file (readdir $dh) {
    next if ($file eq "." or $file eq "..");
    print "one file in $dir_to_process is $file";
    my $file = "$dir_to_porcess/$file";
    # handle file
}

closedir($dh);
```

## 删除文件

```perl
unlink "abc.txt" or warn "Cannot remove abc.txt: $!";

#or

unlink glob "*.o";
```

## 重命名文件

```perl
rename "old", "new" or "Cannot rename odl to new: $!";
```

Examples:

```perl
#!/usr/bin/evn perl

# rename *.jpeg to *.jpg

for my $file (glob "*.jpeg") {
    my $newfile = $file;
    $newfile =~ s/\.jpeg$/.jpg/;

    #my ($newfile = $file) =~ s/\.jpeg$/.jpg/;

    if (-e $newfile) {
        warn "can't renmae $file to $newfile: $newfile exists\n";
    } elsif (rename $file, $newfile) {
        # successful, do nothing
    } else {
        warn "rename $file to $newfile failed: $!\n"
    }
}
```

# 调试

```perl
warn "error here";  # 这个会加上 warn 语句所在的行号
warn "error here\n"; # 这个会加上 warn 语句的行号

# die 也是类似的


# die 的时候会打印调用栈
use Devel::Confess;

# 打印调用栈
use Carp;


# 打印复杂的数据，比如 hash
sub Dumper ($) {
    require Data::Dumper;
    if ($Data::Dumper::Sortkeys != 1) {
        $Data::Dumper::Sortkeys = 1;
    }

    Data::Dumper::Dumper(@_);
}
```

# 哈希表

## 构造哈希表

```perl
my %h = ("a", "b", "c", "d");
# 等价于
my %h = ("a" => "b", "c" => "d",);
```

注意， 哈希表的构造用的是小括号。

## 遍历哈希表

```perl
my %h = ("a" => "b", "c" => "d",);
my @k = keys %h;
my @v = values %h;

# 方法 1
while (my ($k, $v) = each %h) {
    print "$k => $v,\n"
}

# 方法 2

for my $k (sort keys %h) {
    print "$k => $h{$k}\n"
}
```

## 判断哈希表的指定元素是否存在

```perl
if (exists $h{$key}) {
    print "exist\n";
}
```

## 删除哈希表原始

```perl
delete $h{$key};
```

# perl 控制结构

perl 中断循环使用的是 last， 而不是 C 语言的 break;

```perl
while (1) {
    my $line = <STDIN>;
    if ($line = /^$/) {
       last;
    }

    print $line;
}
```

perl 执行下一次循环是用 next，不是 continue；


```perl
while (1) {
    my $line = <STDIN>;
    if ($line = /^$/) {
       continue;
    }

    print $line;
}
```

# perl 模块

## 模块的编译安装

```shell
perl Makefile.PL
make install

# or

perl Makefile.PL PREFIX=/usr/local/module-name
make install

#or

perl Build.PL
./Build install
```

## cpan 安装

```shell
cpan GCI::Prototype
```

## 导入模块

```perl
# 只导入自己需要的函数
use File::Basename qw/ basename /;
```
