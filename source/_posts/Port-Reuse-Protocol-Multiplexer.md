---
title: 如何在Web服务器80端口上开启SSH服务
date: 2016-10-29 20:16:43
categories: [网络]
tags: [网络, Go, Python, 端口复用, sslh]
---
本文所讨论的网络端口复用并非指网络编程中采用SO\_REUSEADDR选项的 Socket Bind 复用。它更像是一个带特定路由功能的端口转发工具，在应用层实现。

## 背景

笔者所处网络中防火墙只开放了一个端口，但却希望能够提供多种网络服务用于测试。所以需要寻求一种解决方案，能够对TCP数据包特征进行识别，用以实现在一个开放端口上同时提供HTTP/SSH/MQTT等多种服务。

比如说，你可以在80端口上复用一个SSH服务，普通用户只知道浏览器访问http://x.x.x.x/ ，而你却可以用 `ssh user@x.x.x.x -p 80` 这样的方式来访问你的服务器，这也不失为一种隐藏SSH服务的办法。

## 端口复用神器 - sslh

[sslh][]是一款采用C语言编写的开源端口复用软件，目前支持 HTTP、SSL、SSH、OpenVPN、tinc、XMPP等多种协议识别。它主要运行于\*nix环境，源代码托管在[GitHub][sslh_gh]上。据官网介绍，Windows系统下可在Cygwin环境中编译运行，笔者未作测试。

编译过程并不复杂，直接按照官方文档操作，不在此赘述。Debian用户可直接通过`sudo apt-get install sslh`安装。

编译生成两个可执行文件：sslh-fork 和 sslh-select 。二者的区别在于工作模式的差异：

- `sslh-fork` 采用\*nix的进程fork模型，为每一个TCP连接fork一个子进程来处理包的转发。对于长连接而言，无需频繁建立大量新连接，fork带来的开销基本可以忽略。但是如果对像HTTP这样的短连接请求，采用fork子进程的方式来进行包转发的话，在出现大量并发请求时，效率会受到一定影响。不过fork模式经过了良好测试，运行起来稳定可靠。
- `sslh-select` 采用单线程监控管理所有网络连接，是比较新的一种方式。但相对epool等基于事件的I/O机制来说，select的传统轮询模式效率还是相对较低的。

sslh支持在配置文件中使用正则表达式来自定义协议识别规则，但是我在尝试 MQTT v3.1 协议识别时，出现了[问题][issuse1]。当然也有可能是我编写的正则表达式和它使用的正则库不匹配。

[sslh]: http://www.rutschle.net/tech/sslh.shtml
[sslh_gh]: https://github.com/yrutschle/sslh
[issuse1]: https://github.com/yrutschle/sslh/issues/87

## 高性能负载均衡器 - HAProxy

[HAProxy][]是一款开源高性能的 TCP/HTTP 软件负载均衡器，目前在游戏后端服务和Web服务器负载均衡等方面都有着非常广泛的应用。通过配置，可以实现多种SSL应用复用同一个端口，比如 HTTPS、SSH、OpenVPN等。这里有一篇[参考文档][1]。

虽然HAProxy性能卓越，但它不容易通过扩展来满足特定的需求。

[HAProxy]: http://www.haproxy.org/
[1]: http://www.tuicool.com/articles/auqyuur

## 为网络而生的现代语言 - Go

[Go][]语言是近几年我学习研究过的优秀编程语言之一，它的简洁和高效深深吸引了我（我喜欢简单的东西，比如Python）。Go语言的goroutine在语言级别提供并发支持，channel又在这些协程之间提供便捷可靠的通信机制。结合起来，Go语言非常适合编写高并发的网络应用。之前也打算过用Python+gevent的方式，最后还是考虑到Go语言静态编译后的高效率，没有选择Python。

在Github上翻腾，找到一个Go语言实现的类sslh项目——[Switcher][]。它很久没有更新，支持的协议也非常少——实际上它只能识别SSH协议。Switcher的实现非常简单，核心代码不到200行。于是决定在它的基础上进行改造，实现我所需要的功能。

[Go]: http://golang.org 
[Switcher]: https://github.com/jamescun/switcher

## D——I——Y

到Github上[fork][]了一份Switcher代码，在它的基础上修改。说是修改，其实已面目全非。新的实现中调整了原有架构，去掉对SSH协议的直接支持，转而采用更加通用的协议识别模式，以求达到可以不通过修改程序而只需简单配置即可支持大部分协议，让程序通用性更强一些。

