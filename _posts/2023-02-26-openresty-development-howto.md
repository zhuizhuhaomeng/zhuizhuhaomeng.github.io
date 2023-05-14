---
layout: post
title: "OpenResty 开发环境搭建"
description: "OpenResty 开发环境搭建"
date: 2023-02-26
tags: [OpenResty, 开发环境, How-To]
---

在开发 OpenResty 功能的时候需要能够本地执行所有的用例，因此就需要构建本地的开发环境。
如果都使用线上的 ci，那么经常会因为一些小问题需要提交很多次的代码。每次提交代码后等 CI
执行完成又需要很长的实际，这样就严重影响了开发效率。

本文基于 Rocky-8 环境搭建，不同的操作系统可能会有一点差异，需要自己调整。

# sudo 用户添加

推荐用普通用户开发 OpenResty，防止一个命令错误导致删除系统根目录或者其它重要的文件。
比如：

```shell
echo "zzhm       ALL=(ALL)       NOPASSWD: ALL" | sudo tee -a /etc/sudoers
```

# 安装软件

```shell
yum install -y git
yum install -y yum-utils
yum install -y epel-release
yum install -y ack
yum install -y gcc
yum install -y gcc-c++
yum install -y gd-devel
yum install -y redis
yum install -y memcached
yum install -y mysql-server
yum install -y wget
yum install -y autoconf 
yum install -y cmake
yum install -y automake
yum install -y libtool
yum install -y gettext-common-devel
yum install -y gettext-devel
yum install -y autoconf-archive
yum install -y txt2man
yum install -y openssl-devel
yum install -y clang
yum install -y patch
yum install -y python2
yum install -y perl-App-cpanminus
yum install -y ccache
yum install -y hg
yum install -y axel
yum install -y rpm-build
yum install -y lrzsz

wget -O axel.tar.gz https://github.com/axel-download-accelerator/axel/archive/v2.17.9.tar.gz
tar -xf axel.tar.gz
cd axel-2.17.9
autoreconf -i
./configure && make && make install
cd .. && rm -fr axel-2.17.9

#安装 Test::Nginx IPC::Run
cpanm --notest Test::Nginx IPC::Run
cpanm JSON::XS 
```

# 配置软件

这些配置是根据.travis.yml的如下位置代码获取的，随时间的变化可能有所变化。

https://github.com/openresty/lua-nginx-module/blob/master/.travis.yml#L56

https://github.com/openresty/lua-nginx-module/blob/master/.travis.yml#L95

https://github.com/openresty/lua-nginx-module/blob/master/.travis.yml#L98

