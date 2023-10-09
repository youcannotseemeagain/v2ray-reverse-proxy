# 这是什么
这是使用v2ray的vmess协议进行内网反向全局代理的使用教程。

# 起源
近几年，由于nps和frp的流量已经被各种流量监控设备锁死了，某次经高人提醒便萌生出了使用v2ray进行反向代理的想法。

# 下载安装
根据官方所说`反向代理功能在 V2Ray 4.0+ 可用`，但根据测试，我建议安装5.0以上的版本使用，不然会有一些莫名其妙的问题。

5.0.6版本下载地址
https://github.com/v2fly/v2ray-core/releases/tag/v5.0.6

我们只需要其中的`v2ray`可执行文件。

# 如何实现
在v2ray中有三个角色，可以简单的称之为`被控端`、`服务端`和`客户端`。
通过执行`v2ray.exe run --config xx.json`命令可以启动v2ray，它的角色由json文件定义。

要实现内网穿透，我们需要将**被控端运行在被代理的内网机器**中，将**服务端运行在vps**上，**客户端可以运行在本地或者vps**，建议运行在vps。
最后通过连接客户端的socks代理可以穿透到内网机器的网络中。


整条流量的方向是

本机——socks协议——>客户端——vmess协议——>服务端——vmess协议——>被控端

json文件中关键的几项配置是IP、端口和UUID。
客户端与服务端的配置要符合，例如客户端的outbounds中的ip、端口、协议和UUID要和服务端inbounds的IP、端口、协议和UUID相同。
被控端和服务端同理。

下面是一些示例的配置，使用时只需要修改相应IP、端口和UUID。
# 配置

## 单个端口转发

### 服务端

```text-plain
{  
  "reverse":{
    "portals":[  
      {  
        "tag":"portal",
        "domain":"www.baidu.com"
      }
    ]
  },
  "inbounds": [
    {  
      "tag":"external", 
      "port":80,
      "protocol":"dokodemo-door",
        "settings":{  
          "address":"127.0.0.1",
          "port":80,
          "network":"tcp"
        }
    },
    {  
      "tag":"external1", 
      "port":808,        //这里是绑定的本地port
      "protocol":"dokodemo-door",
        "settings":{  
          "address":"127.0.0.1",
          "port":808,        //这里是远程的port
          "network":"tcp"
        }
    },
    {
      "tag": "tunnel",
      "port":16824,
      "protocol":"vmess",
      "settings":{  
        "clients":[  
          {  
            "id":"b831381d-6324-4d53-ad4f-8cda48b30811",
            "alterId":0
          }
        ]
      }
    }
  ],
  "routing":{  
    "rules":[  
      {  
        "type":"field",
        "inboundTag":[  
          "external","external1"
        ],
        "outboundTag":"portal"
      },
      {  
        "type":"field",
        "inboundTag":[  
          "tunnel"
        ],
        "domain":[  
          "full:www.baidu.com"
        ],
        "outboundTag":"portal"
      }
    ]
  }
}
```

### 被控端

```text-plain
{  
  "reverse":{ 
    "bridges":[  
      {  
        "tag":"bridge", 
        "domain":"www.baidu.com" 
      }
    ]
  },
  "outbounds": [
    {  
      "tag":"tunnel", 
      "protocol":"vmess",
      "settings":{  
        "vnext":[  
          {  
            "address":"1.1.1.1",        //通讯IP
            "port":16824,            //通讯端口
            "users":[  
              {  
                "id":"b831381d-6324-4d53-ad4f-8cda48b30811",
                "alterId":0
              }
            ]
          }
        ]
      }
    },
    {  
      "protocol":"freedom",
      "settings":{ 
      },
      "tag":"out"
    }
  ],
  "routing":{   
    "rules":[  
      {  
        "type":"field",
        "inboundTag":["bridge"],
        "domain":["full:www.baidu.com"],
        "outboundTag":"tunnel"
      },
      {  
        "type":"field",
        "inboundTag":["bridge"],
        "outboundTag":"out"
      }
    ]
  }
}
```

## TCP/KCP的全局代理

