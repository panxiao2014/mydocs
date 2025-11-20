# 三个概念

要读懂iptable，学会查看和修改它的规则，首先要搞懂下面三个概念：

- 三个table
- 五个hook point
- 一堆chain

## table

iptable把不同的功能归类到三个table里，代表了iptable可以做的三种不同类型的事情：

### filter table

这是最常用的table，用来对网络包实施防火墙策略，决定报文是否可以被接收，或者被拒绝。在iptables命令中用 *-t filter* 指定。如果不加 *-t* 参数，则默认都是filter table。

### nat table

用于对数据报文的源地址或者目的地址做转换。在iptables命令中用 *-t nat* 指定。

### mangle table

不常用。用于对数据报文的某些字段做修改。在iptables命令中用 *-t mangle* 指定。

## hook point

Hook point指的是内核在整个报文处理流程中5个不同的插入点。任何iptable的规则都是在这5个hook point的某一个进行执行。要理解iptable的运行机制，关键就是要理解这5个hook point分别发生在处理流程的什么阶段，这样才能确认自己的规则需要添加在什么地方。下面分别解释每个hook point。

### PREROUTING

发生在报文刚刚从系统的某一个网络接口达到，还没有开始被路由规则处理的时候。

### INPUT

由路由规则确定报文是一个'input'报文，即需要发送给本机的一个程序。INPUT hook point发生在报文要被交给本地程序之前。

### FORWARD

由路由规则确定报文是一个'forward'报文，即需要发送给本机的另外一个网络接口。FORWARD hook point发生在报文要被发送到另一个接口之前。

### OUTPUT

由路由规则确定报文是一个'output'报文，即本机的程序产生的报文，需要发送出去。OUTPUT hook point发生在报文在本机创建之后，并准备发送给网络接口之前。

### POSTROUTING

发生在报文已经确定要从某个网络接口发送出去之后，并且正准备离开网络之前。

## chain

iptable里的所有规则都属于某个chain。一个chain又是属于某个table的。当报文到达某个chain时，会按照chain里的每一条规则对其进行匹配，如果匹配上了，就执行这条规则里的策略，匹配不上就看下一条。如果到chain的末尾都没有规则可以匹配上，就执行这个chain的默认规则。通常系统会默认存在下面这些chain：

```
filter: FORWARD, INPUT, OUTPUT

nat: PREROUTING, OUTPUT, POSTROUTING

mangle: PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING
```
这些默认chain，和上面提到的hook point含义是一致的。即是说，在hook point发生时，上面的这些chain就会被执行。

除了这些默认的chain之外，系统里一般还有很多其他的chain，这些是用户或者某些程序在运行时添加的。

# 实例：input报文

有了上面的这些基础概念，下面就通过一个实例来加深理解。我们从一个最典型的场景开始：系统运行一个web服务器，然后一个外部客户端来访问这个服务。通过报文的处理流程，来理解iptable的工作流程，以及规则是怎样执行的。

先启动一个web服务器。我使用python命令，运行一个web服务，并且使用tcp端口号1234来监听：

```
python3 -m http.server 1234
```

在客户端，可以使用curl命令来访问这个服务：

```
curl http://<server IP>:1234
```

这是一个典型的input类型的报文，因为从主机网络接口进来的报文需要发送给本机的程序。对于input报文，iptable的处理流程为：

**mangle: PREROUTING --> nat: PREROUTING --> mangle: INPUT --> filter: INPUT**

下面以我主机的iptable为例，分析处理流程。

首先我们看到input报文要经过的第一个chain是mangle: PREROUTING，我们看一下它里面的内容：

```
root@ubuntu2:~# iptables -n -vL PREROUTING --line-number -t mangle
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
```

这是一个空的chain，没有任何规则。为了验证报文是会经过这个chain的，我们可以给它添加一个规则：

```
iptables -t mangle -A PREROUTING
```

这是一条空规则，没有执行任何操作，我们看看它的效果：

```
root@ubuntu2:~# iptables -n -vL PREROUTING --line-number -t mangle
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1       21  1420            all  --  *      *       0.0.0.0/0            0.0.0.0/0
```

解释一下各个字段：

>num: 规则的编号
>
>pkts：命中这条规则的packet数量
>
>bytes：命中这条规则的packet字节总量
>
>target：对命中规则的packet执行什么操作。这里是空，没有执行任何操作。
>
>prot：匹配报文的协议，包含tcp, udp, icmp, all
>
>opt：匹配一些特殊选项，通常不用，可忽略
>
>in：匹配报文从哪个网络接口进入到主机。*\** 匹配所有接口
>
>out: 匹配报文要从哪个接口发送出去
>
>source: 匹配报文源地址
>
>destination：匹配报文目的地址

