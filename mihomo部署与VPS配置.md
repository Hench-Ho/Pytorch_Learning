# 服务器 mihomo 部署 + VPS 代理配置说明

> **梯子用的是 xdd 的 VPS**(洛杉矶节点),下面的节点凭证均为该 VPS。换节点/加独立账号找 xdd。
>
> 在一台 Linux 服务器上部署 mihomo 客户端,连到该 VPS 走代理出国。
> 下文用 `~/clash` 作为工作目录,你可换成任意目录;命令里不含机器/用户专属信息,可直接照搬。

## 一、整体结构

```
本机应用 (curl/pip/...)
  → http_proxy=127.0.0.1:7891  (环境变量)
  → mihomo (本机, 混合代理端口 7891)
  → VLESS+Reality 加密隧道
  → VPS 38.60.91.124:23892 (洛杉矶)
  → 出国
```

- **协议**:VLESS + XTLS-Vision + Reality(无需自己的域名/证书,伪装成 `www.oracle.com`)
- **客户端**:mihomo(Clash.Meta 内核),裸二进制运行

## 二、部署步骤

### 1. 准备目录和二进制

```bash
mkdir -p ~/clash && cd ~/clash
# 下载对应架构的 mihomo(x86_64 一般是 linux-amd64):
# 到 https://github.com/MetaCubeX/mihomo/releases 下载后解压重命名为 mihomo
chmod +x mihomo
./mihomo -v   # 确认能跑
```

> 若下载不了,可从已有机器 `scp` 一份 `mihomo` 二进制过去(注意架构一致)。

### 2. 放入分流规则库(可选,首次运行会自动下载)

`geoip.metadb`、`GeoSite.dat` 放在工作目录即可。没有的话 mihomo 首次启动会自动拉取(需要能联网,或先临时用别的代理)。

### 3. 写配置文件 `~/clash/config.yaml`

见下面第三节,直接复制。建议设权限 `chmod 600 config.yaml`(含 uuid 等凭证)。

### 4. 启动

```bash
cd ~/clash
nohup ./mihomo -f config.yaml -d . > mihomo.log 2>&1 &
```
`-d .` 指定工作目录(读 geoip/geosite、写 cache.db)。

### 5. 让终端走代理

加到 `~/.bashrc`(或 `~/.bash_profile`),然后 `source ~/.bashrc`:
```bash
export http_proxy=http://127.0.0.1:7891
export https_proxy=http://127.0.0.1:7891
export HTTP_PROXY=http://127.0.0.1:7891
export HTTPS_PROXY=http://127.0.0.1:7891
export no_proxy=localhost,127.0.0.1,<内网网段如172.31.0.0/16>
```

### 6. 验证

```bash
curl -s -x http://127.0.0.1:7891 --max-time 8 https://api.ipify.org
# 返回 38.60.91.124 即为走 VPS 出国成功
```

## 三、配置文件 config.yaml(直接复制)

```yaml
mixed-port: 7891              # HTTP + SOCKS5 混合代理入口
allow-lan: true              # 如需局域网共享设 true,否则改 false 更安全
bind-address: '*'
mode: rule
log-level: warning           # 想排错时临时改 info
external-controller: '127.0.0.1:9091'

dns:
  enable: true
  ipv6: false
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
  default-nameserver: [223.5.5.5, 119.29.29.29]
  nameserver: [223.5.5.5, 119.29.29.29]
  fallback: [1.1.1.1, 8.8.8.8]
  fallback-filter: { geoip: true, geoip-code: CN }

proxies:
  - name: vps-la
    type: vless
    server: 38.60.91.124        # VPS 公网 IP(洛杉矶)
    port: 23892
    uuid: d773c514-3a28-477f-b044-871962b9121a
    flow: xtls-rprx-vision      # XTLS Vision 流控
    tls: true
    servername: www.oracle.com  # Reality 伪装的 SNI
    reality-opts:
      public-key: 0uEYwL46g56Svzy2u2KbzljbUoN0fg2HnNkFvIYoTGs
      short-id: "90"
    network: tcp
    udp: true
    client-fingerprint: chrome

proxy-groups:
  - name: 🚀 节点选择
    type: select
    proxies:
      - vps-la
      - DIRECT

rules:
  - IP-CIDR,127.0.0.0/8,DIRECT,no-resolve
  - IP-CIDR,192.168.0.0/16,DIRECT,no-resolve
  - IP-CIDR,10.0.0.0/8,DIRECT,no-resolve
  - IP-CIDR,172.16.0.0/12,DIRECT,no-resolve
  - GEOSITE,cn,DIRECT
  - GEOIP,CN,DIRECT
  - GEOSITE,geolocation-!cn,🚀 节点选择
  - MATCH,🚀 节点选择
```

**分流逻辑**:本地/内网直连;国内域名和 IP(GEOSITE,cn / GEOIP,CN)直连;国外走 VPS;兜底走 VPS。

## 四、VPS 节点参数说明

这些参数客户端必须与 VPS 服务端严格一致,任何一项不对都连不上:

| 字段 | 含义 |
|------|------|
| `server` / `port` | VPS 公网 IP 和端口 |
| `uuid` | 用户凭证,服务端 clients 里注册的同一个 |
| `public-key` | VPS 私钥对应的公钥(服务端持私钥) |
| `short-id` | 服务端 Reality shortIds 之一 |
| `servername` | Reality 偷用的真实网站 SNI(`www.oracle.com`) |
| `flow: xtls-rprx-vision` | Vision 流控,服务端 inbound 必须同样开启 |

> 服务端(Xray/sing-box)跑在 VPS `38.60.91.124` 上,持有与 public-key 配对的私钥。多人共用同一节点没问题,但**建议每人一个独立 uuid**,方便管理和封禁——需要的话找 VPS 管理员在服务端加 client。

## 五、常用运维命令

```bash
# 查看进程
ps aux | grep '[m]ihomo'

# 停止(按配置文件精确匹配,避免误杀同机其他 mihomo)
pkill -f 'mihomo -f config.yaml'

# 启动
cd ~/clash && nohup ./mihomo -f config.yaml -d . > mihomo.log 2>&1 &

# 改完配置热重载(通过 external-controller)
curl -X PUT http://127.0.0.1:9091/configs -d "{\"path\":\"$HOME/clash/config.yaml\"}"

# 日志清理(mihomo.log 会持续增长)
: > ~/clash/mihomo.log
```

## 六、可选:开机自启(systemd user service)

服务器重启后 mihomo 不会自动拉起。想自启可建 `~/.config/systemd/user/mihomo.service`:

```ini
[Unit]
Description=mihomo
After=network-online.target

[Service]
WorkingDirectory=%h/clash
ExecStart=%h/clash/mihomo -f config.yaml -d .
Restart=on-failure

[Install]
WantedBy=default.target
```

然后:
```bash
systemctl --user daemon-reload
systemctl --user enable --now mihomo
loginctl enable-linger $USER   # 允许用户服务在未登录时也运行
```

## 七、注意事项

1. **端口冲突**:若 7891/9091 被占,改 `mixed-port` / `external-controller` 端口,同步改环境变量。
2. **`allow-lan`**:仅自己用建议设 `false`;要给同网段其他机器共享才设 `true`。
3. **日志**:排错时把 `log-level` 临时改回 `info`,平时用 `warning` 防止日志暴涨。
4. **换节点/换密钥**:只改 `config.yaml` 里 server/port/uuid/public-key/short-id 五项,与服务端一致后热重载即可。