首先最常见的协议匹配模式是根据packet头几个字节对目标协议特征进行比对。如果只是保存每个协议的头N个字节，不加任何处理逐一比对的话，可能会存在一定的效率问题。一方面，需要对所有pattern进行遍历，逐个与收到的packet进行比较；另一方面，如果网络延时较大，不能一次性收集到足够多的字节，则需要反复多次比对。举一个比较极端的例子，假设我有100个目标协议需要比对匹配，pattern大小都在10字节以上，这时候我通过telnet/netcat连接服务器，一个字节一个字节的发送数据，则服务器可能要进行10\*100次字符串比较。

为了解决这个问题，简单设计了一个树形结构，把所有的pattern都以字节为单位填充到这棵树上，直至末梢。叶节点上保存协议对应的目标IP和端口值。

```go
func (t *MatchTree) Add(p *PREFIX) {
	for _, patternStr := range p.Patterns {
		pattern := []byte(patternStr)
		node := t.Root
		for i, b := range pattern {
			nodes := node.ChildNodes
			if next_node, ok := nodes[b]; ok {
				node = next_node
				continue
			}

			if nodes == nil {
				nodes = make(map[byte]*MatchTreeNode)
				node.ChildNodes = nodes
			}

			root, leaf := createSubTree(pattern[i+1:])
			leaf.Address = p.Address
			nodes[b] = root

			break
		}
	}
}
```

也许是我想太多，在需要比对的协议数量很少的情况下，可能这样的设计并不能带来根本上的效率提升。不过我喜欢这种为了可能的效率提升而不断努力的赶脚 ^\_^

相比packet prefix匹配的模式，正则表达式会更加灵活。所以我采取类似sslh的方式，加入了对正则表达式的支持。考虑到效率和具体实现的问题，对正则表达式匹配规则加入了一定的限制，比如需要知道目标字符串的最大长度。正则表达式只能在packet buffer达到一定长度要求的情况下逐一匹配。

```go
func (p *REGEX) Probe(header []byte) (result ProbeResult, address string) {
	if p.MinLength > 0 && len(header) < p.MinLength {
		return TRYAGAIN, ""
	}
	for _, re := range p.regexpList {
		if re.Match(header) {
			return MATCH, p.Address
		}
	}

	if p.MaxLength > 0 && len(header) >= p.MaxLength {
		return UNMATCH, ""
	}

	return TRYAGAIN, ""
}
```

基于上述两种简单的匹配规则，很容易可以构造出ssh、http等常用的协议。在实现中，我加入了一些常用协议的支持，省去用户自定义的麻烦。

```go
	case "ssh":
		service = "prefix"
		p = &PREFIX{ps.BaseConfig, []string{"SSH"}}
	case "http":
		service = "prefix"
		p = &PREFIX{ps.BaseConfig, []string{"GET ", "POST ", "PUT ", "DELETE ", "HEAD ", "OPTIONS "}}
```

特殊的协议还是需要单独实现的。比如说，我所需要的MQTT协议就无法通过简单的字符串比对或者正则表达式方式来进行识别。因为它没有既定的模式，结构也不是固定长度。MQTT协议识别实现如下：

```go
func (s *MQTT) Probe(header []byte) (result ProbeResult, address string) {
	if header[0] != 0x10 {
		return UNMATCH, ""
	}

	if len(header) < 13 {
		return TRYAGAIN, ""
	}

	i := 1
	for ; ; i++ {
		if header[i]&0x80 == 0 {
			break
		}

		if i == 4 {
			return UNMATCH, ""
		}
	}

	i++

	if bytes.Compare(header[i:i+8], []byte("\x00\x06MQIsdp")) == 0 || bytes.Compare(header[i:i+6], []byte("\x00\x04MQTT")) == 0 {
		return MATCH, s.Address
	}

	return UNMATCH, ""
}
```

配置文件采用了json格式，主要是为了方便和灵活。下面是一个示例：

```
{
    "listen": ":80",
    "default": "127.0.0.1:80",
    "timeout": 1,
    "connect_timeout": 1,
    "protocols": [
        {
            "service": "ssh",
            "addr": "127.0.0.1:22"
        },
        {
            "service": "mqtt",
            "addr": "127.0.0.1:1883"
        },
        {
            "name": "custom_http",
            "service": "regex",
            "addr": "127.0.0.1:8080",
            "patterns": [
                "^(GET|POST|PUT|DELETE|HEAD|\\x79PTIONS) "
            ]
        },
        {
            "service": "prefix",
            "addr": "127.0.0.1:8081",
            "patterns": [
                "GET ",
                "POST "
            ]
        }
    ]
}
```

