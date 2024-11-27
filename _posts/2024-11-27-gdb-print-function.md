---
layout: post
title: "Print function name when entering a function using GDB"
description: "print function name when entering a function using GDB"
date: 2024-11-27
tags: [GDB]
---

Reading code is a very important skill for software developers.
Knowing the calling sequence of a function is essential for understanding.
GDB rbreak command is a powerful tool for code analysis.

# rbreak

1. Add break point on function names start with ngx_http
2. print function using bt 1

Example of the GDB commands

```
rbreak ^ngx_http
commands
silent
bt 1
continue
end
```