服务端、被控端、客户端端都需要运行v2ray，这里的配置开启了kcp，可以只开启服务端到被控端的kcp，kcp实际上是udp协议。

### 服务端

```text-plain
{  
  "reverse":{
    "portals":[  
      {  
        "tag":"portal",
        "domain":"www.baidu.com"
      }
    ]
  },
  "inbounds": [
    {
      "tag": "tunnel1",
      "port":16825,
      "protocol":"vmess",
      "settings":{  
        "clients":[  
          {  
            "id":"364bb82c-e6d3-4769-b5ca-588cc82266c7",  
            "alterId": 0
          }
        ]
      },
    "streamSettings": {
      "network": "kcp",   
      "kcpSettings": {
        "mtu": 1350,
        "tti": 20,
        "uplinkCapacity": 5,
        "downlinkCapacity": 100,
        "congestion": true,
        "readBufferSize": 1,
        "writeBufferSize": 1,
        "header": {
          "type": "none"
        }
      }
    }
    },
    {
      "tag": "tunnel",
      "port":16824,   
      "protocol":"vmess",
      "settings":{  
        "clients":[  
          {  
            "id":"b831381d-6324-4d53-ad4f-8cda48b30811",   
            "alterId":0
          }
        ]
      },
    "streamSettings": {
      "network": "kcp",    
      "kcpSettings": {
        "mtu": 1350,
        "tti": 20,
        "uplinkCapacity": 5,
        "downlinkCapacity": 100,
        "congestion": true,
        "readBufferSize": 1,
        "writeBufferSize": 1,
        "header": {
          "type": "none"
        }
      }
    }
    }
  ],
  "routing":{  
    "rules":[  
      {  
        "type":"field",
        "inboundTag":[  
          "tunnel1"
        ],
        "outboundTag":"portal"
      },
      {  
        "type":"field",
        "inboundTag":[  
          "tunnel"
        ],
        "outboundTag":"portal"
      }
    ]
  }
}
```

### 被控端

```text-plain
{  
  "reverse":{ 
    "bridges":[  
      {  
        "tag":"bridge", 
        "domain":"www.baidu.com" 
      }
    ]
  },
  "outbounds": [
    {  
      "tag":"tunnel", 
      "protocol":"vmess",
      "settings":{  
        "vnext":[  
          {  
            "address":"1.1.1.1",
            "port":16824,
            "users":[  
              {  
                "id":"b831381d-6324-4d53-ad4f-8cda48b30811",
                "alterId":0
              }
            ]
          }
        ]
      },
    "streamSettings": {
      "network": "kcp",
      "kcpSettings": {
        "mtu": 1350,
        "tti": 20,
        "uplinkCapacity": 5,
        "downlinkCapacity": 100,
        "congestion": true,
        "readBufferSize": 1,
        "writeBufferSize": 1,
        "header": {
          "type": "none"
        }
      }
    }
    },
    {  
      "protocol":"freedom",
      "settings":{},
      "tag":"out"
    }
  ],
  "routing":{   
    "rules":[  
      {  
        "type":"field",
        "inboundTag":["bridge"],
        "domain":["full:www.baidu.com"],
        "outboundTag":"tunnel"
      },
      {  
        "type":"field",
        "inboundTag":["bridge"],
        "outboundTag":"out"
      }
    ]
  }
}
```

### 客户端

```text-plain
{
  "log": {
    "access": "",
    "error": "",
    "loglevel": "warning"
  },
  "inbounds": [  
    {  
      "port": 1080,  
      "ip": "0.0.0.0",  
      "protocol": "socks",  
      "version": 5,  
      "settings": {  
        "auth": "password",  
        "accounts": [  
        {  
            "user": "123456",  
            "pass": "123456"  
        }  
        ]  
    }  
    }  
  ],
  "outbounds": [
    {
      "protocol": "vmess",
      "settings": {
        "vnext": [
          {
            "address": "1.1.1.1",
            "port": 16825,
            "users": [
              {
                "id": "364bb82c-e6d3-4769-b5ca-588cc82266c7",
                "alterId": 0
              }
            ]
          }
        ]
      },
    "streamSettings": {
      "network": "kcp",
      "kcpSettings": {
        "mtu": 1350,
        "tti": 20,
        "uplinkCapacity": 5,
        "downlinkCapacity": 100,
        "congestion": true,
        "readBufferSize": 1,
        "writeBufferSize": 1,
        "header": {
          "type": "none"
        }
      }
    }
    }
  ]
}
```

