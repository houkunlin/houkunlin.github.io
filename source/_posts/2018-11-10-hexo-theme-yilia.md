---
title: hexo博客系统更换主题：yilia
date: 2018-11-10 16:00:26
tags: 
- 教程
---

首选文档：[官方中文文档](https://hexo.io/zh-cn/index.html)

官方教程走一遍
```
npm install hexo-cli -g
hexo init blog
cd blog
npm install
hexo server
```

到这里就成功的运行起自己的博客了，接下来就是要更换主题了。

主题地址：[https://github.com/litten/hexo-theme-yilia](https://github.com/litten/hexo-theme-yilia)

同样根据文档走一遍，官方文档最好用，我就不抄了[https://github.com/litten/hexo-theme-yilia/blob/master/README.md](https://github.com/litten/hexo-theme-yilia/blob/master/README.md)

说一点其他的，运行起来后会发现首页的文章列表会显示完整的文章，怎样让它只显示文章的一部分呢？
此时我们需要安装`hexo-excerpt`模块，运行`npm install --save hexo-excerpt`，然后在博客目录下的`_config.yml`添加如下内容
```
excerpt:
  # 最多只显示3个HTML节点
  depth: 3
  excerpt_excludes: []
  more_excludes: []
```
此时再重新运行，完美，首页每篇文章只会显示一部分。
