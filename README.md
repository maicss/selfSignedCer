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

> 这个部分可以使用配置的方式写好，而不用每次都一个个的输入，而且REPL还不支持删除，错了一个就要重新开始。配置参见<a href="#csr配置">csr配置</a>

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
openssl genrsa -des3 -out server.key 4096
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

命名为`v3.ext`，就是接来下命令最后的参数

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
openssl genrsa -des3 -out client.key 4096
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

## csr配置

假设文件名称为`csr.conf`。
```
# OpenSSL configuration to generate a new key with signing requst for a x509v3 
# multidomain certificate
#
# openssl req -config bla.cnf -new | tee csr.pem
# or
# openssl req -config bla.cnf -new -out csr.pem
[ req ]
default_bits       = 4096
default_md         = sha512
default_keyfile    = key.pem
prompt             = no
encrypt_key        = no

# base request
distinguished_name = req_distinguished_name

# extensions
req_extensions     = v3_req

# distinguished_name
[ req_distinguished_name ]
countryName            = "DE"                     # C=
stateOrProvinceName    = "Hessen"                 # ST=
localityName           = "Keller"                 # L=
postalCode             = "424242"                 # L/postalcode=
streetAddress          = "Crater 1621"            # L/street=
organizationName       = "apfelboymschule"        # O=
organizationalUnitName = "IT Department"          # OU=
commonName             = "example.com"            # CN=
emailAddress           = "webmaster@example.com"  # CN/emailAddress=

# req_extensions
# 这个就是上面的v3.ext
[ v3_req ]
# The subject alternative name extension allows various literal values to be 
# included in the configuration file
# http://www.openssl.org/docs/apps/x509v3_config.html
subjectAltName  = @alt_names

[alt_names]

DNS.1 = example.com
DNS.2 = *.example.com

# vim:ft=config
```

请求文件好了之后，创建csr文件：

```bash
openssl req -config csr.conf -new  -key some.key -out some.csr
```

配置根据自己的需要进行修改。就不用交互式的一个个填，而且填错之后重来一遍，烦死了。

而且这个可以保存，一般是ca一个，server一个。当然全部用一个也没什么关系。`default_keyfile`别弄错了，一个是`root.key`, 一个是`server.key`.

## 其他

证书里的`crt`、`key`、`pem`等都是一样的，没啥区别。