```shell
systemctl enable redis
systemctl start redis
systemctl enable memcached
systemctl start memcached
systemctl enable mysqld
systemctl start mysqld

# travis上的创建命令失败，替换如下
mysql -uroot -e "drop database if exists ngx_test;use mysql;delete from user where User='ngx_test';flush privileges;"
mysql -uroot -e "create database ngx_test; CREATE USER 'ngx_test'@'%' IDENTIFIED BY 'ngx_test'; grant all on ngx_test.* to 'ngx_test'@'%'; flush privileges;"

iptables -I OUTPUT 1 -p udp --dport 10086 -j REJECT
iptables -I OUTPUT -p tcp --dst 127.0.0.2 --dport 12345 -j DROP
iptables -I OUTPUT -p udp --dst 127.0.0.2 --dport 12345 -j DROP

# 加入开机启动项
cat  >> /etc/rc.local << EOF
iptables -I OUTPUT 1 -p udp --dport 10086 -j REJECT
iptables -I OUTPUT -p tcp --dst 127.0.0.2 --dport 12345 -j DROP
iptables -I OUTPUT -p udp --dst 127.0.0.2 --dport 12345 -j DROP
EOF

# 如果 pid_max 太大会导致有些用例通不过，因此需要添加这个命令

echo "kernel.pid_max = 10000" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

# github设置

## 添加ssh的公钥

下载源码推荐使用ssh的方式，因此需要将公钥添加到github上。

```shell
ssh-keygen
cat ~/.ssh/id_rsa.pub
```

将上述公钥添加到 https://github.com/settings/keys 

如果是拷贝其它机器的密码，那么需要注意把私钥的权限修改为 600.也就是

```shell
chmod 600 ~/.ssh/id_rsa
```

# 下载源码

1. github fork分支到个人空间

   https://github.com/openresty/lua-nginx-module 右上角的Fork

2. clone代码到本地

   git clone git@github.com:zhuizhuhaomeng/lua-nginx-module.git

3. 本地创建分支

   git checkout -b feature_xxx

# 配置工具

下载到指定路径，并将这个库添加到/etc/bashrc的PATH环境变量。

``` shell
git clone git@github.com:openresty/openresty-devel-utils.git
```

# 编译软件

原始的编译步骤可以参考源码目录下的 .travis.yml https://github.com/openresty/lua-nginx-module/blob/master/.travis.yml。这里把文件整成一个 shell 脚本，脚本名称为 make_travis.sh

**注意**

1. 因为 orxray 依赖 python3。 为了防止残留导致后续开发遇到问题，下面脚本给python2 做软链，使用完后立即删除
2. 使用脚本需要注意，这个跟脚本并没有随 travis 更新，遇到问题需要排查一下。比如依赖的库是否齐全，版本号是否一致
3. 这里将依赖下载，依赖库编译，openresty 编译分成三个函数。开发中第一次全部执行，后续基本只需要执行 openresty 编译即可。
4. 因为 openresty 相关的库经常有更新，比如 lua-resty-core。因此不重新下载库的情况下最好经常更新相关的库。

```shell

mkdir -p download-cache

export CC=gcc
export JOBS=`nproc`
export NGX_BUILD_JOBS=$JOBS
export LUAJIT_PREFIX=/opt/luajit21
export LUAJIT_LIB=$LUAJIT_PREFIX/lib
export LUAJIT_INC=$LUAJIT_PREFIX/include/luajit-2.1
export LUA_INCLUDE_DIR=$LUAJIT_INC
export PCRE_VER=8.45
export PCRE_PREFIX=/opt/pcre
export PCRE_LIB=$PCRE_PREFIX/lib
export PCRE_INC=$PCRE_PREFIX/include
export OPENSSL_PREFIX=/opt/ssl
export OPENSSL_LIB=$OPENSSL_PREFIX/lib
export OPENSSL_INC=$OPENSSL_PREFIX/include
export LIBDRIZZLE_PREFIX=/opt/drizzle
export LIBDRIZZLE_INC=$LIBDRIZZLE_PREFIX/include/libdrizzle-1.0
export LIBDRIZZLE_LIB=$LIBDRIZZLE_PREFIX/lib
export LD_LIBRARY_PATH=$LUAJIT_LIB:$LD_LIBRARY_PATH
export DRIZZLE_VER=2011.07.21
export TEST_NGINX_SLEEP=0.006
export NGINX_VERSION=1.19.9
export OPENSSL_VER=1.1.1i
export OPENSSL_PATCH_VER=1.1.1f
sudo ln -s /usr/bin/python2 /usr/bin/python

