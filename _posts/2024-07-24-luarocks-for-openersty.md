---
layout: post
title: "luarocks for openresty"
description: "luarocks for openresty"
date: 2024-07-23
tags: [luarocks,openresty]
---

luarocks 安装模块给 OpenResty使用，最好是要跟非 LuaJIT 的版本区分，因此应该使用类似这样子的命令

```shell
sudo luarocks install --force --lua-dir=/usr/local/openresty/luajit/ --tree=/usr/local/openresty/luajit --server=https://luarocks.org/manifests/zhuizhuhaomeng  luafilesystem
```

使用上述命令安装后的，lfs.so 在 /usr/local/openresty/luajit/lib64/lua/5.1/lfs.so。
因为上面的文件所在的目录不是 openresty 的默认搜索目录，因此还要将目录添加到 nginx.conf 文件中。

```conf
lua_package_cpath "/usr/local/openresty/luajit/lib64/lua/5.1/?.so;;";
```

验证一下加载的 lfs.so

```shell
$ sudo systemctl restart openresty
$ ps aux | grep "nginx: worker"
nobody    169365  0.0  0.0 152272  9496 ?        Sl   13:08   0:00 nginx: worker process
nobody    169366  0.0  0.0 152244  8728 ?        Sl   13:08   0:00 nginx: worker process
ljl       169413  0.0  0.0 221800  2112 pts/3    S+   13:08   0:00 grep --color=auto --exclude-dir=.bzr --exclude-dir=CVS --exclude-dir=.git --exclude-dir=.hg --exclude-dir=.svn --exclude-dir=.idea --exclude-dir=.tox nginx: worker
$ sudo cat /proc/169365/maps | grep lfs
7ffff7cec000-7ffff7cee000 r--p 00000000 fd:00 68090374                   /usr/local/openresty/luajit/lib64/lua/5.1/lfs.so
7ffff7cee000-7ffff7cf0000 r-xp 00002000 fd:00 68090374                   /usr/local/openresty/luajit/lib64/lua/5.1/lfs.so
7ffff7cf0000-7ffff7cf1000 r--p 00004000 fd:00 68090374                   /usr/local/openresty/luajit/lib64/lua/5.1/lfs.so
7ffff7cf1000-7ffff7cf2000 r--p 00004000 fd:00 68090374                   /usr/local/openresty/luajit/lib64/lua/5.1/lfs.so
7ffff7cf2000-7ffff7cf3000 rw-p 00005000 fd:00 68090374                   /usr/local/openresty/luajit/lib64/lua/5.1/lfs.so
```

完美！！