我们上面添加的规则，匹配了所有报文，并且没有执行任何操作。但通过多次执行查看命令，我们可以看到pkts和bytes栏数字都在增加，证明了报文到达主机后，首先会被mangle: PREROUTING这个chain处理。

接下来第二个chain是nat: PREROUTING，查看它：

```
root@ubuntu2:~# iptables -n -vL PREROUTING --line-number -t nat
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1    10563  599K DOCKER     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL
```

这里只有一条规则，首先我们注意到规则末尾的"ADDRTYPE match dst-type LOCAL"，它表示规则需要匹配目的地址是本机地址的报文。我们在客户端执行的curl命令是要访问本机的IP地址的，应该会被这条规则匹配上。可以多次执行curl命令，然后查看该chain。我们可以看到，pkts和bytes字段数值都在持续增加。

报文在匹配了这条规则后，iptable对它要执行的操作，即'target'字段的值，是'DOCKER'。系统默认的target有：ACCEPT, DROP, QUEUE, RETURN。由此可以知道，这里的'DOCKER'是一个自定义的chain，它可以由用户或者运行的程序创建。所以对于匹配的报文，iptable会接下来查看DOCKER chain，将它上面的规则应用到报文上。

因为此时我们查看的是nat: PREROUTING，所以接下来就该查看nat: DOCKER:

```
root@ubuntu2:~# iptables -n -vL DOCKER --line-number -t nat
Chain DOCKER (2 references)
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 RETURN     all  --  docker0 *       0.0.0.0/0            0.0.0.0/0
2        0     0 RETURN     all  --  br-4167487351c8 *       0.0.0.0/0            0.0.0.0/0
3        0     0 RETURN     all  --  br-1a00c4d540d7 *       0.0.0.0/0            0.0.0.0/0
4       41  2460 RETURN     all  --  br-af963feb0297 *       0.0.0.0/0            0.0.0.0/0
5      779 46256 DNAT       tcp  --  !br-1a00c4d540d7 *       0.0.0.0/0            0.0.0.0/0            tcp dpt:80 to:172.18.0.3:80
6        1    60 DNAT       tcp  --  !br-af963feb0297 *       0.0.0.0/0            0.0.0.0/0            tcp dpt:9113 to:172.20.0.2:9113
```

这是我主机上的输出，它一共有6条规则。我们先看前四条规则，它们的'in'字段都指定了一个网络接口，即匹配从这些接口收到的报文。这里的网络接口都是由docker服务创建的虚拟接口，而我们用curl命令发送的报文都是从主机的一个物理网络接口接收的，所以不会匹配到这些规则。如果不确定，可以在主机上运行抓包命令：

```
tcpdump -n -i any tcp dst port 1234
```

这里可以查看目的地端口是1234的报文，它们是从哪个接口接收的。

接着看第5和第6条规则，这里的接口是'!br-1a00c4d540d7'，意思是匹配所有不是从接口'br-1a00c4d540d7'接收的包，我们的报文是符合的，但是规则末尾还有：

**tcp dpt:80 to:172.18.0.3:80**

意思是说如果报文目的地端口是80，则做地址转换，把报文的目的地址改为172.18.0.3。我们发送的报文目的端口是1234，因此这两条规则也不会匹配。

这样我们遍历了nat: DOCKER chain的所有规则，没有找到匹配。对于自定义chain，在处理完成后，需要跳回到调用该chain的上一级chain里，继续执行。这里的DOCKERchain是被nat: PREROUTING调用的，所以我们现在重新回到这个chain：

```
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1    10634  603K DOCKER     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL
```

可以看到，执行完DOCkER chain并跳转回来后，chain里已经没有其他规则了，所以这个chain的执行也结束。目前为止curl发送的报文到了主机后还没有被执行任何规则。

接下来是mangle: INPUT：

```
root@ubuntu2:~# iptables -n -vL INPUT --line-number -t mangle
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
```

里面是空的，略过。然后就是filter: INPUT：

