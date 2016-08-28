title: 创建笔记
date: 2016-08-15 14:07:32
description: 
categories: 其他
tags:
  - GitHub
toc: 
feature: 
---
有时候看了点东西,总想找个地方系统记录一下,[Hexo](https://hexo.io/)是各方不错的GitHub Pages选择,不少人推荐[Travis CI](https://travis-ci.org/)自动更新,下次有机会再倒腾.
配置过程:
```
npm install -g hexo-cli
hexo init <your-hexo-folder>
cd <your-hexo-folder>
hexo new [layout] <title>
hexo clean && hexo g -d
```
这次使用next主题
```
cd <your-hexo-folder>
git clone https://github.com/iissnan/hexo-theme-next themes/next
vim .gitignore
themes/next/*
!themes/next/_config.yml
```
使用Jetbrains IDE设置 Settings > Editor > File and code Templates:
```
title: ${title}
date: ${YEAR}-${MONTH}-${DAY} ${HOUR}:${MINUTE}:00
description: ${description}
categories: ${categories}
tags: [${tags}]
toc: ${isTocAllowed}
feature: ${thumbnails_url}
---
```
注意:
```
ssh-key已经添加到github,否则hexo deploy将失败.
若在mac下bash访问github比较慢
brew install proxychains-ng
编辑配置文件 vim /usr/local/etc/proxychains.conf
安装完成proxychains
终端输入:proxychains4 bash
即完成终端全局代理
```
PS:OSX 10.11+ proxychains无写入权限[请暂时关闭SIP](http://osxdaily.com/2015/10/05/disable-rootless-system-integrity-protection-mac-os-x/)