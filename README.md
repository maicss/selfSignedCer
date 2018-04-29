# 创建自签名证书

## CA


这部分是干的网上收费颁发证书的活。只不过他们受到了各种设备的默认信任，基本上每个操作系统都安装了，所以浏览器等都认为这种网站是可信的。

自己当办法机构之后，就需要在想浏览自己网站的电脑上安装这个CA证书文件即可。

这里的证书默认以`root`命名。

### CA私钥
```bash
openssl genrsa -des3 -out root.key 4096
```

### CA申请文件

除了`extra`部分，都尽量填
```bash
openssl req -new -key root.key -out root.csr
```

### CA证书

```bash
openssl x509 -req -days 3650 -sha256 -extensions v3_ca -signkey root.key -in root.csr -out root.crt
```

这个证书是3650天的期限，也即约10年

## 服务器证书

这个部分是用户向网站缴费之后，提交一些必要信息，网站给生成的文件，放在服务器使用的。

这个证书这里以`server`命名


### 服务器密钥

1, 生成密钥

```bash
openssl genrsa -des3 -out server.key 2048
```

2，删除密钥密码（可选）

```bash
openssl rsa -in server.key -out server.key
```

### 创建服务器证书申请文件

这里也尽量填满。
这一步和下一步是在商业证书网站上填表格的部分。

```bash
openssl req -new -key server.key -out server.csr
```

### 创建服务器证书使用规则

```
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]

# 如果证书只是本地使用
DNS.1 = localhost
DNS.2 = 127.0.0.1
DNS.3 = 你自己的IP，这个一般也没啥用

# 如果是局域网内使用
DNS.1 = example.com
DNS.2 = *.example.com
DNS.3 = 局域网内server IP
```

注意如果是`a.b.example.com`, 规则就需要写成`*.b.example.com`。

### 创建服务器证书

```bash
openssl x509 -req -days 3650 -sha256 -CA root.crt -CAkey root.key -CAcreateserial -in server.csr -out server.crt -extfile v3.ext
```

## 客户端证书（一般不做）

> 只在需要双向认证的时候使用。这个部分于2017年之前使用的，不确保现在有效。

这里都以`client`命名。

### 创建私钥

```bash
openssl genrsa -des3 -out client.key 2048
```

删除私钥密码（选做）

```bash
openssl rsa -in client.key -out client.key
```

### 创建客户端申请文件

```bash
openssl req -new -key client.key -out client.csr
```

### 创建客户端证书

```
openssl x509 -req -days 365 -sha256 -extensions v3_req -CA root.crt -CAkey root.key -CAcreateserial -in client.csr -out client.crt
```

## 证书压缩

其实这部分也不叫压缩，只能算是合并。客户端和服务端都可以做。证书和密钥文件合并到一个文件里面，使用起来能少写点文件名称和路径。选做。

这里提供的是压缩到`PKCS`格式的。如果用到其他格式和特殊服务器格式再自己搜。

```bash
openssl pkcs12 -export -in server.crt -inkey server.key -out server.pfx

# or
openssl pkcs12 -export -in client.crt -inkey client.key -out client.pfx

```

## 其他

证书里的`crt`、`key`、`pem`等都是一样的，没啥区别。