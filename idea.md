# 激活Jetbrains系列产品(2018 2.x可用)

- 记录一个IDEA激活服务器的相关地址 [rover12421.com](https://rover12421.com/)
- 相关网站 [idea.lanyus.com](http://idea.lanyus.com/)
- 相关教程 
  - [https://blog.ouyanglol.com/article/details/258366](https://blog.ouyanglol.com/article/details/258366)
  - [https://blog.csdn.net/zwj1030711290/article/details/81635167](https://blog.csdn.net/zwj1030711290/article/details/81635167)
- 相关文件
  - 本地下载 [JetbrainsCrack-3.1-release-enc.jar](./download/JetbrainsCrack-3.1-release-enc.jar)
  - 其他网站下载 [http://idea.lanyus.com/jar/JetbrainsCrack-3.1-release-enc.jar](http://idea.lanyus.com/jar/JetbrainsCrack-3.1-release-enc.jar)

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
    "licenseId":"ThisCrackLicenseId",
    "licenseeName":"随便填",
    "assigneeName":"随便填",
    "assigneeEmail":"随便填，随便填",
    "licenseRestriction":"随便填，随便填",
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
