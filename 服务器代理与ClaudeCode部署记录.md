# 服务器代理 + Claude Code 部署全流程梳理

## 背景问题

服务器本身在国内网络环境下，无法直接访问 GitHub、npm 官方源、Anthropic 等海外服务。而 Claude Code 的安装、下载、更新、认证全部依赖这些海外资源，所以必须先在服务器上搭好一条"出国通道"，再在这条通道之上装 Claude Code。

整个过程可以分成两大阶段：**阶段一：搭建代理**，**阶段二：在代理之上装 Claude Code**。每一步遇到的报错，本质上都是"服务器缺一部分联网能力"，解决方式就是把这部分能力补上。

---

## 阶段一：mihomo 代理部署

### 0. 下载 mihomo 二进制

在配置代理之前，第一步是先把 mihomo（Clash.Meta 内核）这个可执行程序本身弄到服务器上。这一步同样面临"服务器还没联网能力，但下载文件需要联网"的问题，所以有两条路径：

**路径一：服务器本身能访问 GitHub 的话，直接下载**

```bash
mkdir -p ~/clash && cd ~/clash

# 先确认服务器 CPU 架构，决定下载哪个版本
uname -m
# x86_64 → 对应 mihomo-linux-amd64
# aarch64 → 对应 mihomo-linux-arm64

# 去 release 页面下载对应架构的压缩包
curl -L -o mihomo.gz https://github.com/MetaCubeX/mihomo/releases/latest/download/mihomo-linux-amd64.gz

gunzip mihomo.gz
mv mihomo-linux-amd64 mihomo   # 如果解压后文件名不同，按实际改名
chmod +x mihomo
./mihomo -v   # 确认能跑，且版本号正常打印
```

**路径二：服务器下载不动，从别的能联网的机器传过去**

在任意一台能访问 GitHub 的机器（比如你自己的电脑）上下载对应架构的 mihomo 二进制，然后用 scp 传到服务器：

```bash
# 本地机器上下载好之后
scp mihomo houhengchang@101.126.5.67:~/clash/mihomo

# 回到服务器上
cd ~/clash
chmod +x mihomo
./mihomo -v
```

**解决的问题**：拿到 mihomo 这个核心可执行程序本身。这一步是后续所有代理搭建工作的前提——没有这个二进制文件，配置文件写得再对也无法启动代理。同时也印证了整个部署过程中反复出现的同一类矛盾：服务器需要先具备联网能力才能搭代理，但搭代理所需的文件又得靠联网获取，只能靠"从其他已联网的地方转运文件"来打破这个循环。

### 1. 部署 mihomo 客户端

在服务器 `~/clash` 目录下放置 mihomo 二进制、写好 `config.yaml`（里面配置了 xdd 提供的洛杉矶 VPS 的 VLESS+Reality 节点信息），启动：

```bash
nohup ./mihomo -f config.yaml -d . > mihomo.log 2>&1 &
```

**解决的问题**：让服务器本机的网络流量，可以通过加密隧道转发到国外 VPS（38.60.91.124）出站，从而具备访问海外服务的能力。

### 2. 遇到问题：GeoSite.dat 损坏，下载超时

启动日志报错：

```
failed to decode geosite file: GeoSite.dat
can't download GeoSite.dat: context deadline exceeded
```

**原因**：mihomo 依赖的域名分流规则库文件（GeoSite.dat）本地已损坏，程序试图自动从 GitHub 重新下载，但此时代理还没搭起来，服务器本身没有出国能力去下载——典型的"先有鸡还是先有蛋"问题。

**解决办法**：换一个当时能连通的下载源（或从其他能联网的机器下载后 scp 传过去），手动把正确的 `GeoSite.dat` / `geoip.metadb` 放到 `~/clash/` 目录下，跳过 mihomo 自动下载这一步。
```
curl -L -o geosite.dat   https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/geosite.dat
curl -L -o geoip.metadb  https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/geoip.metadb
```
**解决的问题**：补齐了 mihomo 启动所需的规则文件，让 mihomo 能正常加载配置、完成初始化。

### 3. 验证代理生效

```bash
curl -s -x http://127.0.0.1:7891 --max-time 8 https://api.ipify.org
```

返回 `38.60.91.124`（VPS 出口 IP），说明代理链路完全打通：本机应用 → mihomo(7891 端口) → VLESS 隧道 → VPS → 国外。

**解决的问题**：确认"出国通道"本身是通的，后续所有安装动作都可以挂在这条通道上。

### 4. 配置开机自启（可选）

前面手动用 `nohup ... &` 启动的 mihomo，一旦服务器重启或者会话断开，进程就没了，需要人工重新启动。为了避免每次都要手动拉起，用 systemd 的用户级服务来管理：