```
root@ubuntu2:~# iptables -n -vL INPUT --line-number -t filter
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1     192K   92M ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED
2        0     0 ACCEPT     icmp --  *      *       0.0.0.0/0            0.0.0.0/0
3      731 68129 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0
4     9622  544K ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            state NEW tcp dpt:22
5        0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            state NEW tcp dpt:23833
6        1    60 ACCEPT     tcp  --  *      *       10.0.0.118           0.0.0.0/0            state NEW tcp dpt:9100
7       41  2460 ACCEPT     tcp  --  br-af963feb0297 *       0.0.0.0/0            0.0.0.0/0            tcp dpt:80
8       47  2820 ACCEPT     tcp  --  *      *       10.0.0.118           0.0.0.0/0            state NEW tcp dpt:9113
9      570  194K REJECT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            reject-with icmp-host-prohibited
```

一共9条规则。先看第一条：如果一个报文属于一个已经创建好或者关联的链接，则直接放行（ACCEPT）。这里的'ESTABLISHED'和'RELATED'，是由系统维护的链接。例如创建好的TCP链接，有双向数据流通的UDP报文等等。我们发送的报文需要经过TCP三次握手才能创建好链接，因此目前还不匹配这条规则。

第二条匹配icmp报文，第三条匹配从本机loopback接收到的报文，都不能匹配我们的curl测试报文。

第四条到第八条，需要配置的目的地址端口都不是我们发送的报文的目的地址端口，也都不匹配。

接着看第九条，它匹配了所有报文，所以这时我们的报文命中了，然后执行规则'REJECT'，这导致了我们的curl命令执行失败。我们可以多执行几次，可以看到第九条规则的pkts和bytes字段值在持续增加。

理解了上面整个流程后，我们知道：只要在filter: INPUT里增加一条规则，允许端口1234的访问，我们的外部访问就可以成功。这里要仔细查看规则表，确定把规则加在哪个位置。这里我们在第四条规则之前增加一条：

```
iptables -t filter -I INPUT 4 -p tcp --dport 1234 -m conntrack --ctstate NEW -j ACCEPT
```

现在再执行curl命令，就可以成功访问web服务器了。同时可以查看，新增加的规则pkts和bytes数量开始增加，也说明我们的报文匹配到了这个规则。

# 实例：访问docker服务

接下来我们看一个稍微复杂一点的，但也是常见的实例：从外部访问主机上运行在docker里的服务。首先我们把刚刚创建的规则删掉：

```
iptables -t filter -D INPUT 4
```

然后启动一个跑在docker里的web服务，用1234端口做监听：

```
docker run --rm --name myweb -p 1234:1234 nginx \
  /bin/bash -c "sed -i 's/listen       80;/listen 1234;/' /etc/nginx/conf.d/default.conf && nginx -g 'daemon off;'"
```

上面的命令会把docker container内的1234端口映射到主机的1234端口，所以我们在客户端可以使用相同的curl命令来访问这个服务：

```
curl http://<server IP>:1234
```

命令执行成功。为什么我们刚刚删除了允许端口1234访问的规则，但是curl命令还是可以执行成功呢？下面就来分析一下报文的处理流程。

由于我们使用curl命令，访问相同的IP地址和端口，因此报文进入主机时和第一种实例的情况是一样的。同样会经过mangle: PREROUTING和nat: PREROUTING的处理。回顾nat: PREROUTING的规则：

```
root@ubuntu2:~# iptables -n -vL PREROUTING --line-number -t nat
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1    11398  653K DOCKER     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL
```

和之前一样，这时要跳转到nat: DOCKER chain里继续处理：

```
root@ubuntu2:~# iptables -n -vL DOCKER --line-number -t nat
Chain DOCKER (2 references)
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 RETURN     all  --  docker0 *       0.0.0.0/0            0.0.0.0/0
2        0     0 RETURN     all  --  br-4167487351c8 *       0.0.0.0/0            0.0.0.0/0
3        0     0 RETURN     all  --  br-1a00c4d540d7 *       0.0.0.0/0            0.0.0.0/0
4       45  2700 RETURN     all  --  br-af963feb0297 *       0.0.0.0/0            0.0.0.0/0
5      862 51236 DNAT       tcp  --  !br-1a00c4d540d7 *       0.0.0.0/0            0.0.0.0/0            tcp dpt:80 to:172.18.0.3:80
6        1    60 DNAT       tcp  --  !br-af963feb0297 *       0.0.0.0/0            0.0.0.0/0            tcp dpt:9113 to:172.20.0.2:9113
7       35  1820 DNAT       tcp  --  !docker0 *       0.0.0.0/0            0.0.0.0/0            tcp dpt:1234 to:172.17.0.2:1234
```

