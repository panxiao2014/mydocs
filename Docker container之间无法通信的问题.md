最近碰到一个docker container之间无法通信的问题，折腾了一整天，最后的解决办法说来很搞笑，就是升级docker就可以了。不过整个查找问题的思路和使用的命令，还是可以记录一下。

# ⚠️问题现象

我写了一个web应用程序，可以采用docker部署。web的前端和后端是独立运行的两个docker container。在docker中创建了一个bridge类型的network，两个container都运行在这个创建的网络中。这样做的目的是让前端可以根据后端的container_name解析出后端的IP地址，然后发送请求给后端。

我的程序在本地部署测试成功后，又部署到了AWS上，运行一切正常。接着尝试把程序运行到Oracle Cloud的虚机上，发现程序无法正常运行，原因是前端container连接后端时超时。前后端无法通信，导致程序退出。

# 🔍查找问题的思路和流程

## ☁️检查cloud security设置

从程序的出错信息中可以看出前端已经正确解析出了后端container的IP地址，但是连接时超时，说明问题就出在前后端的IP通信上。

首先想到的，云服务商通常为运行的虚机只开启最少限度的访问权限。但是这里的docker访问是在虚机内部进程之间的通信，应该不会受到cloud serutity rule的限制。为了证实这一点，在cloud service的控制台中，找到虚机使用的security list。在Ingress rule中增加了一条，打开了所有端口访问权限。同时Egress rule默认就是放行所有通信的。重新运行程序，依然报错，证明确实不是由cloud的security rule导致的。


## 🧱检查防火墙设置

接下来检查是否是虚机自己的防火墙设置导致的。首先是UFW (Uncomplicated Firewall):

```
root@ubuntu1:~# ufw status
Status: inactive
```
ufw并没有运行。接着检查iptable设置：

>iptables -L -v -n --line-numbers | grep -E "DROP|REJECT"

这里有几条貌似和docker以及icmp报文有关，但是无法确定是不是这些规则造成的通信问题，需要测试一下。

## 🐋运行docker

为了验证iptable是否限制了docker container之间的通信，搭建一个简单的环境验证一下。首先创建一个dcoker network：

>docker network create test-network

接着检查创建好的网络：

>docker network inspect test-network

```
...
[
    {
        "Name": "test-network",
        "Id": "4167487351c8c5ef3c4ffe96d16ed6277d276e5930c8380ad15c4f42abeccc3d",
...
```

观察系统的网络接口：

>ip addr

其中的一个接口：

```
...
19: br-4167487351c8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 1e:7f:67:5e:9a:d6 brd ff:ff:ff:ff:ff:ff
    inet 172.19.0.1/16 brd 172.19.255.255 scope global br-4167487351c8

```
ID字符**4167487351c8**和创建的network Id相符，这个就是刚刚创建的接口。它的地址是172.19.0.1。

然后创建两个container并进入到cotainer的shell中：

>docker run --rm -it --network test-network --name test1 nicolaka/netshoot bash
>
>docker run --rm -it --network test-network --name test2 nicolaka/netshoot bash

nicolaka/netshoot是一个带有网络调试工具的image。

此时查看刚才创建的网络：

>docker network inspect test-network

```
...
        "Containers": {
            "4e58959a6a2948ba4c090bf77fd13654e3d984b6d6a2f9966bb8a113b67c669c": {
                "Name": "test1",
                "EndpointID": "0401d7a630aeba277c41438b843bf615beb139722624244d3424aecad8d7ed1b",
                "MacAddress": "72:a3:6e:80:21:04",
                "IPv4Address": "172.19.0.2/16",
                "IPv6Address": ""
            },
            "908336b60c415cd16bc1ecd45b0c64f5706af63ce9aa236273cd10e0ba0c7cfc": {
                "Name": "test2",
                "EndpointID": "2b41af482fb751c5151f0514c9e1ad78273cdcc89a0a9df23b78d42105bdd7a1",
                "MacAddress": "fa:a5:01:55:88:7f",
                "IPv4Address": "172.19.0.3/16",
                "IPv6Address": ""
            }
        },
```

可以看到两个container运行在这个网络中，各自分配了IP地址。接着进入到container test2中。首先测试自己和到网关的IP地址连通性：

>ping 172.19.0.3
>
>ping 172.19.0.1

两个测试都没有问题，接着测试到另一个container：

>ping 172.19.0.2

测试发现不通，和web应用程序运行时的问题一致。在发送ping包的同时，观察iptable：

>iptables -L -v -n --line-numbers | grep -E "DROP|REJECT"

发现与禁止包通行的规则中，包的数量并没有增加，说明ping包并没有触发iptable中的规则。这也就说明了问题不是iptable规则引起的。

为了验证是否创建的docker network有问题，接下来采取对比测试方法。Docker安装时会自动创建一个名为**bridge**的network，接下来就验证在这个网络中是否有相同的问题。

重新启动两个container，这次指定network为**bridge**，即docker自带的网络。测试ping包，发现可以正常和对方container通信。由此感觉是自己创建的network有问题。

# ⚙️检查系统配置和日志

由于怀疑自己创建的network有问题，运行**ip addr**和**docker network inspect**，对比docker自带网络和自己创建网络，但没有发现任何区别。接着检查系统日志，看看是否有报错信息。检查了目录/var/log下的syslog，messages。对比使用两个网络启动container时的日志，仍然没有发现有任何区别。

# ✅️解决问题

至此已经有些困惑了。系统找不但任何禁止container之间通信的配置，也没有任何出错或者告警信息，不太清楚下一步还能做什么。后来突然想到检查一下docker的版本，发现在虚机上运行的版本比我本地的要低，于是升级了docker。然后，问题居然就解决了。

接着重新运行web程序，一切正常。

走了这么多弯路，没想到就是docker版本低造成的😄。

