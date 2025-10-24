最近注册和使用了Oracle Cloud Infrastructure (OCI)。Oracle还是比较大方，提供了一个“Always Free”服务。包含的资源真是多，例如：

* 两台AMD架构的虚机（2核，1G内存，50GB存储）
* SQL和NoSQL数据库
* 一个Load Balancer
* 其他免费资源。。。

这么好的东西，不用太可惜了。注册地址：

[https://www.oracle.com/cloud/](https://www.oracle.com/cloud/)

唯一的要求是，注册时需要一张境外的信用卡。我使用的是香港ZA Bank的虚拟卡。另外，需要填写手机号码。注意填写信用卡所在地的电话号码，只要是格式正确，有效的号码就可以了。这也算是目前的一个福利吧，万一哪天要验证号码，就又多了一点麻烦。

# 创建虚机

注册成功后，当然是先申请一台虚机试试。在OCI控制台的左侧导航中点击""Compute --> Instances --> Create instance""

创建instance时大部分选项不用修改，需要注意的选项有下面这些：

**Image and shape**

选择自己想要的OS，我选择的是Canonical Ubuntu 22.04。

**Shape**

这里一定要选择**Specialty and previous generation**，这个类别里有免费的虚机，选择带有**Always Free-eligible**标志的虚机。如果看不到这个选项，可以尝试修改**Availability domain**，有的AD里面可能不包含免费资源，换一个，再试试。


**Networking**

需要配置SSH的key。在**Add SSH keys**中，选择**Generate a key pair for me**，然后点击**Download private key**，把key文件下载到本地。

其他选项保持默认，创建instance。等待1分钟左右，虚机创建成功。注意虚机创建后，可能没有分配公网IP地址。这时可以点击虚机，切换到**Networking**，在**Attached VNICs**，点击创建好的虚拟网卡，切换到**IP administration**。这里可以看到虚机的一个私有IP地址。点击地址栏最右边的三个点，选择**Edit**。然后选择**Ephemeral public IP**，这样就会为虚机分配公网地址了。

# 配置SSH连接

如果打算从Windows上用Putty连接到虚机，需要配置private key。配置前尽量下载使用最新版的putty.exe和puttygen.exe。我使用的Ubuntu 22.04有一个坑，导致SSH连接一直报错。折腾了一下后找到了解决办法。首先使用puttygen.exe，生成key文件。这里选择**Load**，把刚才创建虚机时下载的private key加载进来，然后注意在最下面key类型中选择**ECDSA**，点击**Save private key**，把key保存成ppk格式。Putty创建新连接时使用该文件，默认登录用户名是**ubuntu**， 就可以SSH到虚机了。

# 端口权限配置

虚机创建时默认只打开22端口用于SSH连接。如果在虚机上运行了其他服务需要远程连接，可以遵循下面的步骤。

首先在控制台增加一条Ingress rule允许端口的访问。在OCI控制台左边导航中点击**Networking --> Virtual Cloud Networks**。在列表中找到创建虚机时指定的VNC，点击它。然后切换到**Security**。在Security Lists中，有一条默认创建好的list，点击它。切换到**Security rules**，这里可以看到ingress rules。点击**Add Ingress Rules**，增加一条规则。至少需要填写两个部分：

>Source CIDR：可以填写0.0.0.0/0
>
>Destination Port Range：填写需要访问的端口号

增加规则后，检查服务是否可以被访问。如果还是不能访问，那么需要检查虚机上的配置。在Ubuntu 22.04中，可以检查iptable的配置：

>iptables -vL INPUT --line-numbers -n

例如输出为：

```
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1     1197  611K ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED
2        0     0 ACCEPT     icmp --  *      *       0.0.0.0/0            0.0.0.0/0
3       35  3030 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0
4       16   720 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            state NEW tcp dpt:22
5        2   696 REJECT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            reject-with icmp-host-prohibited
```

这里可以看到除了允许端口22的访问外，其他请求都会被reject。因此需要添加一条规则。例如需要访问端口12345，就在第4条规则之前增加一条：

>iptables -I INPUT 4 -p tcp --dport 12345 -m conntrack --ctstate NEW -j ACCEPT

这样就可以访问了。

# 使用感受

总的来说，Oracle的云服务不论是用户体验还是功能上，跟AWS相比真是差了一大截。不过，它的Always Free服务承诺的是一直有效。可以常备两台虚机，以后有时间再试试免费数据库和Load Balancer的使用，还是挺值的。