function download()
{
   if [ ! -f download-cache/drizzle7-$DRIZZLE_VER.tar.gz ]; then wget -P download-cache http://openresty.org/download/drizzle7-$DRIZZLE_VER.tar.gz; fi
   if [ ! -f download-cache/pcre-$PCRE_VER.tar.gz ]; then wget -P download-cache https://ftp.pcre.org/pub/pcre/pcre-$PCRE_VER.tar.gz; fi
   if [ ! -f download-cache/openssl-$OPENSSL_VER.tar.gz ]; then wget -P download-cache https://www.openssl.org/source/openssl-$OPENSSL_VER.tar.gz || wget -P download-cache https://www.openssl.org/source/old/${OPENSSL_VER//[a-z]/}/openssl-$OPENSSL_VER.tar.gz; fi
   git clone https://github.com/openresty/test-nginx.git
   git clone https://github.com/openresty/openresty.git ../openresty
   git clone https://github.com/openresty/no-pool-nginx.git ../no-pool-nginx
   git clone https://github.com/openresty/openresty-devel-utils.git
   git clone https://github.com/openresty/mockeagain.git
   git clone https://github.com/openresty/lua-cjson.git lua-cjson
   git clone https://github.com/openresty/lua-upstream-nginx-module.git ../lua-upstream-nginx-module
   git clone https://github.com/openresty/echo-nginx-module.git ../echo-nginx-module
   git clone https://github.com/openresty/nginx-eval-module.git ../nginx-eval-module
   git clone https://github.com/simpl/ngx_devel_kit.git ../ndk-nginx-module
   git clone https://github.com/FRiCKLE/ngx_coolkit.git ../coolkit-nginx-module
   git clone https://github.com/openresty/headers-more-nginx-module.git ../headers-more-nginx-module
   git clone https://github.com/openresty/drizzle-nginx-module.git ../drizzle-nginx-module
   git clone https://github.com/openresty/set-misc-nginx-module.git ../set-misc-nginx-module
   git clone https://github.com/openresty/memc-nginx-module.git ../memc-nginx-module
   git clone https://github.com/openresty/rds-json-nginx-module.git ../rds-json-nginx-module
   git clone https://github.com/openresty/srcache-nginx-module.git ../srcache-nginx-module
   git clone https://github.com/openresty/redis2-nginx-module.git ../redis2-nginx-module
   git clone https://github.com/openresty/lua-resty-core.git ../lua-resty-core
   git clone https://github.com/openresty/lua-resty-lrucache.git ../lua-resty-lrucache
   git clone https://github.com/openresty/lua-resty-mysql.git ../lua-resty-mysql
   git clone https://github.com/openresty/lua-resty-string.git ../lua-resty-string
   git clone https://github.com/openresty/stream-lua-nginx-module.git ../stream-lua-nginx-module
   git clone -b v2.1-agentzh https://github.com/openresty/luajit2.git luajit2
}

function make_deps()
{
   cd luajit2/
   make -j$JOBS CCDEBUG=-g Q= PREFIX=$LUAJIT_PREFIX CC=$CC XCFLAGS='-DLUA_USE_APICHECK -DLUA_USE_ASSERT -msse4.2'
   sudo make install PREFIX=$LUAJIT_PREFIX
   cd ..
   tar xzf download-cache/drizzle7-$DRIZZLE_VER.tar.gz && cd drizzle7-$DRIZZLE_VER
   ./configure --prefix=$LIBDRIZZLE_PREFIX --without-server

   make libdrizzle-1.0 -j$JOBS
   sudo make install-libdrizzle-1.0
   cd ../mockeagain/ && make CC=$CC -j$JOBS && cd ..
   cd lua-cjson/ && make -j$JOBS && sudo make install && cd ..
   tar zxf download-cache/pcre-$PCRE_VER.tar.gz
   cd pcre-$PCRE_VER/
   ./configure --prefix=$PCRE_PREFIX --enable-jit --enable-utf --enable-unicode-properties
   make -j$JOBS
   sudo PATH=$PATH make install
   cd ..
   tar zxf download-cache/openssl-$OPENSSL_VER.tar.gz
   cd openssl-$OPENSSL_VER/
   patch -p1 < ../../openresty/patches/openssl-$OPENSSL_PATCH_VER-sess_set_get_cb_yield.patch
   ./config shared enable-ssl3 enable-ssl3-method -g --prefix=$OPENSSL_PREFIX -DPURIFY
   make -j$JOBS
   sudo make PATH=$PATH install_sw
   cd ..
}


