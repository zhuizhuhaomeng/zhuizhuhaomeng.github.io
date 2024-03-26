---
layout: post
title: "Add Open Telemetry Support for OpenResty"
description: "OpenResty 的软件是如何保障高质量的"
date: 2023-03-05
tags: [OpenResty, Nginx, Valgrind, Address Sanity, Mock, Test, Quality]
---


server {
        server_name example.com;
        return 301 $scheme://www.example.com$request_uri;
}

server {
        server_name "~^(?!www\.).*" ;
        return 301 $scheme://www.$host$request_uri;
}

server {
        server_name www.example.com;
        return 301 $scheme://example.com$request_uri;
}

server {
         server_name "~^www\.(.*)$" ;
         return 301 $scheme://$1$request_uri ;
}