## TCP/WS+TLS加密的全局代理

需要为自己的域名注册一个证书来使用

### 服务端

```text-plain
{  
  "reverse":{
    "portals":[  
      {  
        "tag":"portal",
        "domain":"www.baidu.com"
      }
    ]
  },
  "inbounds": [
    {
      "tag": "tunnel1",    //这个inbound属于客户端与服务端的连接，可以不用管
      "port":16825,
      "protocol":"vmess",
      "settings":{  
        "clients":[  
          {  
            "id":"364bb82c-e6d3-4769-b5ca-588cc82266c7",
            "alterId": 0
          }
        ]
      }
    },
    {
      "tag": "tunnel",    ////这个inbound属于服务端与被控端的连接，需要加密
      "port":443,
      "protocol":"vmess",
      "settings":{  
        "clients":[  
          {  
            "id":"b831381d-6324-4d53-ad4f-8cda48b30811",
            "alterId":0
          }
        ]
      },
      "streamSettings": {
        "network": "ws",   //可以改tcp，但没必要改kcp
        "security": "tls",
      "tlsSettings": {
        "certificates": [
          {
            "certificateFile": "/tmp/full_chain.pem",  //Lets Encrypt要用full_chain
            "keyFile": "/tmp/v2ray.key"
          }
        ]
      }
      }
    }
  ],
  "routing":{  
    "rules":[  
      {  
        "type":"field",
        "inboundTag":[  
          "tunnel1"
        ],
        "outboundTag":"portal"
      },
      {  
        "type":"field",
        "inboundTag":[  
          "tunnel"
        ],
        "outboundTag":"portal"
      }
    ]
  }
}
```

### 被控端

```text-plain
{  
  "reverse":{ 
    "bridges":[  
      {  
        "tag":"bridge", 
        "domain":"www.baidu.com" 
      }
    ]
  },
  "outbounds": [
    {  
      "tag":"tunnel", 
      "protocol":"vmess",
      "settings":{  
        "vnext":[  
          {  
            "address":"1.1.1.1",    //自己的域名
            "port":443,
            "users":[  
              {  
                "id":"b831381d-6324-4d53-ad4f-8cda48b30811",
                "alterId":0
              }
            ]
          }
        ]
      },
      "streamSettings": {
        "network": "ws",        //保持与服务端一致
        "security": "tls"        //保持与服务端一致
      }
    },
    {  
      "protocol":"freedom",
      "settings":{},
      "tag":"out"
    }
  ],
  "routing":{   
    "rules":[  
      {  
        "type":"field",
        "inboundTag":["bridge"],
        "domain":["full:www.baidu.com"],
        "outboundTag":"tunnel"
      },
      {  
        "type":"field",
        "inboundTag":["bridge"],
        "outboundTag":"out"
      }
    ]
  }
}
```

### 客户端

这里的端口随意了，启动后会本地开一个S5代理，没有设置认证

```text-plain
{
  "log": {
    "access": "",
    "error": "",
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "protocol": "socks",
      "port": 10809,
      "listen": "127.0.0.1",
      "settings": {
        "auth": "noauth",
        "accounts": [],
        "udp": false,
        "userLevel": 0
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "vmess",
      "settings": {
        "vnext": [
          {
            "address": "1.1.1.1",
            "port": 16825,
            "users": [
              {
                "id": "364bb82c-e6d3-4769-b5ca-588cc82266c7",
                "alterId": 0
              }
            ]
          }
        ]
      }
    }
  ]
}
```

## 仅KCP

### 服务端

