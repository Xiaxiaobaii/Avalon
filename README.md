# Avalon

avalon, 名称来源意为 阿瓦隆, 象征亚瑟王的安息之地，也是亚瑟王手中`遥远的理想乡`（大概）

我称他为IP托管软件，目的是通过xdp/iptables将某台机器的流量通过quic/tcp mux反向代理到另一台机器，与frp等内网穿透软件不同的是，该软件除了可以使用xdp进行内核级数据包转发与路由，同时还使用tun+linux offload来处理流量

软件分为两个部分，wall端以及avalon端，wall端一般作为公网机器，将来临的流量转发到avalon端（一般作为内网机器）

[warn] 在默认情况下，wall会转发机器的**全部流量**, 所以需要在jump_ports提前包含wall端机器的ssh端口以免失联，当然，iptables/xdp的设置并不会将其持久化，所以如果失联只需在云服务器的控制台或者物理机重启即可

[warn] 目前软件仅转发TCP包，对于ICMP/UDP包则会直接由wall端机器接收，对于UDP包会在未来进行支持

[warn] 切记wall端与avalon端需要保证使用相同分支（quic or tcp mux）

使用的编程语言：Rust

## 配置文件解释

v1.6.0版本的配置文件名称统一为config.json, 此前版本wall端为wall_config.json, avalon端为avalon_config.json

### wall side
- jump_ports: 可选, 类型为数组(值为u16), 数组内的端口都不会转发到内网服务器而是在wall端处理
- ban_ports: 可选，类型为数组(值为u16)，数组内的端口会被直接丢弃
- call_port: 可选，类型为u16, 默认为39999, wall端与avalon端沟通的端口
- debug: 可选, 类型为布尔值, 默认为false, 用处不大
- black_ip_path: 可选, 类型为文件路径(String), 黑名单ip数据库，支持的数据库结构参照：https://github.com/borestad/blocklist-abuseipdb，黑名单ip的请求访问都会被丢弃
- black_ips:list: 可选，类型为数组(值为String), 数组内的ip段都会被丢弃
- **mss_less**: 可选，类型为u32，启用后将Maximum Segment Size强制设置为配置值，用于实现MSS钳制以解决pmtu问题
- tls_pem_path: 可选, 类型为文件路径(String), QUIC所需的X.509证书公钥，如缺失则自动fallback至tcp复用分支
- tls_key_path: 可选, 类型为文件路径(String), QUIC所需的X.509证书私钥，如缺失则自动fallback至tcp复用分支
- enable_trie: 可选, 类型为布尔值，是否使用trie算法来进行黑名单IP控制，默认算法为bloom
- enable_xdp: 可选，类型为String(需要路由的网卡名称)，是否使用xdp来路由流量，默认使用iptables进行路由
- use_skb: 可选，类型为布尔值，如果网卡不支持native xdp,则会提示使用Skb mode, 此时启用该选项即可
- disable_offload, 可选，类型为布尔值，启用后会禁用Linux Offload, 一般来说不建议使用
- old_handler: 可选，类型为布尔值，启用后强制使用tcp多路复用线路而非quic

-------

### avalon side
- **remote_addr**: **必选**, 类型为String, 配置为wall端服务器的地址+端口, 如"159.43.243.12:3999"
- tls_pem_path: 可选, 类型为文件路径(String), QUIC所需的X.509证书公钥, 与服务器公钥相同，如缺失则自动fallback至tcp复用分支
- old_handler: 可选，类型为布尔值，启用后强制使用tcp多路复用线路而非quic

```
示例文件
wall side:

{"tls_pem_path": "ed25519_cert.pem", "tls_key_path": "ed25519_private.pem"}

avalon side:
{"tls_pem_path": "ed25519_cert.pem", "remote_addr": "your remote address"}
```

## 关于黑名单算法
qp trie与bloom对于个人使用几乎只有内存区别，由于bloom算法特性，bloom算法的内存使用几乎只有qp trie的1/10, 在百万条的数据库基准上，bloom算法约占用9mb内存，而qp trie则是90mb

默认使用bloom算法，假阳率为千分之一，最大可容纳1150000的黑名单ip数，该两项参数在之后版本会调整为可配置状态

qp trie的好处

## 关于mss_less

由于配置值为mss而非mtu, 假设主机出入口网卡mtu为1400, 则配置值应当改为1360, 默认情况下, mss_less配置值为1460

## 关于未来更新路线（TODO）

1. 提供类似SRV解析的方式使能通过不同的域名转发到不同的端口（预计会解析更深层次的信息，目前计划上不会默认启用）

2. UDP相关支持

3. 更多更多的性能优化

4. 使用qp trie以支持网段过滤，提供更深层次与更高性能的黑名单控制

5. 提供简单可选的API以能够查看软件状态和统计信息（预计会解析更深层次的信息，目前计划上不会默认启用）

## 关于QUIC密钥的简单生成

v1.6.0及以后版本可使用--tls-create flag交互式生成证书,  需要安装openssl软件包

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
