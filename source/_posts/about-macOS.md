title: macOS相关小记录
date: 2016-10-16 16:19:31
description: 
categories: 其他
tags: [macOS]
toc: 
feature: 
---
### 安全性和隐私->"任何来源"
```
打开终端，执行sudo spctl --master-disable即可
```
### Patch工具运行失败
```
右键显示包内容进入Contents->MacOS,两个文件eyePatch,patcher.
打开 终端 ，按照以下顺序，将对应文件依次拖入终端窗口-> patcher - app - eyePatch - app(注意:-为空格)
```
### 安装Homebrew
```
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```
### npm更换国内源
```
npm config get registry
npm config set registry https://registry.cnpmjs.org
恢复:npm config set registry https://registry.npmjs.org/
```
### npm发布模块
```
npm adduser
npm whoami
npm publish
```
### 安装全局模块
```
sudo npm install -g hexo-cli bower grunt-cli
```
### 终端SS
```
brew install proxychains-ng
如果是SS客户端,默认端口1080
/usr/local/etc/proxychains.conf (end)->socks5  127.0.0.1 9050(改成1080)
```
### 安装compass
```
sudo gem install compass
```
### 小版本更新后登陆节目英文
```
sudo languagesetup
```
选择中文序号,无需重启.