```text-plain
{  
  "reverse":{
    "portals":[  
      {  
        "tag":"portal",
        "domain":"www.baidu.com"
      }
    ]
  },
  "inbounds": [
    {
      "tag": "tunnel1",
      "port":16825,
      "protocol":"vmess",
      "settings":{  
        "clients":[  
          {  
            "id":"364bb82c-e6d3-4769-b5ca-588cc82266c7",
            "alterId": 0
          }
        ]
      },
    "streamSettings": {
      "network": "tcp"
    }
    },
    {
      "tag": "tunnel",
      "port":53,
      "protocol":"vmess",
      "settings":{  
        "clients":[  
          {  
            "id":"b831381d-6324-4d53-ad4f-8cda48b30811",
            "alterId":0
          }
        ]
      },
    "streamSettings": {
      "network": "kcp",
      "kcpSettings": {
        "mtu": 1350,
        "tti": 20,
        "uplinkCapacity": 5,
        "downlinkCapacity": 100,
        "congestion": true,
        "readBufferSize": 1,
        "writeBufferSize": 1,
        "header": {
          "type": "none"
        }
      }
    }
    }
  ],
  "routing":{  
    "rules":[  
      {  
        "type":"field",
        "inboundTag":[  
          "tunnel1"
        ],
        "outboundTag":"portal"
      },
      {  
        "type":"field",
        "inboundTag":[  
          "tunnel"
        ],
        "outboundTag":"portal"
      }
    ]
  }
}
```

### 被控端

```text-plain
{  
  "reverse":{ 
    "bridges":[  
      {  
        "tag":"bridge", 
        "domain":"www.baidu.com" 
      }
    ]
  },
  "outbounds": [
    {  
      "tag":"tunnel", 
      "protocol":"vmess",
      "settings":{  
        "vnext":[  
          {  
            "address":"1.1.1.1",
            "port":53,
            "users":[  
              {  
                "id":"b831381d-6324-4d53-ad4f-8cda48b30811",
                "alterId":0
              }
            ]
          }
        ]
      },
    "streamSettings": {
      "network": "kcp",
      "kcpSettings": {
        "mtu": 1350,
        "tti": 20,
        "uplinkCapacity": 5,
        "downlinkCapacity": 100,
        "congestion": true,
        "readBufferSize": 1,
        "writeBufferSize": 1,
        "header": {
          "type": "none"
        }
      }
    }
    },
    {  
      "protocol":"freedom",
      "settings":{},
      "tag":"out"
    }
  ],
  "routing":{   
    "rules":[  
      {  
        "type":"field",
        "inboundTag":["bridge"],
        "domain":["full:www.baidu.com"],
        "outboundTag":"tunnel"
      },
      {  
        "type":"field",
        "inboundTag":["bridge"],
        "outboundTag":"out"
      }
    ]
  }
}
```

### 客户端

```text-plain
{
  "log": {
    "access": "",
    "error": "",
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "port": 10901,
      "ip": "0.0.0.0",
      "protocol": "socks",
      "version": 5,
      "settings": {
        "auth": "password",
        "accounts": [
        {
            "user": "123123",
            "pass": "123123"
        }
        ]
    }
    }
  ],
  "outbounds": [
    {
      "protocol": "vmess",
      "settings": {
        "vnext": [
          {
            "address": "127.0.0.1",
            "port": 16825,
            "users": [
              {
                "id": "364bb82c-e6d3-4769-b5ca-588cc82266c7",
                "alterId": 0
              }
            ]
          }
        ]
      },
    "streamSettings": {
      "network": "tcp"
    }
    }
  ]
}
```

## 仅TCP

### 服务端

```text-plain
{  
  "reverse":{
    "portals":[  
      {  
        "tag":"portal",
        "domain":"www.baidu.com"
      }
    ]
  },
  "inbounds": [
    {
      "tag": "tunnel1",
      "port":16825,
      "protocol":"vmess",
      "settings":{  
        "clients":[  
          {  
            "id":"364bb82c-e6d3-4769-b5ca-588cc82266c7",
            "alterId": 0
          }
        ]
      },
    "streamSettings": {
      "network": "tcp"
    }
    },
    {
      "tag": "tunnel",
      "port":53,
      "protocol":"vmess",
      "settings":{  
        "clients":[  
          {  
            "id":"b831381d-6324-4d53-ad4f-8cda48b30811",
            "alterId":0
          }
        ]
      },
    "streamSettings": {
      "network": "tcp"    
    }
    }
  ],
  "routing":{  
    "rules":[  
      {  
        "type":"field",
        "inboundTag":[  
          "tunnel1"
        ],
        "outboundTag":"portal"
      },
      {  
        "type":"field",
        "inboundTag":[  
          "tunnel"
        ],
        "outboundTag":"portal"
      }
    ]
  }
}
```