注意到第七条rule是新添加的。当我们执行docker run命令，并指定了'-p 1234:1234'参数时，docker会自动在这里添加一条rule。这条rule的意思是：如果报文的目的地端口是1234，并且报文不是从docker0这个端口接收到的，则把报文的目的地地址改为172.17.0.2:1234，即实现了NAT功能。我们的报文是匹配这条规则的，如果我们多次执行curl命令，会发现pkts和bytes都在增加。

172.17.0.2这个地址，是docker container运行起来后，docker内部网络接口的地址。因此现在报文的目的地地址被修改了，然后nat: DOCKER chain执行完成，流程跳回到nat: PREROUTING。这个chain同样也执行完成，需要进入到下一个chain。

这里进入关键的要点。在第一个实例中，input报文接下来要遍历mangle: INPUT和filter: INPUT。而在此处，报文经过NAT出来后，已经不再是一个input报文了。现在主机要决定的是，一个目的地地址是172.17.0.2的报文应该往哪里送。我们看一下路由表：

```
root@ubuntu2:~# ip route
default via 10.0.0.1 dev ens3
default via 10.0.0.1 dev ens3 proto dhcp src 10.0.0.47 metric 100
10.0.0.0/24 dev ens3 proto kernel scope link src 10.0.0.47 metric 100
10.0.0.1 dev ens3 proto dhcp scope link src 10.0.0.47 metric 100
169.254.0.0/16 dev ens3 scope link
169.254.0.0/16 dev ens3 proto dhcp scope link src 10.0.0.47 metric 100
169.254.169.254 via 10.0.0.1 dev ens3 proto dhcp src 10.0.0.47 metric 100
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1
172.18.0.0/16 dev br-1a00c4d540d7 proto kernel scope link src 172.18.0.1
172.19.0.0/16 dev br-4167487351c8 proto kernel scope link src 172.19.0.1 linkdown
172.20.0.0/16 dev br-af963feb0297 proto kernel scope link src 172.20.0.1
```

其中这一条：

>**172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1**

它表示如果报文的目的地地址是172.17.0.0/16,则把报文发送到docker0接口。我们的curl报文在经过地址转换后是符合这条规则的，因此系统决定把它发送到docker0接口。

我们用'ip addr'命令，可以看到docker0的信息：

```
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether a6:f5:e5:8e:a8:4e brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
```

它是由docker服务创建的一个虚拟接口，地址是172.17.0.1。我们报文的目的地地址，172.17.0.2，即位于这个接口所在的网络中。docker有它自己独立的网络接口，虽然它创建的接口是虚拟的，但对主机来说，这就是一个把报文从一个接口转发到另一个接口。

因此，现在对于报文，已经变成了一个forward类型的报文：它从主机的网络接口接收，并且要转发给系统的另一个接口。它不再是一个input报文了，这就是为什么我们在filter: INPUT chain里，即使没有配置允许端口1234访问的规则也不会影响报文的接收，因为系统已经不再会走到这个chain里去了。

对于forward报文，iptable的处理流程为：

**mangle: PREROUTING --> nat: PREROUTING --> mangle: FORWARD --> filter: FORWARD --> mangle: POSTROUTING --> nat: POSTROUTING**

前面两个chain这时已经走完了，接下来是mangle: FORWARD：

```
root@ubuntu2:~# iptables -n -vL FORWARD --line-number -t mangle
Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
```

这时一个空的chain，再看下一个，filter: FORWARD：

```
root@ubuntu2:~# iptables -n -vL FORWARD --line-number -t filter
Chain FORWARD (policy DROP 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1    51189   20M DOCKER-USER  all  --  *      *       0.0.0.0/0            0.0.0.0/0
2    51189   20M DOCKER-FORWARD  all  --  *      *       0.0.0.0/0            0.0.0.0/0
3        0     0 REJECT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            reject-with icmp-host-prohibited
```

第一条rule显然是匹配的，于是跳到filter: DOCKER-USER chain:

```
root@ubuntu2:~# iptables -n -vL DOCKER-USER --line-number -t filter
Chain DOCKER-USER (1 references)
num   pkts bytes target     prot opt in     out     source               destination
```

这又是一张空表，跳回到filter: FORWARD，然后接着看第二条规则，也是匹配的，于是跳到filter: DOCKER-FORWARD：

