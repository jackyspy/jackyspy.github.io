---
title: 玩Python之HTTP代理
date: 2016-11-03 15:29:38
categories: [Python]
tags: [Python, Proxy, 代理]
---
## 0x00 前言

大家对HTTP代理应该都非常熟悉，它在很多方面都有着极为广泛的应用。HTTP代理分为正向代理和反向代理两种，后者一般用于将防火墙后面的服务提供给用户访问或者进行负载均衡，典型的有Nginx、HAProxy等。本文所讨论的是正向代理。

HTTP代理最常见的用途是用于网络共享、网络加速和网络限制突破等。此外，HTTP代理也常用于Web应用调试、Android/IOS APP 中所调用的Web API监控和分析，目前的知名软件有Fiddler、Charles、Burp Suite和mitmproxy等。HTTP代理还可用于请求/响应内容修改，在不改变服务端的情况下为Web应用增加额外的功能或者改变应用行为等。

## 0x01 HTTP代理是什么

HTTP代理本质上是一个Web应用，它和其他普通Web应用没有根本区别。HTTP代理收到请求后，根据Header中Host字段的主机名和Get/POST请求地址综合判断目标主机，建立新的HTTP请求并转发请求数据，并将收到的响应数据转发给客户端。

如果请求地址是绝对地址，HTTP代理采用该地址中的Host，否则使用Header中的HOST字段。做一个简单测试，假设网络环境如下：
- 192.168.1.2 Web服务器
- 192.168.1.3 HTTP代理服务器

使用telnet进行测试

```
$ telnet 192.168.1.3
GET / HTTP/1.0
HOST: 192.168.1.2


```

注意最后需要连续两个回车，这是HTTP协议要求。完成后，可以收到 http://192.168.1.2/ 的页面内容。下面做一下调整，GET请求时带上绝对地址

```
$ telnet 192.168.1.3
GET http://httpbin.org/ip HTTP/1.0
HOST: 192.168.1.2


```

注意这里同样设置了HOST为192.168.1.2，但运行结果却返回了 http://httpbin.org/ip 页面的内容，也就是公网IP地址信息。

从上面的测试过程可以看出，HTTP代理并不是什么很复杂的东西，只要将原始请求发送到代理服务器即可。在无法设置HTTP代理的情况下，对于少量Host需要走HTTP代理的场景来说，最简单的方式就是将目标Host域名的IP指向代理服务器，可以采取修改hosts文件的方式来实现。

## 0x02 Python程序中设置HTTP代理

### urllib2/urllib 代理设置

urllib2是Python标准库，功能很强大，只是使用起来稍微麻烦一点。在Python 3中，urllib2不再保留，迁移到了urllib模块中。urllib2中通过ProxyHandler来设置使用代理服务器。

```
proxy_handler = urllib2.ProxyHandler({'http': '121.193.143.249:80'})
opener = urllib2.build_opener(proxy_handler)
r = opener.open('http://httpbin.org/ip')
print(r.read())
```

也可以用install_opener将配置好的opener安装到全局环境中，这样所有的urllib2.urlopen都会自动使用代理。

```
urllib2.install_opener(opener)
r = urllib2.urlopen('http://httpbin.org/ip')
print(r.read())
```

在Python 3中，使用urllib。

```
proxy_handler = urllib.request.ProxyHandler({'http': 'http://121.193.143.249:80/'})
opener = urllib.request.build_opener(proxy_handler)
r = opener.open('http://httpbin.org/ip')
print(r.read())
```

### requests 代理设置

requests是目前最优秀的HTTP库之一，也是我平时构造http请求时使用最多的库。它的API设计非常人性化，使用起来很容易上手。给requests设置代理很简单，只需要给proxies设置一个形如 `{'http': 'x.x.x.x:8080', 'https': 'x.x.x.x:8080'}` 的参数即可。其中http和https相互独立。

```
In [5]: requests.get('http://httpbin.org/ip', proxies={'http': '121.193.143.249:80'}).json()
Out[5]: {'origin': '121.193.143.249'}
```

可以直接设置session的proxies属性，省去每次请求都要带上proxies参数的麻烦。

```
s = requests.session()
s.proxies = {'http': '121.193.143.249:80'}
print(s.get('http://httpbin.org/ip').json())
```

## 0x03 HTTP_PROXY / HTTPS_PROXY 环境变量

urllib2 和 Requests 库都能识别 HTTP_PROXY 和 HTTPS_PROXY 环境变量，一旦检测到这些环境变量就会自动设置使用代理。这在用HTTP代理进行调试的时候非常有用，因为不用修改代码，可以随意根据环境变量来调整代理服务器的ip地址和端口。\*nix中的大部分软件也都支持HTTP_PROXY环境变量识别，比如curl、wget、axel、aria2c等。

```
$ http_proxy=121.193.143.249:80 python -c 'import requests; print(requests.get("http://httpbin.org/ip").json())'
{u'origin': u'121.193.143.249'}

$ http_proxy=121.193.143.249:80 curl httpbin.org/ip
{
  "origin": "121.193.143.249"
}
```

在IPython交互环境中，可能经常需要临时性地调试HTTP请求，可以简单通过设置 `os.environ['http_proxy']` 增加/取消HTTP代理来实现。

```
In [245]: os.environ['http_proxy'] = '121.193.143.249:80'
In [246]: requests.get("http://httpbin.org/ip").json()
Out[246]: {u'origin': u'121.193.143.249'}
In [249]: os.environ['http_proxy'] = ''
In [250]: requests.get("http://httpbin.org/ip").json()
Out[250]: {u'origin': u'x.x.x.x'}
```

## 0x04 MITM-Proxy

MITM 源于 Man-in-the-Middle Attack，指中间人攻击，一般在客户端和服务器之间的网络中拦截、监听和篡改数据。

[mitmproxy][]是一款Python语言开发的开源中间人代理神器，支持SSL，支持透明代理、反向代理，支持流量录制回放，支持自定义脚本等。功能上同Windows中的[Fiddler][]有些类似，但mitmproxy是一款console程序，没有GUI界面，不过用起来还算方便。使用mitmproxy可以很方便的过滤、拦截、修改任意经过代理的HTTP请求/响应数据包，甚至可以利用它的scripting API，编写脚本达到自动拦截修改HTTP数据的目的。

```
# test.py
def response(flow):
    flow.response.headers["BOOM"] = "boom!boom!boom!"
```

上面的脚本会在所有经过代理的Http响应包头里面加上一个名为BOOM的header。用`mitmproxy -s 'test.py'`命令启动mitmproxy，curl验证结果发现的确多了一个BOOM头。

```
$ http_proxy=localhost:8080 curl -I 'httpbin.org/get'
HTTP/1.1 200 OK
Server: nginx
Date: Thu, 03 Nov 2016 09:02:04 GMT
Content-Type: application/json
Content-Length: 186
Connection: keep-alive
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
BOOM: boom!boom!boom!
...
```

显然mitmproxy脚本能做的事情远不止这些，结合Python强大的功能，可以衍生出很多应用途径。除此之外，mitmproxy还提供了强大的API，在这些API的基础上，完全可以自己定制一个实现了特殊功能的专属代理服务器。

经过性能测试，发现mitmproxy的效率并不是特别高。如果只是用于调试目的那还好，但如果要用到生产环境，有大量并发请求通过代理的时候，性能还是稍微差点。我用twisted实现了一个简单的proxy，用于给公司内部网站增加功能、改善用户体验，以后有机会再和大家分享。

[mitmproxy]: https://mitmproxy.org/
[Fiddler]: http://www.telerik.com/fiddler