### 被控端

```text-plain
{  
  "reverse":{ 
    "bridges":[  
      {  
        "tag":"bridge", 
        "domain":"www.baidu.com" 
      }
    ]
  },
  "outbounds": [
    {  
      "tag":"tunnel", 
      "protocol":"vmess",
      "settings":{  
        "vnext":[  
          {  
            "address":"1.1.1.1",
            "port":53,
            "users":[  
              {  
                "id":"b831381d-6324-4d53-ad4f-8cda48b30811",
                "alterId":0
              }
            ]
          }
        ]
      },
    "streamSettings": {
      "network": "tcp"      
    }
    },
    {  
      "protocol":"freedom",
      "settings":{},
      "tag":"out"
    }
  ],
  "routing":{   
    "rules":[  
      {  
        "type":"field",
        "inboundTag":["bridge"],
        "domain":["full:www.baidu.com"],
        "outboundTag":"tunnel"
      },
      {  
        "type":"field",
        "inboundTag":["bridge"],
        "outboundTag":"out"
      }
    ]
  }
}
```

### 客户端

```text-plain
{
  "log": {
    "access": "",
    "error": "",
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "port": 10901,
      "ip": "0.0.0.0",
      "protocol": "socks",
      "version": 5,
      "settings": {
        "auth": "password",
        "accounts": [
        {
            "user": "123123",
            "pass": "123123"
        }
        ]
    }
    }
  ],
  "outbounds": [
    {
      "protocol": "vmess",
      "settings": {
        "vnext": [
          {
            "address": "127.0.0.1",
            "port": 16825,
            "users": [
              {
                "id": "364bb82c-e6d3-4769-b5ca-588cc82266c7",
                "alterId": 0
              }
            ]
          }
        ]
      },
    "streamSettings": {
      "network": "tcp"
    }
    }
  ]
}
```

## 级联-适用于多层内网的情况

**注：只适用于内网多网卡情况，不适用于内外网混合级联。最里层接一层反代，后面连S角色是通过T来正向连接的，所以需要是多网卡的情况，让T可以直连S。不能一次用一个以上的反向。**
**想要实现反向多层级联，需要打多组反代，挂多层代理。**


在原本S（外网vps角色）、I（内网被控端）、C（客户端）的角色上在S与C之间增加N个角色以便达到级联效果。

新增T（内网中转）角色，T角色承接S和I角色作之间的流量，而且T角色与T角色串联，可以有多个。在TCP/KCP方面可以根据实际情况修改

I角色，未改变

```text-plain
{  
  "reverse":{ 
    "bridges":[  
      {  
        "tag":"bridge", 
        "domain":"www.baidu.com" 
      }
    ]
  },
  "outbounds": [
    {  
      "tag":"tunnel", 
      "protocol":"vmess",
      "settings":{  
        "vnext":[  
          {  
            "address":"192.168.15.100",
            "port":153,
            "users":[  
              {  
                "id":"b831381d-6324-4d53-ad4f-8cda48b30811",
                "alterId":0
              }
            ]
          }
        ]
      },
    "streamSettings": {
      "network": "kcp",
      "kcpSettings": {
        "mtu": 1350,
        "tti": 20,
        "uplinkCapacity": 5,
        "downlinkCapacity": 100,
        "congestion": true,
        "readBufferSize": 1,
        "writeBufferSize": 1,
        "header": {
          "type": "none"
        }
      }
    }
    },
    {  
      "protocol":"freedom",
      "settings":{},
      "tag":"out"
    }
  ],
  "routing":{   
    "rules":[  
      {  
        "type":"field",
        "inboundTag":["bridge"],
        "domain":["full:www.baidu.com"],
        "outboundTag":"tunnel"
      },
      {  
        "type":"field",
        "inboundTag":["bridge"],
        "outboundTag":"out"
      }
    ]
  }
}
```

