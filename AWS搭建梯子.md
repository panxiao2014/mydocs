# EC2设置
EC2上需要把v2ray server端监听的端口号打开。在运行的instance页面，左边选择*Network & Security --> Security Groups*。选择要使用的Security group ID，点击*Edit inbound rules*。增加一条rule：
>Type: Custom TCP
>
>Port range: 16832
>
>Source: Custom 0.0.0.0/0
>
点击'Save rules'。注意这里的端口设置，必须和以下server和client的端口一致。

# Server端配置
## 安装v2ray server
>bash <(curl -L https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh)
>
>bash <(curl -L https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-dat-release.sh)
>
## 配置server
编辑文件/usr/local/etc/v2ray/config.json。示例：
```json
{
  "log": {
    "loglevel": "debug", //级别可以为debug, info, warning, error, none
    "access": "/var/log/v2ray/access.log", // 这是 Linux 的路径
    "error": "/var/log/v2ray/error.log"
  },
  "inbounds": [
    {
      "port": 16832, // 服务器监听端口
      "protocol": "vmess",    // 主传入协议
      "settings": {
        "clients": [
          {
            "id": "xxx",  // 用户 ID，客户端与服务器必须相同。使用UUID生成器创建。
            "alterId": 0
          }
        ]
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",  // 主传出协议
      "settings": {}
    }
  ]
}
```

## 启动server
>systemctl start v2ray
>
# Windows client设置
## 安装client
[https://github.com/v2fly/v2ray-core/releases](https://github.com/2dust/v2rayN/releases)

下载*v2rayN-windows-64-SelfContained.zip*

## 配置client
运行v2rayN.exe。选择*配置文件 --> 添加[VMess]配置文件*。配置以下内容：
>别名：任意取一个名字
>
>地址：运行v2ray server的EC2 instance地址
>
>端口：server端配置的端口
>
>用户ID：server端创建的UUID
>
>额外ID：server端配置的alterId
>
其他配置保持默认选项。

在*设置 --> 参数设置 --> Core:基础设置*中，可以调整日志等级。

## 运行client
在窗口下面，系统代理中，选择'自动配置系统代理'。