[fork]: https://github.com/jackyspy/switcher

## 性能测试

### 准备工作

首先准备一个简单的Web服务应用。之前用Python+bjoern写过一个简易脚本，用来自己测试网络带宽，但找了半天没找着。干脆用Go语言重新弄了一个，功能是根据传入参数值N，返回N个字符。

```go
package main

import (
	"bytes"
	"flag"
	"fmt"
	"log"
	"net/http"
	"regexp"
	"strconv"
	"strings"
)

func defaultHandler(w http.ResponseWriter, r *http.Request) {
	if r.URL.Path == "/" {
		fmt.Fprintln(w, "It works.")
		return
	}

	myHandler(w, r)
}

func myHandler(w http.ResponseWriter, r *http.Request) {
	re := regexp.MustCompile(`^/(\d+)([kKmMgGtT]?)$`)
	match := re.FindStringSubmatch(r.URL.Path)
	if match == nil {
		http.NotFound(w, r)
		return
	}

	buffSize := 20480
	buff := bytes.Repeat([]byte{'X'}, buffSize)

	size, _ := strconv.ParseInt(match[1], 10, 64)
	switch strings.ToLower(match[2]) {
	case "k":
		size *= 1 << 10
	case "m":
		size *= 1 << 20
	case "g":
		size *= 1 << 30
	case "t":
		size *= 1 << 40
	}

	w.Header().Set("Content-Length", strconv.FormatInt(size, 10))
	for buffSize := int64(buffSize); size >= buffSize; size -= buffSize {
		w.Write(buff)
	}
	if size > 0 {
		w.Write(bytes.Repeat([]byte{'X'}, int(size)))
	}

}

func main() {
	portPtr := flag.Int("port", 8080, "监听端口")

	flag.Parse()

	http.HandleFunc("/", defaultHandler)
	err := http.ListenAndServe(fmt.Sprintf(":%d", *portPtr), nil)
	if err != nil {
		log.Fatal("ListenAndServe: ", err)
	}
}
```

编译运行，测试Web服务器运行正常。

```bash
$ go build test.go
$ ./test -port 9999 &
$ curl localhost:9999/1
X
$ curl localhost:9999/10
XXXXXXXXXX
$ curl -o /dev/null localhost:9999/10g
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 10.0G  100 10.0G    0     0  1437M      0  0:00:07  0:00:07 --:--:-- 1469M
```

类似于上面的演示过程，我用curl下载大文件来测试网络I/O速率。当然本次测试过程并没有经过物理网卡，而是直接通过了loopback接口。这样可以更客观的比对在经过代理后，速度下降的幅度。

另外，采用类似Apache的ab压力测试工具，测试高并发情况下的Web响应速度。这里，我使用了比ab更变态的boom。它是一款Go语言实现的开源压测软件，最近刚更名为hey，主页上称因其与Python版压力测试工具[Boom!][boom]名称冲突。安装和使用都很简单：

```bash
$ go get -u github.com/rakyll/hey
$ $GOPATH/bin/hey http://localhost:9999/1
......
All requests done.

Summary:
  Total:        0.0223 secs
  Slowest:      0.0182 secs
  Fastest:      0.0002 secs
  Average:      0.0039 secs
  Requests/sec: 8962.9371
  Total data:   200 bytes
  Size/request: 1 bytes
......
```

下载安装我修改过的Switcher版本：

```
$ go get github.com/jackyspy/switcher
```

sslh运行的命令如下：

```
$ sudo sslh-select -n -p 127.0.0.1:9998 --ssh 127.0.0.1:22 --http 127.0.0.1:9999
```

测试Switcher采用下面的配置文件default.cfg：

```
{
    "listen": ":9997",
    "default": "127.0.0.1:22",
    "timeout": 1,
    "connect_timeout": 1,
    "protocols": [
        {
            "service": "ssh",
            "addr": "127.0.0.1:22"
        },
        {
            "service": "http",
            "addr": "127.0.0.1:9999"
        }
    ]
}
```

测试过程主要用到下面两条命令，测试sslh和switcher时更改端口号即可。

```
$ curl -o /dev/null localhost:9999/10g
$ $GOPATH/bin/hey -n 100000 http://localhost:9999/1
```

OK，万事俱备，只待开测。

