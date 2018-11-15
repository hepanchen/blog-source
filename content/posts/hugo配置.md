---
title: "Github pages +  Hugo配置"
date: 2018-11-07T16:35:24+08:00
draft: true
---

![](hugo配置-0c547.png)

# Github pages 配置

* 创建以用户名开头+.github.io的项目
![](hugo配置-61d42.png)

---
# Hugo 配置

* hugo 入门

 * [hugo quick start](https://gohugo.io/getting-started/quick-start/)


* hugo github pages 整合

 * [integrating github pages and hugo](https://gohugo.io/hosting-and-deployment/hosting-on-github/)

---
# trouble shooting

* hugo 未发布pags的问题
```
hugo --cleanDestinationDir -c content -v -D 构建web
```

* hugo 图片放置路径和相对路径不顺手的问题
```
使用 hugo-image 插件解决
```

* apm deploy插件的问题
```
git remote set-url https://<username>:<password>@github.com/<username>/<repo_name>.git
git config --global credential.helper wincred
apm publish 0.0.0
```
