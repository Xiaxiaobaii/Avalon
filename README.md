# Avalon

avalon, 名称来源意为 阿瓦隆, 象征亚瑟王的安息之地，也是亚瑟王手中`遥远的理想乡`（大概）

目前是一个底部为QUIC协议的端口转发程序，但不同于frp的是

avalon通过iptables和tun在网络层工作，所以使用该程序需要拥有root权限和iptables软件包，这也意味着该程序仅在linux所被支持，同时意味着avalon并不会监听任何端口（除了wall端的call_port需要监听一个用于于avalon进行反向代理的端口）

好消息是只有wall端(外网端, 拥有公网ip暴露在外网的服务器)需要这个要求

坏消息是由于该项目为闭源项目，而且作者是懒狗不想打包windows二进制，考虑到软件本身就没人用（几乎只有作者自己）

[TODO] 更好的底层tun库与Hardware Offload正在测试中.....

[warn] 在默认情况下，wall会转发机器的**全部端口**, 所以需要在jump_ports提前jump掉wall端机器的ssh端口以免失联，当然，iptables的设置并不会将其持久化，所以如果失联只需在云服务器的控制台或者物理机重启即可

[warn] 目前软件会处理所有ICMP及TCP包，但是不会处理UDP包（等待支持中）

## 配置文件解释

v1.6.0版本的配置文件名称统一为config.json, 此前版本wall端为wall_config.json, avalon端为avalon_config.json

### wall side
- jump_ports: 可选, 类型为数组(值为u16), 数组内的端口都不会转发到内网服务器而是在wall端处理
- ban_ports: 可选，类型为数组(值为u16)，数组内的端口会被直接丢弃
- call_port: 可选，类型为u16, 默认为39999, wall端与avalon端沟通的端口
- debug: 可选, 类型为布尔值, 默认为false, 用处不大
- black_ip_path: 可选, 类型为文件路径(String), 黑名单ip数据库，支持的数据库结构参照：https://github.com/borestad/blocklist-abuseipdb，黑名单ip的请求访问都会被丢弃
- black_ips:list: 可选，类型为数组(值为String), 数组内的ip段都会被丢弃
- **mss_less**: 可选，类型为u32，启用后将Maximum Segment Size强制设置为配置值，用于实现MSS 钳制以解决mtu问题
- tls_pem_path: 必选, 类型为文件路径(String), QUIC所需的X.509证书公钥
- tls_key_path: 必选, 类型为文件路径(String), QUIC所需的X.509证书私钥

如果遇到奇怪的数据丢失问题，连接成功却无法正常通信的问题，尝试将mss_less依次尝试设置为: 1460 -> 1400 -> 1360

-------

### avalon side
- remote_addr: 必选, 类型为String, 配置为wall端服务器的地址+端口, 如"159.43.243.12:3999"
- tls_pem_path: 必选, 类型为文件路径(String), QUIC所需的X.509证书公钥, 与服务器公钥相同

```
示例文件
wall side:

{"tls_pem_path": "ed25519_cert.pem", "tls_key_path": "ed25519_private.pem"}

avalon side:
{"tls_pem_path": "ed25519_cert.pem", "remote_addr": "your remote address"}
```

## 关于QUIC密钥的简单生成

v1.6.0版本可使用--tls-create flag交互式生成证书,  需要安装openssl软件包

`sudo apt install openssl`

启用软件时附加--tls-create即可进入交互式生成环节

密钥在wall端生成一次后，将公钥copy一份给avalon端即可

### 手动生成

指令只需要修改subjectAltName后面的参数即可

注：签名的域名/IP就是客户端连接服务端时使用的域名/IP, 如配置文件中, remote_addr设置为"159.43.243.12:3999", 则使用仅IP的签名并设置`subjectAltName=IP:159.43.243.12`下面有完整的签名示例

仅域名
```
openssl genpkey -algorithm ed25519 -out ed25519_private.pem
openssl x509 -req -days 365 \
  -in <(openssl req -new -key ed25519_private.pem -subj "/CN=avalon") \
  -signkey ed25519_private.pem \
  -out ed25519_cert.pem \
  -extfile <(cat << 'EOF'
subjectAltName = DNS:lyxnxia.space
basicConstraints = critical,CA:FALSE
keyUsage = digitalSignature
extendedKeyUsage = serverAuth
EOF
)
```

该指令会在当前目录输出`ed25519_private.pem`和`ed25519_cert.pem`文件，并签名域名`lyxnxia.space`

仅IP
```
openssl genpkey -algorithm ed25519 -out ed25519_private.pem
openssl x509 -req -days 365 \
  -in <(openssl req -new -key ed25519_private.pem -subj "/CN=avalon") \
  -signkey ed25519_private.pem \
  -out ed25519_cert.pem \
  -extfile <(cat << 'EOF'
subjectAltName = IP:192.168.1.100
basicConstraints = critical,CA:FALSE
keyUsage = digitalSignature
extendedKeyUsage = serverAuth
EOF
)
```

该指令会在当前目录输出`ed25519_private.pem`和`ed25519_cert.pem`文件，并签名IP`192.168.1.100`

混合生成

```
openssl genpkey -algorithm ed25519 -out ed25519_private.pem
openssl x509 -req -days 365 \
  -in <(openssl req -new -key ed25519_private.pem -subj "/CN=avalon") \
  -signkey ed25519_private.pem \
  -out ed25519_cert.pem \
  -extfile <(cat << 'EOF'
subjectAltName = DNS:example.com,DNS:*.myserver.com,IP:192.168.1.100,IP:10.0.0.5
basicConstraints = critical,CA:FALSE
keyUsage = digitalSignature
extendedKeyUsage = serverAuth
EOF
)
```

该指令会在当前目录输出`ed25519_private.pem`和`ed25519_cert.pem`文件，并签名域名`example.com`和example.com的泛域名, 还有IP`192.168.1.100`与`10.0.0.5`
