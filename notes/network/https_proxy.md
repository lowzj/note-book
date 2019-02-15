# HTTPS Proxy 部署

## 生成自签名证书

> 转自: https://www.barretlee.com/blog/2015/10/05/how-to-build-a-https-server/

```bash
#!/bin/bash

# step 1 为服务器端和客户端准备公钥、私钥
# 生成服务器端私钥
openssl genrsa -out server.key 1024
# 生成服务器端公钥
openssl rsa -in server.key -pubout -out server.pem


# step 2 生成 CA 证书
# 生成客户端私钥
openssl genrsa -out client.key 1024
# 生成客户端公钥
openssl rsa -in client.key -pubout -out client.pem

# 生成 CA 私钥
openssl genrsa -out ca.key 1024
# X.509 Certificate Signing Request (CSR) Management.
openssl req -new -key ca.key -out ca.csr
# X.509 Certificate Data Management.
openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt

# ➜  keys  openssl req -new -key ca.key -out ca.csr
# You are about to be asked to enter information that will be incorporated
# into your certificate request.
# What you are about to enter is what is called a Distinguished Name or a DN.
# There are quite a few fields but you can leave some blank
# For some fields there will be a default value,
# If you enter '.', the field will be left blank.
# -----
# Country Name (2 letter code) [AU]:CN
# State or Province Name (full name) [Some-State]:Zhejiang
# Locality Name (eg, city) []:Hangzhou
# Organization Name (eg, company) [Internet Widgits Pty Ltd]:My CA
# Organizational Unit Name (eg, section) []:
# Common Name (e.g. server FQDN or YOUR name) []:localhost
# Email Address []:


# step 3 生成服务器端证书和客户端证书
# 服务器端需要向 CA 机构申请签名证书，在申请签名证书之前依然是创建自己的 CSR 文件
openssl req -new -key server.key -out server.csr
# 向自己的 CA 机构申请证书，签名过程需要 CA 的证书和私钥参与，最终颁发一个带有 CA 签名的证书
openssl x509 -req -CA ca.crt -CAkey ca.key -CAcreateserial -in server.csr -out server.crt

# client 端
openssl req -new -key client.key -out client.csr
# client 端到 CA 签名
openssl x509 -req -CA ca.crt -CAkey ca.key -CAcreateserial -in client.csr -out client.crt
```

获取服务端签名, 即`server.crt`的内容:

```bash
openssl x509 -in <(openssl s_client -showcerts -servername dfdaemon.com -connect dfdaemon.com:65001 -prexit 2>/dev/null)
```

## 七层 HTTPS Proxy

### 使用nginx做代理

    ```ini
    # proxy config
    server {
        listen       443 ssl;
        server_name localhost;
    
        # 上面生成的自签名证书
        ssl_certificate /tmp/ssl/server.crt;
        ssl_certificate_key /tmp/ssl/server.key;
    
        ssl_session_timeout 5m;
    
        location / {
            proxy_set_header  X-Real-IP  $remote_addr;
            proxy_set_header  Host $host;
            proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
            # 后端服务
            proxy_pass http://localhost:8080;
        }
    }
    ```

### 使用httpx-static做代理

> 地址: https://github.com/ossrs/go-oryx/tree/develop/httpx-static

* 下载安装: 

    ```bash
    go get github.com/ossrs/go-oryx/httpx-static
    ```
* 使用

    ```bash
    # 例子1: 创建 https 服务
    $GOPATH/bin/httpx-static -ssc /tmp/ssl/server.crt -ssk /tmp/ssl/server.key -https 443
    
    # 例子2: https proxy
    $GOPATH/bin/httpx-static -ssc /tmp/ssl/server.crt -ssk /tmp/ssl/server.key -https 443 -proxy http://localhost:8080
    ```

### 四层 HTTPS Proxy

利用nginx stream可以代理到任意其他https网站, 详细可参考:

* http://nginx.org/en/docs/stream/ngx_stream_ssl_module.html
* https://docs.nginx.com/nginx/admin-guide/security-controls/terminating-ssl-tcp/

```ini
stream {
    server {
        listen                443 ssl;
        proxy_pass            https://$ssl_preread_server_name;

        ssl_certificate       /tmp/ssl/server.crt;
        ssl_certificate_key   /tmp/ssl/server.key;
        ssl_protocols         SSLv3 TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers           HIGH:!aNULL:!MD5;
        ssl_session_cache     shared:SSL:20m;
        ssl_session_timeout   4h;
        ssl_handshake_timeout 30s;
     }
}
```

### 访问 HTTPS Proxy

使用`curl`进行测试验证, 因为我们生成的是自签名的证书, 所以得带上证书.

* 七层

    ```bash
    curl --cacert /tmp/ssl/server.crt 'https://localhost'
    ```

* 四层

```bash
export https_proxy=https://localhost:443
curl --proxy-cacert /tmp/ssl/server.crt 'https://

```