```
root@ubuntu2:~# iptables -n -vL DOCKER-FORWARD --line-number -t filter
Chain DOCKER-FORWARD (1 references)
num   pkts bytes target     prot opt in     out     source               destination
1    51245   20M DOCKER-CT  all  --  *      *       0.0.0.0/0            0.0.0.0/0
2    23552   17M DOCKER-ISOLATION-STAGE-1  all  --  *      *       0.0.0.0/0            0.0.0.0/0
3    23552   17M DOCKER-BRIDGE  all  --  *      *       0.0.0.0/0            0.0.0.0/0
4    16345   15M ACCEPT     all  --  br-af963feb0297 *       0.0.0.0/0            0.0.0.0/0
5     6232 1140K ACCEPT     all  --  br-1a00c4d540d7 *       0.0.0.0/0            0.0.0.0/0
6        0     0 ACCEPT     all  --  br-4167487351c8 *       0.0.0.0/0            0.0.0.0/0
7       50  2000 ACCEPT     all  --  docker0 *       0.0.0.0/0            0.0.0.0/0
```

第一条规则匹配所有报文，所以继续跳转，来到filter: DOCKER-CT：

```
root@ubuntu2:~# iptables -n -vL DOCKER-CT --line-number -t filter
Chain DOCKER-CT (1 references)
num   pkts bytes target     prot opt in     out     source               destination
1    21787 3102K ACCEPT     all  --  *      br-af963feb0297  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
2     5916  664K ACCEPT     all  --  *      br-1a00c4d540d7  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
3        0     0 ACCEPT     all  --  *      br-4167487351c8  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
4        0     0 ACCEPT     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
```

前面三条匹配目的网络接口是'br-'开头的网络，我们的报文是要发送到docker0的网络接口，因此不匹配。第四条，是匹配发送到docker0的，但同时还需要是RELATED或者ESTABLISHED的报文，我们发送的报文是从TCP三次握手开始的初始报文，也不匹配。因此DOCKER-CT走完，没有执行任何规则，跳回到filter:DOCKER-FORWARD。接着看该chain里的下一条：

```
2    23552   17M DOCKER-ISOLATION-STAGE-1  all  --  *      *       0.0.0.0/0            0.0.0.0/0
```

匹配所有报文，于是查看filter: DOCKER-ISOLATION-STAGE-1：

```
root@ubuntu2:~# iptables -n -vL DOCKER-ISOLATION-STAGE-1 --line-number -t filter
Chain DOCKER-ISOLATION-STAGE-1 (1 references)
num   pkts bytes target     prot opt in     out     source               destination
1    16441   15M DOCKER-ISOLATION-STAGE-2  all  --  br-af963feb0297 !br-af963feb0297  0.0.0.0/0            0.0.0.0/0
2     6257 1143K DOCKER-ISOLATION-STAGE-2  all  --  br-1a00c4d540d7 !br-1a00c4d540d7  0.0.0.0/0            0.0.0.0/0
3        0     0 DOCKER-ISOLATION-STAGE-2  all  --  br-4167487351c8 !br-4167487351c8  0.0.0.0/0            0.0.0.0/0
4       50  2000 DOCKER-ISOLATION-STAGE-2  all  --  docker0 !docker0  0.0.0.0/0            0.0.0.0/0
```

这里所有规则要求匹配报文是从'br-'或者docker0进入的报文，我们的报文是从主机物理端口进入的，都不匹配。因此跳回到filter: DOCKER-FORWARD，看下一条：

```
3    23552   17M DOCKER-BRIDGE  all  --  *      *       0.0.0.0/0            0.0.0.0/0
```

还是匹配所有报文，于是查看filter: DOCKER-BRIDGE：

```
root@ubuntu2:~# iptables -n -vL DOCKER-BRIDGE --line-number -t filter
Chain DOCKER-BRIDGE (1 references)
num   pkts bytes target     prot opt in     out     source               destination
1        2   120 DOCKER     all  --  *      br-af963feb0297  0.0.0.0/0            0.0.0.0/0
2      881 52376 DOCKER     all  --  *      br-1a00c4d540d7  0.0.0.0/0            0.0.0.0/0
3        0     0 DOCKER     all  --  *      br-4167487351c8  0.0.0.0/0            0.0.0.0/0
4       60  3120 DOCKER     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0
```

第四条规则，匹配的是发送到网络接口docker0的报文，我们的报文在经过NAT地址转换和路由规则处理后，正是发送到docker0的，因此匹配了该规则。反复执行curl操作，也可以看到这条规则的pkts和bytes计数在增加。这条规则执行的操作是跳转到fiter: DOCKER chain。查看该chain：