```bash
mkdir -p ~/.config/systemd/user
cat > ~/.config/systemd/user/mihomo.service << 'EOF'
[Unit]
Description=mihomo
After=network-online.target

[Service]
WorkingDirectory=%h/clash
ExecStart=%h/clash/mihomo -f config.yaml -d .
Restart=on-failure

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user enable --now mihomo

# 允许用户级服务在没有登录会话时也能运行（否则 SSH 断开后服务可能被杀掉）
loginctl enable-linger $USER
```

配置好之后可以用下面命令确认状态、或者手动重启：

```bash
systemctl --user status mihomo
systemctl --user restart mihomo
```

**解决的问题**：
- 服务器重启后 mihomo 能自动拉起，不用每次手动 `nohup` 启动；
- `Restart=on-failure` 让 mihomo 意外崩溃退出时能自动重启，减少代理断连的情况；
- `loginctl enable-linger` 解决了"SSH 会话一断开，用户级服务就被systemd清理掉"的问题，保证代理在你退出终端后依然常驻运行。

---

## 阶段二：安装 Claude Code

### 5. 遇到问题：install.sh 返回 403

```bash
curl -fsSL -x http://127.0.0.1:7891 https://claude.ai/install.sh | sh
# curl: (22) The requested URL returned error: 403
```

**原因**：这个 403 不是网络不通（代理链路本身没问题），而是目标网站/CDN 对这个 VPS 出口 IP 做了访问限制——数据中心/VPS 类 IP 比住宅 IP 更容易被风控系统标记。

**解决办法**：绕开这条路径，换成 npm 安装方式（走的是 npm registry，被拦截概率较低）。

**解决的问题**：绕过了单一下载源被拦截的问题，改用另一条可用的安装路径。

### 6. 遇到问题：Node 版本太旧 + npm 权限不足

```
npm WARN EBADENGINE required: { node: '>=22.0.0' }, current: { node: 'v12.22.9' }
npm ERR! EACCES: permission denied, mkdir '/usr/local/lib/node_modules'
```

**原因**：
- 服务器系统自带的 Node.js 版本（v12）远低于 Claude Code 要求的 v22+，装了也无法运行；
- npm 默认往系统目录 `/usr/local/lib/node_modules` 装包，普通用户没有写权限，只能用 sudo 或换目录。

**解决办法**：用 **nvm**（Node Version Manager）在用户自己的家目录下装一个独立的 Node 22，不碰系统 Node、不需要 sudo：

```bash
curl -o- -x http://127.0.0.1:7891 https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
nvm install 22
nvm use 22
nvm alias default 22
```

**解决的问题**：
- 一次性解决了版本不达标的问题（用户级安装最新 Node，不影响系统环境）；
- 顺带解决了权限问题（nvm 管理的 npm 全局目录默认就在用户家目录下，不再需要 root 权限）。

### 7. 安装 Claude Code

```bash
npm install -g @anthropic-ai/claude-code
claude --version
```

**解决的问题**：在满足版本要求、且有出国网络能力的前提下，成功把 Claude Code 装到服务器上。

### 8. 认证（下一步要做的）

两种方式二选一：
- **浏览器登录**：运行 `claude`，复制终端打印的登录链接，在本地电脑浏览器里完成登录，把返回的授权码粘贴回终端；
- **API Key**：在 console.anthropic.com 生成 Key，`export ANTHROPIC_API_KEY="sk-ant-..."` 写入 `~/.bashrc`，跳过浏览器登录。

---

## 全程解决的核心问题一览

| 阶段 | 具体报错/障碍 | 本质原因 | 解决方式 |
|---|---|---|---|
| 代理搭建 | 服务器没有 mihomo 可执行文件 | 装代理前先要有代理程序本身，但下载它同样需要联网 | 服务器能连时直接下载对应架构版本；连不上就从其他联网机器下载后 scp 传过去 |
| 代理搭建 | GeoSite.dat 下载超时 | 规则文件损坏 + 当时无出国能力，陷入循环依赖 | 手动放置正确的规则文件，跳过自动下载 |
| 代理验证 | — | 需要确认链路可用 | curl 验证出口 IP |
| 代理常驻 | 重启/断开 SSH 后 mihomo 进程消失 | 手动 nohup 启动的进程不受系统管理，无法自愈 | systemd 用户级服务 + `enable --now` + `loginctl enable-linger` |
| 装 Claude Code | install.sh 返回 403 | VPS 出口 IP 被目标站点风控拦截 | 换用 npm 安装路径 |
| 装 Claude Code | Node 版本过旧（v12 vs 需要 v22+） | 系统自带 Node 太老 | nvm 装用户级 Node 22，不动系统环境 |
| 装 Claude Code | npm EACCES 权限错误 | 默认装到系统目录，无写权限 | nvm 管理的 npm 全局目录在用户家目录下，天然规避 sudo 需求 |

一句话总结：**这一整套操作解决的是"服务器在无出国能力、旧系统环境、无 root 权限的三重限制下，如何搭建可用的科学上网通道，并在此基础上装好满足版本要求的 Claude Code"这一个复合问题。**
