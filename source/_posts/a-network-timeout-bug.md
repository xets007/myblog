---
title: Bug你别跑，让我拍死你
date: 2016-09-20 21:34:21
categories:
- Network
tags:
- Process
---

在cuckoo上测试我的xenforce，任务运行结果出现大量timeout错误，虽然以前也遇到过这个问题，但没当回事放过去了，现在有了xenforce还出现这种问题，是可忍孰不可忍，今天就认真排查下这个问题，消灭这只bug。

Bug现象是这样的，任务运行超时，错误日志有两种：

<!-- more -->

```
2016-09-20 14:08:58,540 [lib.cuckoo.core.scheduler] ERROR: vm-111: the guest initialization hit the critical timeout, analysis aborted.
```

```
2016-09-20 14:04:27,790 [lib.cuckoo.core.guest] DEBUG: vm-117: error retrieving status: timed out
2016-09-20 14:04:28,799 [lib.cuckoo.core.scheduler] ERROR: The analysis hit the critical timeout, terminating.
```

还有另外一种情况，没错误，但运行时间也超出正常值，时间主要消耗在这里：

```
2016-09-20 19:06:52,752 [lib.cuckoo.core.guest] DEBUG: vm-103: waiting for status 0x0001
2016-09-20 19:08:52,803 [lib.cuckoo.core.guest] DEBUG: vm-103: not ready yet
```

总之，是访问虚拟机内agent.py提供的XMLRPC服务出现了超时问题。Why？为什么会超时？这影响的因素就太多了，只能一条条排除。

首先排除掉网络配置问题，xenforce在网络配置上做了优化，虚拟机快照恢复后，立马自动配置IP地址，可正常访问网络。观察日志，以及虚拟机内检测包和外部resultserver的正常通信也证实了我的想法。

是虚拟机环境问题导致的吗？虚拟机是通过快照恢复的，也就是说环境都是一样的，先排除掉。

既然是网络连接问题，先在主机上看下网络连接(8000是agent.py提供的XMLRPC服务端口，这里只列出了部分netstat输出结果)：

```
# netstat -anp | grep :8000
tcp        0      0 172.16.1.1:53033            172.16.1.108:8000           ESTABLISHED 336/tcpdump         
tcp        0      0 172.16.1.1:37476            172.16.1.122:8000           ESTABLISHED 336/tcpdump         
tcp        0      0 172.16.1.1:36381            172.16.1.110:8000           ESTABLISHED 336/tcpdump         
tcp        0      0 172.16.1.1:50635            172.16.1.112:8000           ESTABLISHED 336/tcpdump         
tcp        0      0 172.16.1.1:50222            172.16.1.104:8000           ESTABLISHED 336/tcpdump         
tcp        0      0 172.16.1.1:34851            172.16.1.103:8000           ESTABLISHED 336/tcpdump         
tcp        0      0 172.16.1.1:46671            172.16.1.121:8000           ESTABLISHED 32445/tcpdump       
tcp        0      0 172.16.1.1:53098            172.16.1.113:8000           ESTABLISHED 30667/tcpdump       
tcp        0      0 172.16.1.1:46680            172.16.1.121:8000           ESTABLISHED 336/tcpdump         
tcp        0      0 172.16.1.1:44107            172.16.1.123:8000           ESTABLISHED 32559/tcpdump       
tcp        0      0 172.16.1.1:53634            172.16.1.116:8000           CLOSE_WAIT  336/tcpdump 
```

像CLOSE_WAIT, TIME_WAIT这种状态，大量出现一般是有问题的。另外，还有**一个更有意思的现象**，这些网络连接为什么显示是`tcpdump`进程建立的？Why？`tcpdump`就一个抓包进程，怎么会去访问XMLRPC服务。哦？哦！**`tcpdump`进程是以子进程形式启动的，继承了父进程建立的网络连接，所以才会有上面的显示**。

直觉问题就出在`tcpdump`。

关掉抓包模块，跑了一遍测试，没有timeout错误，又跑了一遍，仍然没有。:)

`tcpdump`进程是用`subprocess`模块启动的，将`subprocess.Popen()`的`close_fds`参数设置为`True`，子进程启动时，关闭从父进程继承的文件描述符。跑了一遍测试，没有timeout错误，又跑了一遍，仍然没有。:)

接着排查。

当tcpdump进程继承了ESTABLISHED状态的连接时，当相同IP的虚拟机再次启动时，里面的agent.py就会卡住，对外面的请求没有响应。如:

```
tcp        0      0 172.16.1.1:44727            172.16.1.121:8000           ESTABLISHED 32445/tcpdump  
```

172.16.1.121这台虚拟机再次启动时就会出现问题，agent.py卡住，长时间没响应，于是外面访问时出现超时问题。在虚拟机内部观察netstat状态：

![netstat](http://7xtc3e.com1.z0.glb.clouddn.com/a-network-timeout-bug/agent_netstat.png)

将对应的tcpdump进程手动kill掉，ESTABLISHED连接状态消失，agent.py恢复正常，可正常响应请求。


问题来了，要找到根本原因得回答下面的问题：

1. ESTABLISHED，CLOSE_WAIT状态是怎么出现的？
2. tcpdump进程继承的连接为什么会导致这些状态出现？
3. 这些状态出现时，agent.py为什么会卡住没响应？

---

(2016.09.21 10:34更新)

先回答问题3， Python的SimpleXMLRPCServer是单线程的，串行执行请求，也就是说每次只能处理一个请求，当有请求正在处理，新的请求会阻塞住，就会出现agent.py卡住没响应的现象。

写个简单的demo测试下：

[function_server.py](https://github.com/consen/demo/blob/master/python/library/SimpleXMLRPCServer/function_server.py)，提供一个简单的XMLRPC服务。

```
from SimpleXMLRPCServer import SimpleXMLRPCServer
import logging
import os
import time

# Set up logging
logging.basicConfig(level=logging.DEBUG)

server = SimpleXMLRPCServer(('localhost', 9090), logRequests=True)

# Expose a function
def list_contents(dir_name):
    logging.debug('list_contents(%s)', dir_name)
    time.sleep(20)
    return os.listdir(dir_name)

def list_contents2(dir_name):
    logging.debug('list_contents(%s)', dir_name)
    return os.listdir(dir_name)

server.register_function(list_contents)
server.register_function(list_contents2)

try:
    print 'Use Control-C to exit'
    server.serve_forever()
except KeyboardInterrupt:
    print 'Exiting'
```

[function_client.py](https://github.com/consen/demo/blob/master/python/library/SimpleXMLRPCServer/function_client.py)

```
import xmlrpclib

proxy = xmlrpclib.ServerProxy('http://localhost:9090')
print proxy.list_contents('./')
```

[function_client2.py](https://github.com/consen/demo/blob/master/python/library/SimpleXMLRPCServer/function_client2.py)

```
import xmlrpclib

proxy = xmlrpclib.ServerProxy('http://localhost:9090')
print proxy.list_contents2('./')
```

先启动function_client.py，由于list_contents()会sleep 20s，模拟XMLRPC服务正在长时间处理当前请求。

再启动fuction_client2.py，由于XMLRPC服务正在处理function_client.py的list_contents()请求，list_contents2()请求会被阻塞住，没响应。

![xmlprc demo](http://7xtc3e.com1.z0.glb.clouddn.com/a-network-timeout-bug/xmlrpc_demo.png)

这就解释了虚拟机内agent.py提供的XMLRPC服务，为什么会出现无响应的现象，因为有连接建立，有正在处理的请求，**见上面虚拟机内netstat截图中ESTABLISHED状态的连接**。
