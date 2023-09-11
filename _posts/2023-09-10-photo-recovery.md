---
layout: post
title: "U 盘照片恢复"
description: "U 盘照片恢复"
date: 2023-09-09
tags: [photorec, picture]
---

U 盘出问题了，U 盘上的照片读不出来，心里那个郁闷啊。
好在有万能的 Linux 可以帮忙。仅仅用下面的命令就将 U 盘上的照片恢复了。
注意，如果你也要恢复文件，需要将 /dev/sdc 替换为你实际的 U 盘设备。

```shell
sudo apt-get install testdisk

cd ~
mkdir recup_dir
sudo photorec /d recup_dir /dev/sdc
```

上面的 U 盘数据虽然恢复了，但是文件命名没有规律，不知道是什么时候拍摄的。
我就写了下面这段小脚本来将文件重命名为拍摄时间， 比如 2018-04-14-856847.jpg。

```perl
#!/usr/bin/perl

use strict;
use warnings;
use File::Find;
use File::Basename;

my $directory = './';

find(\&process_file, $directory);

sub process_file {
    my $file = $_;

    if (-f $file && $file =~ /\.jpe?g$/i) {
        my $output = `exiftool -DateTimeOriginal -d '%Y-%m-%d' -SubSecTimeOriginal "$file"`;

        my ($datetime,$subsecond);
        if ($output =~ /.*Date\/Time Original\s+:\s*([-0-9]+).*/ms) {
            $datetime = $1;
        } else {
            return;
        }

        if ($output =~ /.*Sub Sec Time Original\s+:\s*([-0-9]+).*/ms) {
            $subsecond = $1;
        } else {
            return;
        }

        my $nfile = "$datetime-$subsecond.jpg";
        print "$file to $nfile\n";
        if (! -f $nfile) {
            rename $file, $nfile;
        }
    }
}
```

将上面的代码报文为 rename-photo.pl, 保存在恢复出来的照片的目录 recup_dir 里面，然后执行 `perl rename-photo.pl`就大功告成了。
