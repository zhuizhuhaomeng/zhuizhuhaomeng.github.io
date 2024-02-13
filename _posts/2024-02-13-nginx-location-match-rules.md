---
layout: post
title: "Nginx location 匹配规则"
description: "Nginx locaiton 匹配规则"
date: 2024-02-13
tags: [nginx,location]
---

Nginx 的 location 匹配规则是可以时说是比较灵活的，但是也可以说是比较复杂的。

有些同学总喜欢问一些 `奇怪的问题`，但是他们不说这些问题背后的场景。

到底是他们必须这么使用还是说他自己做了试验发现不符合自己脑袋中的预期。
比如 ["location = /" can not be matched](https://github.com/openresty/openresty/issues/933) 这个问题。

其实在 [Nginx 的官方文档](https://docs.nginx.com/nginx/admin-guide/web-server/web-server/#locations)中，对 Location 的匹配规则说的是非常的清楚。我们把 Location 部分的摘录如下：

To find the location that best matches a URI, NGINX Plus first compares the URI to the locations with a prefix string. It then searches the locations with a regular expression.

Higher priority is given to regular expressions, unless the ^~ modifier is used. Among the prefix strings NGINX Plus selects the most specific one (that is, the longest and most complete string). The exact logic for selecting a location to process a request is given below:

1. Test the URI against all prefix strings.
2. The = (equals sign) modifier defines an exact match of the URI and a prefix string. If the exact match is found, the search stops.
3. If the ^~ (caret-tilde) modifier prepends the longest matching prefix string, the regular expressions are not checked.
4. Store the longest matching prefix string.
5. Test the URI against regular expressions.
6. Stop processing when the first matching regular expression is found and use the corresponding location.
7. If no regular expression matches, use the location corresponding to the stored prefix string.

从上面可以看到，
1. 正则表达式的优先级更高，但是测试正则表达式是在前缀匹配之后的。
2. 可能存在多个 location 的前缀都能够匹配目标 URI。
3. ^~ 的匹配如果是最长的前缀，那么会跳过正则表达式。反之，如果最长的前缀匹配不是 ^~, 那么还是优先使用正则匹配的结果。
4. =~ 是精确匹配，如果 URI 匹配了 =~，那么就不在继续匹配了。所谓精确匹配，就是跟前缀匹配是不同的，必须整个 URI 和 location = 定义的路径完全相同。这样也意味着对于以 / 结尾的情况是不会展开去匹配 index.html 的。

所以前面的奇怪问题就是因为 `location =/` 不会展开成 `location =/index.html`, 所以 `location =/` 并没有匹配成功, 因此会走到 `location /`。


同时官方文档中也说明了， `=` 修饰符的作用主要是用来加速 `location` 的处理。

```text
A typical use case for the = modifier is requests for / (forward slash). If requests for / are frequent, specifying = / as the parameter to the location directive speeds up processing, because the search for matches stops after the first comparison.
```

```nginx
location = / {
    #...
}
```
