---
title: Ansible学习 
tags: 
---

# 安装
```sh
yum -y install epel-release
yum -y install ansible
```

# ansible基础
## 清单文件
```
[servers]
172.20.21.120
172.20.21.121
172.20.21.123
```
默认清单文件是`/etc/ansible/hosts`。

同时也可以在运行时通过`-i`参数来指定清单文件。

## 测试主机连通性
```sh
ansible www.baidu.com -m ping
```
发现执行结果异常，这是因为`ansible`只纳管定义在**清单文件**中的主机。

![需要清单文件][1]

```sh
ansible servers -m ping
ansible servers -m ping -o
```
通过`-o`参数，可以让结果在一行中进行显示。

![测试主机连通性][2]

## 执行远程指令
```sh
ansible servers -m shell -a 'uptime'
ansible servers -m command -a 'uptime'
```
![执行远程指令][3]


  [1]: https://www.github.com/hoxis/token4md/raw/master/1514178939675.jpg
  [2]: https://www.github.com/hoxis/token4md/raw/master/1514179663946.jpg
  [3]: https://www.github.com/hoxis/token4md/raw/master/1514179908403.jpg