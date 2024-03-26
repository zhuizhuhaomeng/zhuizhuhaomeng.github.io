---
layout: post
title: "试用 git pre-commit hooks 做好代码提交前的检查"
description: "不想再犯错，配置好 git 的检查钩子"
date: 2024-03-26
tags: [git, hooks]
---

每个公司甚至小组都有各种规范，但是有时候又是那么的容易忘记。
这时候试用 git 自带的 commit hooks 可以强制检查，拒绝二次犯错。

比如这个是我的 pre-commit 的钩子

```shell
#!/bin/sh

find t -name "*.t" | xargs grep -E -- "(--- ONLY|--- LAST)" > check.log
if [ -s check.log ]; then
    cat check.log
    rm check.log
    echo -e "\033[31mPlease remove --- (ONLY|LAST)\033[0m"
    exit 1
fi

rm check.log
exit 0
```

把上面的代码保存成文件 pre-commit 放在 .git/hooks 目录下。
