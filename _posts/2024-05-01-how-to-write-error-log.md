---
layout: post
title: "How to write error log when checking the arguments?"
description: "The error log in Lua for checking arguments"
date: 2024-05-1
tags: [OpenResty, Errorlog, Lua ]
---

A good error log should help you to resolve the problem.
Or give you enough information to find the cause of the problem.

For the error log of the Lua code, I think it should at least include the filename and line number of the code,
the function name trigger the error and the argument trigger the error.

The openresty's `ngx.log` will add the file name and the linenumber by default.

For example this is a error log in openresty.

```errorlog
2023/01/04 10:32:20 [error] 5087#5087: *3097 [lua] content_by_lua(nginx.conf:393):3: log message here, client: 127.0.0.1, server: , request: "GET /test-error-log HTTP/1.1", host: "localhost"
```

But here, my greater concern is how programmers should handle returning error messages when checking function parameters.

This is a error message thrown by the LuaJIT itself.

```text
ERROR: gen-test-set.lua:6: bad argument #1 to 'open' (string expected, got nil)
```

So when writing the error message, we should keepa the same format as the above error message.
The message above is short, clear and contains enough message to the end user.