[boom]: https://github.com/tarekziade/boom

### 开始测试

测试分为两块，一是测试大文件下载速率，为了不受限于网卡速率，在本机测试。另一块是测试Web请求并发量，在另一台电脑上发起测试。

为了减少人工操作量，简单用Python写了一段代码，用于多次测试速度并输出结果：

```python
# coding=utf-8
from __future__ import print_function
import itertools
from subprocess import check_output


def get_speed(port):
    cmd = 'curl -o /dev/null -s -w %{{speed_download}} localhost:{}/10g'.format(port)  # noqa
    speed = check_output(cmd.split())
    return float(speed)


def test_multi_times(port, times):
    return map(get_speed, itertools.repeat(port, times))


def format_speed(speed):
    return str(int(0.5 + speed / 1024 / 1024))


def main():
    testcases = {
        'Direct': 9999,
        'sslh': 9998,
        'switcher': 9997
    }

    count = 10

    print('| Target | {} | Avg | '.format(
        ' | '.join(str(x) for x in range(1, count + 1))))
    print(' --: '.join('|' * (count + 3)))
    for name, port in testcases.items():
        speed_list = test_multi_times(port, count)
        speed_list.append(sum(speed_list) / len(speed_list))
        print('|{}|{}|'.format(name, '|'.join(map(format_speed, speed_list))))


if __name__ == '__main__':
    main()
```

运行后得到结果如下（速度单位是MB/s）：

| Target | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | Avg | 
| --: | --: | --: | --: | --: | --: | --: | --: | --: | --: | --: | --: |
|switcher|870|876|924|915|885|928|904|880|909|898|899|
|sslh|866|865|860|880|865|861|866|863|864|856|865|
|Direct|1446|1505|1392|1362|1423|1419|1395|1492|1412|1427|1427|

可以看出经过代理后，下行速率有明显下降。其中sslh比switcher略低，差异不是太大。

同样的，为了方便测试并发请求响应，也写了一个脚本来完成：

```
# coding=utf-8
from __future__ import print_function
import itertools
from subprocess import check_output


def get_speed(url):
    cmd = "hey -n 100000 -c 50 {}  | grep 'Requests/sec'".format(url)  # noqa
    output = check_output(cmd, shell=True)
    return float(output.partition(':')[2])


def test_multi_times(url, times):
    return map(get_speed, itertools.repeat(url, times))


def main():
    testcases = {
        'Direct': 'http://x.x.x.x:9999/1',
        'sslh': 'http://x.x.x.x:9998/1',
        'switcher': 'http://x.x.x.x:9997/1'
    }

    count = 10

    print('| Target | {} | Average | '.format(
        ' | '.join(str(x) for x in range(1, count + 1))))
    print(' --: '.join('|' * (count + 3)))
    for name, port in testcases.items():
        speed_list = test_multi_times(port, count)
        speed_list.append(sum(speed_list) / len(speed_list))
        print('|{}|{}|'.format(name, '|'.join('{:.0f}'.format(x + 0.5)
                                              for x in speed_list)))


if __name__ == '__main__':
    main()
```

运行后得到结果如下（速度单位是Requests/s）：

| Target | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | Average | 
| --: | --: | --: | --: | --: | --: | --: | --: | --: | --: | --: | --: |
|switcher|14367|14886|15144|14289|15456|14834|14871|14951|14610|14865|14827|
|sslh|13892|14281|14469|14352|14468|14132|14510|14565|14633|14555|14386|
|Direct|20494|20110|20558|19519|19467|19891|19777|19682|20737|20396|20063|

类似前面的测试，RPS在经过代理后也存在较明显下降。sslh比switcher略低，差异不大。

## 更多应用场景 ？？
本文描述的网络端口复用，其实现方式本质上还是一个TCP应用代理。基于这一点，我们还可以扩展出很多其他的应用场景。

我想到的一种场景是动态IP认证。我们对HTTP和SSH进行复用，默认情况下HTTP可以被所有人访问，但SSH却需要通过IP地址认证后才会进行包转发。跟iptables等防火墙实现的IP地址访问规则不同，它是在应用层面来进行限制的，具有很强的灵活性，可以通过程序动态增加和删除。比如说，我通过手机浏览器访问特定的鉴权页面，通过验证后，系统自动将我当前在用的公网IP地址加入到访问列表，然后就能够顺利地通过SSH访问服务器了。连接建立后，可以将临时IP地址从访问列表中剔除，从一定程度上加强了服务器安全。

