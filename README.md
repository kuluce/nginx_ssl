# 使用说明

# cfssl安装

```bash
$ go install github.com/cloudflare/cfssl/cmd/...@latest
```

## 初始化

```bash
cfssl print-defaults config > ca-config.json
cfssl print-defaults csr > ca-csr.json
```

## 1 CA证书
### 1.1 修改ca-config.json
```json
{
  "signing": {
    "default": {
      "expiry": "87660h"
    },
    "profiles": {
      "peer": {
        "expiry": "87660h",
        "usages": ["signing", "key encipherment", "server auth", "client auth"]
      },
      "server": {
        "expiry": "87660h",
        "usages": ["signing", "key encipherment", "server auth"]
      },
      "client": {
        "expiry": "87660h",
        "usages": ["signing", "key encipherment", "client auth"]
      }
    }
  }
}
```

### 1.2 生成CA证书

```bash
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
```

## 2 服务器证书

### 2.1 生成服务端证书配置模板

模版文件： server-csr.json

```json
{
    "CN":"hongsheng",
    "hosts":[
        "127.0.0.1",
        "192.168.1.110", 
        "dev.hhs",
        "*.dev.hhs"
    ],
    "key":{
        "algo":"rsa",
        "size":4096
    },
    "names":[
        {
            "C":"CN",
            "L":"SH",
            "ST":"Shanghai",
            "O":"hhs",
            "OU":"dev"
        }
    ]
}
```

### 2.2 生成服务端证书 

```bash
cfssl gencert -ca=ca.pem \
-ca-key=ca-key.pem \
-config=ca-config.json \
-profile=server server-csr.json \
| cfssljson -bare server
```

## 3 客户端证书

### 3.1 生成客户端证书配置模板

模板文件: client1-csr.json

```json
{
    "CN": "client1",
    "key": {
      "algo": "rsa",
      "size": 4096
    },
    "names": [
      {
        "C":"CN",
        "L":"SH",
        "ST":"Shanghai", 
        "O": "Client Certificate 1",
        "OU": "Client Certificate 1"
      }
    ],
    "hosts": [
      "127.0.0.1"
    ]
  }
```

### 3.2 生成客户端证书
```bash
cfssl gencert -ca=ca.pem \
-ca-key=ca-key.pem \
-config=ca-config.json \
-profile=client client1-csr.json \
| cfssljson -bare client_1
```



# Nginx双向认证配置



server 段配置

```shell
# 双向认证站点例子
server {
    listen       443 ssl; 
    server_name localhost *.dev.hhs; 

    access_log  logs/28083/access.log  main;
    error_log   logs/28083/error.log;

    # 服务器证书和私钥
    ssl_certificate      cert/28083/server.crt;                    # 服务端证书
    ssl_certificate_key  cert/28083/server-key.pem;                # 服务端私钥

    # 客户端证书
    ssl_client_certificate  cert/28083/ca.pem;
    ssl_verify_client on;                                          # 开启客户端证书校验
    ssl_verify_depth 6;                                            # 校验深度

    # CA证书
    ssl_trusted_certificate cert/28083/ca.pem;                     # 将CA证书设为受信任的证书

    # 可选的SSL配置项  
    ssl_prefer_server_ciphers on;                                  # 优先采取服务器算法
    ssl_session_cache    shared:SSL:1m;                            # 配置共享会话缓存大小
    ssl_session_timeout  5m;                                       # session有效期5分钟
    ssl_protocols  TLSv1.2;                                        #启用指定的协议
    ssl_ciphers  ALL:!DH:!EXPORT:!RC4:+HIGH:+MEDIUM:-LOW:!aNULL:!eNULL; #加密算法 

    # 应用上下文
    location ~* ^/(hello|hello2|admin|) {        
        proxy_pass http://127.0.0.1:19999; 
        proxy_connect_timeout 300s;
        proxy_send_timeout 300s;
        proxy_read_timeout 300s;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto http;
    } 

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   html;
    }

    # 禁止访问根目录
    location / {
        return 403;
    }

}
```

