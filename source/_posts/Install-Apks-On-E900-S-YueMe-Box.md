---
title: 在创维E900-S悦Me盒子上安装第三方软件
date: 2016-11-10 10:42:54
categories: [Android]
tags: [创维, E900-S, 悦Me, Go]
---
## 0x00 不甘寂寞
创维E900-S这款悦Me盒子功能还算可以，但不能接受它禁止安装第三方软件这一点。网上搜了半天，可能是比较新的机型没人关注，找不到任何方法，只好自己动手试试。

## 0x01 Fiddler寻路
请出小伙伴[Fiddler][]，盒子上Wifi挂代理。在高级设置的WLAN中，在已连接的SSID上按住确认键几秒钟会出现修改网络选项，设置好代理服务器（运行Fiddler的机器）的IP和端口。

在应用市场里面安装几个软件，抓包看到一些APK下载地址。在Fiddler的AutoResponder中替换为需要安装的apk，比如电视猫。再次安装会出现下载不完整的错误提示，初步猜测对文件大小或者Hash值进行了校验。下载原始文件，修改一个字节，AutoResponder替换，发现还是同样的错误，基本判定是对文件Hash进行校验。请求的文件名包含一个32字节的16进制串，估计是MD5，但是经过计算，与文件实际MD5值并不相同。怀疑是经过加了Salt或者其他处理方式得出的最终结果，不知道算法的话无法伪造。而且Url也不是通过网络请求获取，估计是在应用市场里面内置的数据。

想劫持应用市场的路不通，只能换条路走。还是开着代理，重启盒子，逐条观察Http请求。发现了一个更新应用的请求 `http://appStoreRrc.cnitv.net:8090/tv/updater2?userId=&mac=&brand=YUEME&model=E900-S&areaNo=null&osver=990104900.1008000000.10000000200&applist=[{"pkgName":"com.keylab.speech.core.yueme","version":"3.02.008"},{"pkgName":"com.keylab.speech.view.yueme","version":"3.01.008"},{"pkgName":"com.hisense.bluetooth","version":"3.01.008"}]` ，返回的Json格式化之后如下：
```
[
    {
        "result": {
            "code": 0,
            "description": "成功获取升级信息",
            "appURL": [
                {
                    "pkgName": "com.keylab.speech.core.yueme",
                    "version": "3.02.008",
                    "isUpdate": false,
                    "url": ""
                },
                {
                    "fileName": "aiView_YueMe.apk",
                    "appName": "语音助手view",
                    "pkgName": "com.keylab.speech.view.yueme",
                    "version": "3.02.008",
                    "isUpdate": true,
                    "url": "http://RSAppStore.cnitv.net/TV_appation/ctviptv/AD/aiView_YueMe.apk",
                    "md5": "a5e7af5648aa6d2a6d34ffdc925857c0"
                },
                {
                    "pkgName": "com.hisense.bluetooth",
                    "version": "3.01.008",
                    "isUpdate": false,
                    "url": ""
                }
            ]
        }
    }
]
```

于是打算在此做文章。同样的，在Fiddler中拦截替换该请求，保证有一个应用的isUpdate值为true，它就会自动尝试下载安装该apk。AutoResponder中替换所有\*.apk请求指向本地apk文件，重启盒子。Fiddler里面看到的确下载了我替换的apk文件，再到盒子里面看一下，安装成功！！

至此，基本有比较清晰的思路了。拦截update请求，将返回的JSON中isUpdate值置为true，url替换为真正需要安装的apk的地址。url如果不是可以直接访问下载的互联网地址，而是本地apk文件的话，进一步在Fiddler中进行拦截替换。

经过测试，一般系统启动时会检查3个内置软件的版本更新情况，如果返回的JSON中应用列表数量小于3的话，则不会下载安装。如果返回数量大于3，则只下载安装前3个应用。所以这种方法盒子每次启动安装的软件数量是有限制的。

[Fiddler]: http://www.telerik.com/fiddler

## 0x02 Go小程序
按照前面的方法，已经可以正常安装第三方软件了。但是，还是嫌步骤复杂了一些，尤其是要帮很多朋友大量安装的话就太麻烦。

简单用Go语言实现了一个小程序[E900-S-proxy](https://github.com/jackyspy/E900-S-proxy)，实现代理和数据替换功能。只要把待安装的apk文件放到同一目录下，启动程序，即可在电脑上启动一个默认端口为8080的代理服务器。在盒子上设置HTTP代理指向这台电脑的IP，重启设备就可以实现自动安装，大大简化了操作过程，降低门槛。存在的局限是每次最多只能安装3个软件。

需要Windows下编译好的exe文件的朋友，请移步我在[高清范发的帖子](http://www.hdpfans.com/thread-746619-1-1.html)。

最后，软件安装完毕后，别忘了把HTTP代理取消。

## 0x03 更多……
除了设置HTTP代理服务器IP端口意外，其实根据[HTTP代理的本质](http://python.jobbole.com/86747/)，还可以通过DNS劫持来实现。具体而言，可以在路由器上将域名appStoreRrc.cnitv.net和PROXY都指向运行程序的电脑，当然代理端口要改为8090才行。

尝试了3款不同一键Root工具，都没能搞定这款机器。更多玩法，期待高手们分享。

