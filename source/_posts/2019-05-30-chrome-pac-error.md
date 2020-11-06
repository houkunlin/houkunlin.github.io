---
title: Ubuntu下使用本地PAC文件代理导致谷歌浏览器无法走代理问题解决方案
date: 2019-05-30 16:31:23
updated: 2019-05-30 16:31:23
tags:
---

今天升级了我ubuntu的谷歌浏览器，最新版本74.0.3729.169，具体为`版本 74.0.3729.169（正式版本）Built on Ubuntu , running on Ubuntu 18.04 （64 位）`，升级后出问题了，我的谷歌浏览器无法访问走代理了。

原始的pac路径为：file:///opt/autoproxy.pac
{% asset_img pac-local-file.png Ubuntu代理配置 %}


最后通过谷歌搜索搜到了这么一篇文章，[【正解】Chrome 使用 PAC 时出现 “网页可能暂时无法连接，或者它已永久性地移动到了新网址”](https://www.oixxu.com/chrome-pac-error/)

这篇文章指出了这么一个GitHub的链接[ERR_MANDATORY_PROXY_CONFIGURATION_FAILED #1417](https://github.com/FelisCatus/SwitchyOmega/issues/1474)，其中有一个回复是这样的[该回复楼层](https://github.com/FelisCatus/SwitchyOmega/issues/1474#issuecomment-396818539)


    Seems it destn't work for --proxy-pac-url setting "file:///xxx" and "data:...", but works for "http://ip/xx.pac".
    So here's a workaround, which cannot update web page list automatically.
    
        1. export the pac script file.
        2. run a web server, and put the pac file in web server.
        3. disable the extension or switch to system setting.
        4. run with: chrome --proxy-pac-url="http://ip/xx.pac"
        I tried the workaround on one PC, it works, but didn't try more.
    
    @FelisCatus, If "--proxy-pac-url" not support "data:..." any more, it there other way to set proxy?

他说谷歌浏览器在使用--proxy-pac-url参数时，不适用于`file://`和`data:`开头的协议或内容，在这两个协议上谷歌浏览器的代理将不能正常工作，但是它使用于`http://`协议。

因此他提出了一个解决方案，就是把我们的本地pac文件导出来，放到web服务器中，使用http协议访问这个pac文件，以及禁用我们的谷歌浏览器相关扩展。

因此我在我的Ubuntu下安装nginx，然后在配置文件中新增一个路径映射
```
location /autoproxy.pac {
    alias /opt/autoproxy.pac;
}
```
当然也可以把我们的autoproxy.pac文件放到/var/www/html路径下，然后配置
```$xslt
root /var/www/html;
location / {
    try_files $uri $uri/ =404;
}
```
重启nginx，在浏览器测试访问`http://127.0.0.1/autoproxy.pac`确保这个路径能改正常访问这个pac文件即可。

之后更改我们的系统设置，然后重启谷歌浏览器即可正常的走代理。
{% asset_img pac-net-file.png Ubuntu代理配置 %}