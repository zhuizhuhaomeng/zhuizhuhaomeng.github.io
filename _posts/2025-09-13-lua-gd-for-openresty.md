---
layout: post
title: "为 OpenResty 编译 lua-gd"
description: "build lua-gd for OpenResty"
date: 2025-09-13
modified: 2025-09-13
tags: [OpenResty]
---

# 背景说明

lua-gd 也需要链接 luajit 才能被 OpenResty 使用，因此 luarocks 也需要重新编译。

# 编译 luarocks

```shell
sudo yum install patch gcc autoconf automake git wget
wget https://luarocks.github.io/luarocks/releases/luarocks-3.12.2.tar.gz

tar -xf luarocks-3.12.2.tar.gz
cd luarocks-3.12.2
./configure --prefix=/usr/local/openresty/luajit \
    --with-lua=/usr/local/openresty/luajit/ \
    --lua-suffix=jit \
    --with-lua-include=/usr/local/openresty/luajit/include/luajit-2.1
make
sudo make install
```

# 编译 lua-gd

## lua-gd patch 

将这个 patch 保存到当前目录，名称为 lua-gd.patch

```patch
diff --git a/Makefile b/Makefile
index 21f1d6f..34a01df 100644
--- a/Makefile
+++ b/Makefile
@@ -29,7 +29,7 @@
 VERSION=2.0.33r3
 
 # Command used to run Lua code
-LUABIN=lua5.1
+LUABIN=luajit
 
 # Optimization for the brave of heart ;)
 OMITFP=-fomit-frame-pointer
@@ -41,15 +41,16 @@ OMITFP=-fomit-frame-pointer
 # comment out these lines and uncomment and change the next ones.
 
 # Name of .pc file. "lua5.1" on Debian/Ubuntu
-LUAPKG=lua5.1
+LUAPKG=luajit
 OUTFILE=gd.so
 
-CFLAGS=-O3 -Wall -fPIC $(OMITFP)
+CFLAGS=-O3 -Wall -fPIC $(OMITFP) -I/usr/local/openresty/luajit/include/luajit-2.1
 CFLAGS+=`pkg-config $(LUAPKG) --cflags`
 CFLAGS+=-DVERSION=\"$(VERSION)\"
 
 GDFEATURES=-DGD_XPM -DGD_JPEG -DGD_FONTCONFIG -DGD_FREETYPE -DGD_PNG -DGD_GIF
-LFLAGS=-shared `pkg-config $(LUAPKG) --libs` -lgd
+LFLAGS=-shared `pkg-config $(LUAPKG) --libs` -Wl,-rpath,/usr/local/openresty/luajit/lib -lgd -L/usr/local/openresty/luajit/lib -lluajit-5.1
+
 
 INSTALL_PATH := `pkg-config $(LUAPKG) --variable=INSTALL_CMOD`
 
```

## 编译 lua-gd

```shell
set -e
git clone https://github.com/ittner/lua-gd.git
cd lua-gd
cat ../lua-gd.patch | patch -p1
/usr/local/openresty/luajit/bin/luarocks build
```
编译完成后 gd.so 位于 /usr/local/openresty/luajit/lib/lua/5.1/gd.so
