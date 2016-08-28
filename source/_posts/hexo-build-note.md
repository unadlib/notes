title: 创建笔记
date: 2016-08-15 14:07:32
description: 
categories: 
tags: [] 
toc: 
feature: 
---
有时候看了点东西,总想找个地方系统记录一下,hexo是各方不错的GitHub Pages选择,不少人推荐[Travis CI](https://travis-ci.org/)自动更新,下次有机会再倒腾.
配置过程:
```
npm install -g hexo-cli
hexo init <your-hexo-folder>
cd <your-hexo-folder>
hexo new [layout] <title>
hexo clean && hexo g -d
```
使用Jetbrains IDE设置 Settings > Editor > File and code Templates:
```
title: ${title}
date: ${YEAR}-${MONTH}-${DAY} ${HOUR}:${MINUTE}:${SECOND}
description: ${description}
categories: ${categories}
tags: [${tags}]
toc: ${isTocAllowed}
feature: ${thumbnails_url}
```
注意:
```
ssh-key已经添加到github,否则hexo deploy将失败.
```