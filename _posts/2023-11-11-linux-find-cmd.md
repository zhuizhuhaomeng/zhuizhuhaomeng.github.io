---
layout: post
title: "Examples of Linux find command"
description: "Example of Linux find command"
date: 2023-11-11
tags: [find]
---

```shell
#!/bin/bash

for name in `find /  -maxdepth 1 -type d -regex "/[0-9]+t-[a-z]"`; do
    for d in `find /16t-a/trace-pkg-db -maxdepth 2 -type d -regex ".*/\(rpms\|debs\)"`; do
        find $d -type f -regex  ".*/\(kernel\|linux\).*"
    done
done
```