export PATH=$PWD/work/nginx/sbin:$PWD/openresty-devel-utils:$PATH
export NGX_BUILD_CC=$CC

function make_ngx()
{
    rm -fr buildroot
    sh util/build.sh $NGINX_VERSION 2>&1 | tee build.log
}

download
make_deps
make_ngx

find t -name "*.t" | xargs reindex >/dev/null 2>&1
sudo rm  /usr/bin/python

reindex  t/*.t 2>&1 | grep -v skipped
ngx-releng
#nginx -V
ldd `which nginx`|grep -E 'luajit|ssl|pcre'
export LD_PRELOAD=$PWD/mockeagain/mockeagain.so
export LD_LIBRARY_PATH=$PWD/mockeagain:$LD_LIBRARY_PATH
export TEST_NGINX_RESOLVER=8.8.4.4
#export TEST_NGINX_USE_VALGRIND=1
#export TEST_NGINX_NO_CLEAN=1
#export TEST_NGINX_CHECK_LEAK=1
#export TEST_NGINX_CHECK_LEAK_COUNT=1000

#export TEST_NGINX_RANDOMIZE=1

#export TEST_NGINX_EVENT_TYPE=poll
#export TEST_NGINX_POSTPONE_OUTPUT=1
#export MOCKEAGAIN=w
##export MOCKEAGAIN=rw
#export MOCKEAGAIN_VERBOSE=1

#dig +short myip.opendns.com @resolver1.opendns.com || exit 0
#dig +short @$TEST_NGINX_RESOLVER openresty.org || exit 0
#dig +short @$TEST_NGINX_RESOLVER agentzh.org || exit 0

#prove -j6 -I. -Itest-nginx/lib t/

#prove -I. -Itest-nginx/lib t/166-ssl-client-hello.t


# 用 Test::Nginx::Socket  测试台跑用例时，可以通过下面这个环境变量来开启“每语句 full GC 模式“:
# export TEST_NGINX_INIT_BY_LUA="debug.sethook(function () collectgarbage() end, 'l') jit.off() package.path = '/usr/share/lua/5.1/?.lua;$PWD/../lua-resty-core/lib/?.lua;$PWD/../lua-resty-lrucache/lib/?.lua;' .. (package.path or '') require 'resty.core' require('resty.core.base').set_string_buf_size(1) require('resty.core.regex').set_buf_grow_ratio(1)"
```

上述脚本有加入代码风格检查，因为检查到问题并没有让脚本退出执行。所以要注意脚本的输出结果，如果检查到问题要立即修改。代码风格不能够依靠这个检查脚本，这个脚本是帮助大家养成符合nginx风格的习惯。

# 软件测试

软件测试可以把 make_travis.sh 中的编译相关的三个函数注释掉，这样就直接跑测试了。

不能直接手动执行 prove，因为脚本里头有设置环境变量，直接跑使用的 nginx 就不是当面目录下的 work/nginx/sbin/nginx。

## 测试调试

### 单独执行用例

如果某个用例通不过，需要单独跑一个用例，可以添加 --- ONLY单独跑指定的用例。

用例执行相关的结果存在t/servroot/目录下。比如

```shell
$ ls t/servroot/
client_body_temp  conf  fastcgi_temp  html  logs  proxy_temp  scgi_temp  uwsgi_temp
```

想要查看生成的配置文件，错误日志等可以在这里查看。

### 用例测试个数调整

在测试文件的最开头会有测试个数的计算，最终的测试个数应该和计算的结果是一致的。

```perl
   # vim:set ft= ts=4 sw=4 et fdm=marker:
   use t::TestJsonb;
   
   repeat_each(1);
   
   plan tests => repeat_each() * (blocks() * 3);

```

出现如下错误，就是用例个数对不上。预期应该是3的倍数，基本可以判断是某个用例有4个测试判断，所以变成了40个。因此上面的paln tests 可以修改成repeat_each() * (blocks() * 3 + 1)。 

当然最好是每个用例保持3个测试判断比较合适。

```shell
t/iter.t .. 29/39 # Looks like you planned 39 tests but ran 40.
t/iter.t .. Dubious, test returned 255 (wstat 65280, 0xff00)
All 39 subtests passed 

Test Summary Report
-------------------
t/iter.t (Wstat: 65280 Tests: 40 Failed: 1)
  Failed test:  40
  Non-zero exit status: 255
  Parse errors: Bad plan.  You planned 39 tests but ran 40.
```

一般情况下都是一个指令一个判断，但是对于--- response_headers，则是一个 header是一个判断。比如下下面的测试指令的测试计数为3。

```text
--- response_body
 arr1: 3
 arr2: 4
 --- response_headers
 Content-Type: text/plain
 Content-Length: 20
```

### 测试结束不退出

export TEST_NGINX_NO_CLEAN=1

执行prove后，nginx不会被关闭。

## 相关资料

介绍Test::Nginx这个测试框架，各种各样的测试模式。

https://openresty.gitbooks.io/programming-openresty/content/



测试框架的函数说明可以在这里查找。

https://metacpan.org/pod/Test::Nginx::Socket

# 提交PR

## 保存本地的修改

1. 本地修改和保存

   git add path_to_file

   git commit -m "feature: add FFI interface to verify SSL client certificate"

## 提交PR

1. 将代码更新到最新

``` shell
git checkout master
git remote add op git@github.com:openresty/lua-nginx-module.git
git pull op master
git checkout feature_xxx
git rebase master
```

2. 更新到自己的github空间

   ``` shell
   git push self feature_xxx
   ```

3. 创建pr

   根据上一个步骤给的提示连接登录github创建pr。

## git 工作流

https://openresty.org/en/git-workflow.html

主要是commit的日志怎么写，需要符合这个规范。

# 编码风格

https://openresty.org/en/c-coding-style-guide.html

## 编码风格检查

每次提交代码前都应该使用这两个工具检查一下

https://github.com/openresty/openresty-devel-utils.git

ngx-releng 检查openresty 相关的C代码（文件名称需要以ngx_开头）

lua-releng  检查openresty 相关的lua代码



因为ngx-releng只检查ngx开头的代码文件，因此如果不是openresty相关的工程，但是又要符合openresty的规范，那么可以用ngx-style.pl来检查,不过比ngx-releng少检查一点。

# 代码阅读

## C 和 lua 接口查找

目前的 C 和 lua 交互一部分是通过注册 C 函数到 Lua，一部分是通过 ffi 接口调用。

因此如果想了解相关的接口，可以通过搜索 ffi 关键字和 lua_pushcfunction。另一部分常数的注入可以通过搜索 ngx_http_lua_inject 关键字。

比如：

``` shell
[ljl@localhost lua-nginx-module]$ find src -name "*.c" | xargs grep -n lua_pushcfunction
src/ngx_http_lua_api.c:56:        lua_pushcfunction(L, func);
src/ngx_http_lua_args.c:397:    lua_pushcfunction(L, ngx_http_lua_ngx_req_set_uri_args);
src/ngx_http_lua_args.c:400:    lua_pushcfunction(L, ngx_http_lua_ngx_req_get_post_args);
src/ngx_http_lua_balancer.c:370:    lua_pushcfunction(L, ngx_http_lua_traceback);
src/ngx_http_lua_bodyfilterby.c:100:    lua_pushcfunction(L, ngx_http_lua_traceback);
....
```

# 其它学习笔记推荐

这个学习笔记是我的一个同事做的，里面记录了很多大家都会遇到的疑难问题。

https://github.com/isshe/coding-life/tree/master/K.%E5%B7%A5%E5%85%B7/OpenResty
