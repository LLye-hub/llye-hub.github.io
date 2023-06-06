---
title: SQL之distirbute by rand有坑
tags:
  - 文章内容的关键词
  - private
categories:
  - 对照文件存放的目录名称
toc: true
abbrlink: 68201c19
date: 2023-05-23 17:29:09
---
https://zhuanlan.zhihu.com/p/252776975

 

select
posexplode(split(space(99), ' ')) as (pos, val),
pmod(hash(1000 * rand(3)), 80)