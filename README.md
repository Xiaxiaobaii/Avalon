# Avalon

Avalon 是一个用于 **内网穿透** 工具，与frp不同的是，它采用tun+反向代理+连接池在传输层将数据进行tcp转发

部署连接之后自动转发全部端口，给家里云套上公网IP体验

它由两个程序组成：

- `wall`：部署在 **公网机器** 上，负责接收入站连接
- `avalon`：部署在 **内网机器** 上，负责把流量转发到本机服务

---

### 公网机器运行 `wall`

这台机器需要：

- 能被外部访问
- Linux 环境
- 安装iptables
- 一般需要 root 权限运行

### 内网机器运行 `avalon`

这台机器需要：

- 能主动访问公网机器的 `call_port`

---

## 示例配置

```json
// wall_config.json
{
  "ports":[22],
  "call_port":39999,
  "debug":false,
  "black_ip_path":"black.txt",
  "black_ips_list":["45.186.0.0/16"]
}
```

配置在第一次启动时会自动生成

### 字段解释

- `jump_ports`：该数组内的端口会被本机接管不会被穿透
- `ban_ports`: 该数组内的端口会被iptables直接丢弃
- `call_port`：监听端口，用于反向代理
- `debug`：基本没什么用，默认false就好
- `black_ip_path`：可配置的黑名单数据库，支持的数据库结构参照：https://github.com/borestad/blocklist-abuseipdb
- `black_ips_list`: 可配置的黑名单IP段

---

```json
// avalon_config.json
{
  "remote_addr":"YOUR_WALL_IP:WALL_CALL_PORT",
}
```

remote_addr就是你的Wall程序部署机器的ip:call_port

</details>