```
root@ubuntu2:~# iptables -n -vL DOCKER --line-number -t filter
Chain DOCKER (4 references)
num   pkts bytes target     prot opt in     out     source               destination
1       80  4160 ACCEPT     tcp  --  !docker0 docker0  0.0.0.0/0            172.17.0.2           tcp dpt:1234
2        1    60 ACCEPT     tcp  --  !br-af963feb0297 br-af963feb0297  0.0.0.0/0            172.20.0.2           tcp dpt:9113
3      886 52676 ACCEPT     tcp  --  !br-1a00c4d540d7 br-1a00c4d540d7  0.0.0.0/0            172.18.0.3           tcp dpt:80
4        0     0 DROP       all  --  !br-af963feb0297 br-af963feb0297  0.0.0.0/0            0.0.0.0/0
5        0     0 DROP       all  --  !br-1a00c4d540d7 br-1a00c4d540d7  0.0.0.0/0            0.0.0.0/0
6        0     0 DROP       all  --  !br-4167487351c8 br-4167487351c8  0.0.0.0/0            0.0.0.0/0
7        0     0 DROP       all  --  !docker0 docker0  0.0.0.0/0            0.0.0.0/0
```

第一条rule：凡是不是从docker0接收到的报文，并且是发送到docker0的，同时目的地地址是172.17.0.2:1234, 则执行ACCEPT操作。我们的报文匹配了这条规则，这里执行了ACCEPT后，报文在filter: FORWARD中的处理立即结束，不用再跳转回调用的chain。

对于forward类型的报文，接着还有两个chain要走：

**mangle: POSTROUTING --> nat: POSTROUTING**。

先看mangle: POSTROUTING：

```
root@ubuntu2:~# iptables -n -vL POSTROUTING --line-number -t mangle
Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
```
是空的，然后看nat: POSTROUTING：

```
root@ubuntu2:~# iptables -n -vL POSTROUTING --line-number -t nat
Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 MASQUERADE  all  --  *      !docker0  172.17.0.0/16        0.0.0.0/0
2        0     0 MASQUERADE  all  --  *      !br-4167487351c8  172.19.0.0/16        0.0.0.0/0
3        6   344 MASQUERADE  all  --  *      !br-1a00c4d540d7  172.18.0.0/16        0.0.0.0/0
4        0     0 MASQUERADE  all  --  *      !br-af963feb0297  172.20.0.0/16        0.0.0.0/0
```

这里要匹配的是发送的目的端口不是docker0或者'br-'的端口。我们的目的地端口是docker0，不匹配。

最终，iptable的流程走完，报文接受了ACCEPT的规则，因此我们的curl命令可以顺利执行完成。

# 总结

上面的流程看似比较复杂，在各个不同的chain里跳来跳去，不过只要掌握好规则，还是可以一步步把流程理清的。

下面总结一下分析iptable流程的步骤。首先，判断报文是什么类型的报文，这里有四种类型：

- input: 报文从某一个网络接口接收，并准备发送给本地程序
- forward: 报文从某一个网络接口接收，并准备发送给另一个接口
- output: 报文由一个本地程序产生，并准备发送给一个网络接口
- local: 报文由一个本地程序产生，并准备发给本地另一个程序

然后，对于这四种类型的报文，它们的默认iptable处理链分别为：

## input

mangle: PREROUTING --> nat: PREROUTING --> mangle: INPUT --> filter: INPUT

## forward

mangle: PREROUTING --> nat: PREROUTING --> mangle: FORWARD --> filter: FORWARD --> mangle: POSTROUTING --> nat: POSTROUTING

## output

mangle: OUTPUT --> nat: OUTPUT --> filter: OUTPUT --> mangle: POSTROUTING --> nat: POSTROUTING

## local

mangle: OUTPUT --> nat: OUTPUT --> filter: OUTPUT --> filter: INPUT --> mangle: INPUT

在确定了报文类型和对应的默认处理链后，就可以开始分析报文。注意几个点：

1. 尽量简化网络配置，减少运行的程序，从而方便后继的分析。
2. 配合tcpdump工具观察各个接口的报文，效果更佳。
3. 善用各个chain里各个规则的ptks和bytes计数，用它来观察规则有没有被执行。

然后从各个类型报文默认处理链的一个chain开始，逐步分析。遇到跳转到用户自定义的chain时，注意遍历完后又要回到调用的chain，继续下一个规则的处理。这个类似函数的调用和返回。



