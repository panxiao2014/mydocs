# EC2设置
EC2上需要把v2ray server端监听的端口号打开。在运行的instance页面，左边选择Network & Security --> Security Groups。选择要使用的Security group ID，点击'Edit inbound rules'。增加一条rule：
>Type: Custom TCP
>
>Port range: 16832
>
>Source: Custom 0.0.0.0/0
>
点击'Save rules'。

# Server端配置
## 安装v2ray server
>bash <(curl -L https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh)
>
>bash <(curl -L https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-dat-release.sh)
>

##配置server
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