S角色，未改变。S角色必须与I角色相连

```text-plain
{  
  "reverse":{
    "portals":[  
      {  
        "tag":"portal",
        "domain":"www.baidu.com"
      }
    ]
  },
  "inbounds": [
    {
      "tag": "tunnel1",
      "port":16825,
      "protocol":"vmess",
      "settings":{  
        "clients":[  
          {  
            "id":"364bb82c-e6d3-4769-b5ca-588cc82266c7",
            "alterId": 0
          }
        ]
      },
    "streamSettings": {
      "network": "tcp"
    }
    },
    {
      "tag": "tunnel",
      "port":153,
      "protocol":"vmess",
      "settings":{  
        "clients":[  
          {  
            "id":"b831381d-6324-4d53-ad4f-8cda48b30811",
            "alterId":0
          }
        ]
      },
    "streamSettings": {
      "network": "kcp",
      "kcpSettings": {
        "mtu": 1350,
        "tti": 20,
        "uplinkCapacity": 5,
        "downlinkCapacity": 100,
        "congestion": true,
        "readBufferSize": 1,
        "writeBufferSize": 1,
        "header": {
          "type": "none"
        }
      }
    }
    }
  ],
  "routing":{  
    "rules":[  
      {  
        "type":"field",
        "inboundTag":[  
          "tunnel1"
        ],
        "outboundTag":"portal"
      },
      {  
        "type":"field",
        "inboundTag":[  
          "tunnel"
        ],
        "outboundTag":"portal"
      }
    ]
  }
}
```

T角色，连接上游S的流量，outbound与S角色的inbound对应

```text-plain
{
  "log": {
    "access": "",
    "error": "",
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "tag": "tunnel1",
      "port":16826,
      "protocol":"vmess",
      "settings":{  
        "clients":[  
          {  
            "id":"643023d6-a151-fa17-814e-a1609212dc96",
            "alterId": 0
          }
        ]
      },
    "streamSettings": {
      "network": "tcp"
    }
    }    
  ],
  "outbounds": [
    {
      "protocol": "vmess",
      "settings": {
        "vnext": [
          {
            "address": "192.168.14.100",
            "port": 16825,
            "users": [
              {
                "id": "364bb82c-e6d3-4769-b5ca-588cc82266c7",
                "alterId": 0
              }
            ]
          }
        ]
      },
    "streamSettings": {
      "network": "tcp"
    }
    }
  ]
}
```

继续增加T角色，outbounds与上游T角色inbound对应，后面可以无限增加T角色

```text-plain
{
  "log": {
    "access": "",
    "error": "",
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "tag": "tunnel1",
      "port":16827,
      "protocol":"vmess",
      "settings":{  
        "clients":[  
          {  
            "id":"a27a24ca-860b-769c-fa0b-6cc3944ce61a",
            "alterId": 0
          }
        ]
      },
    "streamSettings": {
      "network": "tcp"
    }
    }    
  ],
  "outbounds": [
    {
      "protocol": "vmess",
      "settings": {
        "vnext": [
          {
            "address": "127.0.0.1",
            "port": 16826,
            "users": [
              {
                "id": "643023d6-a151-fa17-814e-a1609212dc96",
                "alterId": 0
              }
            ]
          }
        ]
      },
    "streamSettings": {
      "network": "tcp"
    }
    }
  ]
}
```

C角色，与最后一个T角色转接

```text-plain
{
  "log": {
    "access": "",
    "error": "",
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "port": 10901,
      "ip": "0.0.0.0",
      "protocol": "socks",
      "version": 5,
      "settings": {
        "auth": "password",
        "accounts": [
        {
            "user": "123123",
            "pass": "123123"
        }
        ]
    }
    }
  ],
  "outbounds": [
    {
      "protocol": "vmess",
      "settings": {
        "vnext": [
          {
            "address": "127.0.0.1",
            "port": 16827,
            "users": [
              {
                "id": "a27a24ca-860b-769c-fa0b-6cc3944ce61a",
                "alterId": 0
              }
            ]
          }
        ]
      },
    "streamSettings": {
      "network": "tcp"
    }
    }
  ]
}
