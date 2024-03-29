---
title: 激活 Jetbrains 系列产品(2018 2.x/3.x可用)
date: 2018-08-23 10:24
updated: 2018-11-27 09:33:44
categories:
- 开发工具
- Jetbrains
tags: 
- IDEA
- WebStorm
---
> 2023年4月20日更新：这个方法已经过时了，无法再激活新版本了

> 2018年11月27日更新记录：添加破解补丁3.4版本。
    在下载2018.3.x版本的GoLang时，使用3.1版本破解补丁无法破解，此时更换破解补丁为3.4版本即可。破解过程与3.1版本一致并无变化。

- 记录一个IDEA激活服务器的相关地址 [rover12421.com](https://rover12421.com/)
- 相关网站，Jetbrains激活相关网站 [idea.lanyus.com](http://idea.lanyus.com/)
- 相关教程 
  - [https://blog.ouyanglol.com/article/details/258366](https://blog.ouyanglol.com/article/details/258366)
  - [https://blog.csdn.net/zwj1030711290/article/details/81635167](https://blog.csdn.net/zwj1030711290/article/details/81635167)
- 相关文件
  - 本地下载(2018 2.x可用) [JetbrainsCrack-3.1-release-enc.jar](./JetbrainsCrack-3.1-release-enc.jar)
  - 本地下载(2018 3.x可用) [JetbrainsIdesCrack-3.4-release-enc.jar](./JetbrainsIdesCrack-3.4-release-enc.jar)
  - 其他网站下载(2018 2.x可用) [http://idea.lanyus.com/jar/JetbrainsCrack-3.1-release-enc.jar](http://idea.lanyus.com/jar/JetbrainsCrack-3.1-release-enc.jar)
  - 其他网站下载(2018 3.x可用) [http://idea.lanyus.com/jar/JetbrainsIdesCrack-3.4-release-enc.jar](http://idea.lanyus.com/jar/JetbrainsIdesCrack-3.4-release-enc.jar)

## 亲测 IDEA、WebStorm 可用

##### 准备条件：
下载JetbrainsCrack-3.1-release-enc.jar保存到本地，并记住保存路径

1. 进入到软件安装目录，打开安装目录下的bin目录
2. 打开 *.vmoptions 文件，准备进行编辑
    1. IDEA 是 idea.vmoptions 和 idea64.vmoptions 文件
    2. WebStorm 是 webstorm.vmoptions 和 webstorm64.vmoptions 文件
3. 在 *.vmoptions 文件末尾添加一行代码，然后保存文件

```
-javaagent:/home/JetbrainsCrack-3.1-release-enc.jar

解释：其中 -javaagent: 破解jar包的全路径(或者相对路径)
```

4. 开始运行软件，在运行到激活步骤的时候，选择 Activation code 激活方式，然后在输入框里面输入下面代码

```
{
    "licenseId":"许可ID",
    "licenseeName":"被许可人姓名",
    "assigneeName":"受让人姓名",
    "assigneeEmail":"受让人邮箱",
    "licenseRestriction":"许可限制",
    "checkConcurrentUse":false,
    "products":[
        {"code":"II","paidUpTo":"2099-12-31"},
        {"code":"DM","paidUpTo":"2099-12-31"},
        {"code":"AC","paidUpTo":"2099-12-31"},
        {"code":"RS0","paidUpTo":"2099-12-31"},
        {"code":"WS","paidUpTo":"2099-12-31"},
        {"code":"DPN","paidUpTo":"2099-12-31"},
        {"code":"RC","paidUpTo":"2099-12-31"},
        {"code":"PS","paidUpTo":"2099-12-31"},
        {"code":"DC","paidUpTo":"2099-12-31"},
        {"code":"RM","paidUpTo":"2099-12-31"},
        {"code":"CL","paidUpTo":"2099-12-31"},
        {"code":"PC","paidUpTo":"2099-12-31"},
        {"code":"DB","paidUpTo":"2099-12-31"},
        {"code":"GO","paidUpTo":"2099-12-31"},
        {"code":"RD","paidUpTo":"2099-12-31"}
    ],
    "hash":"2911276/0",
    "gracePeriodDays":7,
    "autoProlongated":false
}
```

5. OK，激活完成！
