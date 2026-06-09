# Avalon

avalon, 名称来源意为 阿瓦隆, 象征亚瑟王的安息之地，也是亚瑟王手中`遥远的理想乡`（大概）

我称他为IP托管软件，目的是通过xdp/iptables将某台机器的流量通过quic/tcp mux反向代理到另一台机器，与frp等内网穿透软件不同的是，该软件除了可以使用xdp进行内核级数据包转发与路由，同时还使用tun+linux offload来处理流量, 在默认情况下，他将代理主机的全部流量而不是部分端口.....这是与常规内网穿透软件的显著差异

软件分为两个部分，wall端以及avalon端，wall端一般作为公网机器，将来临的流量转发到avalon端（一般作为内网机器）

 - 在默认情况下，wall会转发机器的**全部流量**, 所以需要在jump_ports提前包含wall端机器的ssh端口以免失联，当然，iptables/xdp的设置并不会将其持久化，所以如果失联只需在云服务器的控制台或者物理机重启即可

 - 软件在1.8.0已经支持tcp+udp, ICMP与未知网络层数据包会被投送进wall端机器正常内核网络栈中

 - 切记wall端与avalon端需要保证使用相同分支（quic or tcp mux）

 - README以最新版release为准. 

使用的编程语言：Rust

**配置文件解释:现已全部转移到Config.md**

## 关于未来更新路线（TODO）

1. 提供类似SRV解析的方式使能通过不同的域名转发到不同的端口（预计会解析更深层次的信息，目前计划上不会默认启用）

2. 更多更多的性能优化

3. 提供简单可选的API以能够查看软件状态和统计信息（预计会解析更深层次的信息，目前计划上不会默认启用）

4. 合并black_ip_path与black_ips:list参数

## 可能需要的技术细节与bug  

### 关于xdp

目前有一个已知的bug, 对于xdp路线并不会处理网段过滤

对于xdp分支, jump_ports与ban_ports均在xdp层面被拦截，而黑名单ip功能在用户态，未来会将黑名单ip/网段以bloomfilter + lpm trie聚合的方式实现在xdp中

由于没有动态机制，所以对于xdp分支，jump_ports与ban_ports每个均只能承受最高50个端口

### 关于黑名单算法
qp trie与bloom对于个人使用几乎只有内存区别，由于bloom算法特性，bloom算法的内存使用几乎只有qp trie的1/10, 在百万条的数据库基准上，bloom算法约占用9mb内存，而qp trie则是90mb

默认使用bloom算法，假阳率为千分之一，最大可容纳1150000的黑名单ip数，该两项参数在之后版本会调整为可配置状态

### 关于mss_less

由于配置值为mss而非mtu, 假设主机出入口网卡mtu为1400, 则配置值应当改为1360, 默认情况下, mss_less配置值为1460

## 关于QUIC密钥生成

v1.6.0及以后版本可使用--tls-create flag交互式生成证书, 需要安装openssl软件包

`sudo apt install openssl`

启用软件时附加--tls-create即可进入交互式生成环节, 在生成完成后, 程序会在wall端程序目录下生成`ed25519_private.pem`与`ed25519_cert.pem`, `ed25519_cert.pem`为程序公钥, 而`ed25519_private.pem`为程序私钥，使用ed25519进行加密

密钥在wall端生成后, 将公钥复制一份到avalon端机器并填写配置即可

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
