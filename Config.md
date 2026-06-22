# 配置文件解释

v1.6.0版本的配置文件名称统一为config.json, 此前版本wall端为wall_config.json, avalon端为avalon_config.json

## wall side
- jump_ports: 可选, 类型为数组(值为u16), 数组内的端口都不会转发到内网服务器而是在wall端处理
- ban_ports: 可选，类型为数组(值为u16)，数组内的端口会被直接丢弃
- call_port: 可选，类型为u16, 默认为39999, wall端与avalon端沟通的端口
- debug: 可选, 类型为布尔值, 默认为false
- black_ip_path: 可选, 类型为文件路径(String), 黑名单ip数据库，支持的数据库结构参照：https://github.com/borestad/blocklist-abuseipdb，按行读取ip和ip段，会忽略#开头的段落，ip采用bloom/qptrie算法过滤，ip段使用Lpmtrie
- **mss_less**: 可选，类型为u32，启用后将Maximum Segment Size强制设置为配置值，用于实现MSS钳制以解决pmtu问题
- tls_pem_path: 可选, 类型为文件路径(String), QUIC所需的X.509证书公钥，如缺失则自动fallback至tcp复用分支
- tls_key_path: 可选, 类型为文件路径(String), QUIC所需的X.509证书私钥，如缺失则自动fallback至tcp复用分支
- enable_trie: 可选, 类型为布尔值，是否使用trie算法来进行黑名单IP控制，默认算法为bloom
- enable_xdp: 可选，类型为String(需要路由的网卡名称)，是否使用xdp来路由流量，默认使用iptables进行路由
- use_skb: 可选，类型为布尔值，如果网卡不支持native xdp,则会提示使用Skb mode, 此时启用该选项即可
- disable_offload, 可选，类型为布尔值，启用后会禁用Linux Offload, 一般来说不建议使用
- old_handler: 可选，类型为布尔值，启用后强制使用tcp多路复用线路而非quic

-------

## avalon side
- **remote_addr**: **必选**, 类型为String, 配置为wall端服务器的地址+端口, 如"159.43.243.12:3999"
- tls_pem_path: 可选, 类型为文件路径(String), QUIC所需的X.509证书公钥, 与服务器公钥相同，如缺失则自动fallback至tcp复用分支
- forward_addr: 可选，类型为IP地址（String），默认为127.0.0.1,在wall端流量到来时请求的对应端口ip
- old_handler: 可选，类型为布尔值，启用后强制使用tcp多路复用线路而非quic
- debug: 可选, 类型为布尔值, 默认为false, 用处不大

```
示例文件
wall side:
{"tls_pem_path": "ed25519_cert.pem", "tls_key_path": "ed25519_private.pem"}

avalon side:
{"tls_pem_path": "ed25519_cert.pem", "remote_addr": "your remote address"}
```
