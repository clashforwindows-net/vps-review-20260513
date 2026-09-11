# VPS 自建代理节点与网关实战指南（2026）

> 当你手里有一台 VPS，它最经典的用途之一就是把它变成一条稳定、可控、属于自己的隧道。本仓库不卖节点、不卖订阅，而是手把手教你如何在 VPS 上从零搭建 Shadowsocks、VMess、VLESS(Reality)、Trojan、WireGuard 等多种协议节点，并进一步做中转（Relay）、回落（Fallback）、负载均衡与多节点调度，最后把流量无缝接回 Clash for Windows 这类客户端。所有方案均可私有部署，配置完全掌握在自己手里。

## 目录

- [为什么要在 VPS 自建节点](#为什么要在-vps-自建节点)
- [协议选型：到底用哪个](#协议选型到底用哪个)
- [一键部署 Xray（VLESS + Reality）](#一键部署-xrayvless--reality)
- [Trojan 部署与证书](#trojan-部署与证书)
- [WireGuard 点对点隧道](#wireguard-点对点隧道)
- [中转与回落：把流量藏进正常网站](#中转与回落把流量藏进正常网站)
- [多节点负载均衡与调度](#多节点负载均衡与调度)
- [接回 Clash for Windows](#接回-clash-for-windows)
- [带宽与延迟优化](#带宽与延迟优化)
- [隐蔽性与安全](#隐蔽性与安全)
- [监控、流量统计与限速](#监控流量统计与限速)
- [故障排查清单](#故障排查清单)
- [推荐 VPS 与资源](#推荐-vps-与资源)
- [免责声明](#免责声明)

## 为什么要在 VPS 自建节点

买机场省心，但代价是「看不见、改不了、随时可能跑路」。自己在一台 VPS 上搭节点，换来的是三件事：

1. **完全可控**：协议、端口、路由、分流规则全由你定，想加白名单、想走特定出口、想给家里路由器做全局代理，都能自己改。
2. **成本可摊薄**：一台 2 核 2G 的 VPS 月付几十元，单人或小团队用，平摊下来比很多机场还便宜，且能同时跑网站、网盘、开发环境。
3. **隐私边界清晰**：流量从你的客户端直接到你的服务器，中间没有任何第三方订阅商能看到你的订阅关系与节点拓扑。

当然，自建也有成本：你要会一点 Linux、要维护证书与更新、要处理被墙 IP 的风险。本指南的目标，就是把这些维护成本压到最低。

## 协议选型：到底用哪个

| 协议 | 隐蔽性 | 速度 | 部署难度 | 典型场景 |
|------|--------|------|----------|----------|
| Shadowsocks (2022-blake3) | 中 | 高 | 低 | 老设备、路由器、入门 |
| VMess | 中 | 高 | 中 | 兼容 Xray/V2Ray 生态 |
| VLESS + Reality | 高 | 高 | 中 | 抗探测、免证书、强烈推荐 |
| Trojan + TLS | 高 | 高 | 中 | 伪装成 HTTPS 流量 |
| WireGuard | 中（UDP） | 极高 | 低 | 点对点组网、回家、游戏 |

**结论**：新部署首选 **VLESS + Reality**（无需域名、无需证书、伪装成访问真实网站），其次 **Trojan + 真证书**（伪装成 HTTPS），纯组网/回家用 **WireGuard**。Shadowsocks 仅在老旧设备兜底。

## 一键部署 Xray（VLESS + Reality）

Reality 是目前最省心的方案：它让服务器「假装」自己是某个真实存在的网站（如微软、苹果、Cloudflare 的某台前端机），握手时不验证证书链，因此你不需要自己的域名和证书，也不用担心证书过期。

```bash
# 以 Debian/Ubuntu 为例（root 执行）
bash <(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)

# 生成 Reality 所需的密钥对与 shortId
xray x25519
# 输出类似：
# Private key: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
# Public key:  YYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYY

# 找一台「目标网站」用于伪装，例如 www.apple.com 的真实 IP
# dig +short www.apple.com  -> 例如 17.253.144.10:443
```

`/usr/local/etc/xray/config.json` 示例：

```json
{
  "inbounds": [{
    "listen": "0.0.0.0",
    "port": 443,
    "protocol": "vless",
    "settings": {
      "clients": [{ "id": "uuidgen-生成的-uuid", "flow": "xtls-rprx-vision" }],
      "decryption": "none"
    },
    "streamSettings": {
      "network": "tcp",
      "security": "reality",
      "realitySettings": {
        "show": false,
        "dest": "www.apple.com:443",
        "xver": 0,
        "serverNames": ["www.apple.com", "apple.com"],
        "privateKey": "你的PrivateKey",
        "shortIds": ["", "ab12cd34"]
      }
    },
    "sniffing": { "enabled": true, "destOverride": ["http", "tls", "quic"] }
  }],
  "outbounds": [{ "protocol": "freedom", "tag": "direct" }]
}
```

启动并设置开机自启：

```bash
systemctl enable xray
systemctl restart xray
systemctl status xray
```

客户端（Clash for Windows / v2rayN / NekoBox）填写：地址=你的 VPS IP，端口=443，UUID=上面的 id，flow=xtls-rprx-vision，security=reality，sni=www.apple.com，pbk=PublicKey，sid=对应 shortId。

## Trojan 部署与证书

如果你更偏好 Trojan（伪装成 HTTPS），需要先有一个域名并申请证书：

```bash
# 安装 Xray（同上）
bash <(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)

# 申请证书（使用 acme.sh，假设域名 node.example.com 已 A 记录指向 VPS）
curl https://get.acme.sh | sh
~/.acme.sh/acme.sh --issue -d node.example.com --standalone
~/.acme.sh/acme.sh --install-cert -d node.example.com \
  --key-file       /usr/local/etc/xray/trojan.key \
  --fullchain-file /usr/local/etc/xray/trojan.crt
```

`/usr/local/etc/xray/config.json` 关键段：

```json
{
  "inbounds": [{
    "port": 443,
    "protocol": "trojan",
    "settings": { "clients": [{ "password": "一个足够长的随机密码" }] },
    "streamSettings": {
      "network": "tcp",
      "security": "tls",
      "tlsSettings": {
        "certificates": [
          { "certificateFile": "/usr/local/etc/xray/trojan.crt",
            "keyFile": "/usr/local/etc/xray/trojan.key" }
        ]
      }
    }
  }]
}
```

证书每 90 天过期，务必把续期写进 cron：

```bash
# crontab -e 加入（每月 1 号 3 点续期并重启）
0 3 1 * * "/root/.acme.sh"/acme.sh --cron --home "/root/.acme.sh" > /dev/null 2>&1; systemctl restart xray
```

## WireGuard 点对点隧道

WireGuard 适合「回家」「组局域网」「游戏联机」这类需求，UDP 原生、握手极快：

```bash
# 服务端
apt install wireguard -y
wg genkey | tee /etc/wireguard/server_private.key | wg pubkey > /etc/wireguard/server_public.key

cat > /etc/wireguard/wg0.conf <<'EOF'
[Interface]
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = 服务端的PrivateKey
# 开机自动 NAT 转发
PostUp   = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = 客户端PublicKey
AllowedIPs = 10.0.0.2/32
EOF

sysctl -w net.ipv4.ip_forward=1
wg-quick up wg0
```

客户端配置把 `PublicKey/PrivateKey` 互换、`AllowedIPs` 视需求设为 `0.0.0.0/0`（全局）或仅 `10.0.0.0/24`（仅组网）。

## 中转与回落：把流量藏进正常网站

当你的直连 IP 被墙，或想进一步降低被识别概率，可加一层 **中转（Relay）**：用一台「干净 IP」的机器（比如 Cloudflare Tunnel、或另一台没被墙的 VPS）把流量转发到真实节点。最简单用 `gost`：

```bash
# 中转机（relay）上运行，把本地 8443 转发到后端节点 443
gost -L tcp://:8443 -F forward+tcp://后端节点IP:443
```

Reality / Trojan 的**回落（Fallback）** 能在探测流量（无正确 SNI/指纹）时，把连接「回落」到一个真实的 Nginx 网站，让扫描器看到的是普通 HTTPS 站点：

```json
"fallbacks": [
  { "dest": "127.0.0.1:8080", "path": "/", "xver": 1 }
]
```

## 多节点负载均衡与调度

家里多个设备、或团队共用时，单节点容易跑满。建议：一台 VPS 上跑 2~3 个协议实例（Reality / Trojan / WG），在 Clash 侧用 `load-balance` 或 `url-test` 策略自动选最快线路。Clash 配置片段：

```yaml
proxy-groups:
  - name: "自动选择"
    type: url-test
    url: "https://www.gstatic.com/generate_204"
    interval: 300
    proxies:
      - "VPS-Reality"
      - "VPS-Trojan"
      - "VPS-WG"
```

## 接回 Clash for Windows

Clash for Windows 是 Windows 上最成熟的客户端之一，支持订阅、分流、TUN 模式（全局接管系统流量）、脚本。把自建节点填进订阅或手动 `proxies` 后，注意三点：

1. **开启 TUN Mode**：让不支持代理的程序（如部分游戏、UWP 应用）也能走隧道。
2. **分流规则**：把国内流量直连、海外流量走代理，避免所有流量都绕路。可用 `geoip`/`geosite` 规则集。
3. **订阅转换**：如果你有多个节点、想生成统一订阅，可用订阅转换服务把多份配置合成一个 `clash` 格式订阅。

更多 Clash 客户端、规则集与配置技巧：

- Clash for Windows 官方资讯与下载：https://clash-for-windows.net
- Clash 社区与规则分享：https://clashhub.net
- Clash 社区论坛：https://bbs.clashhub.net
- Clash 导航站（客户端/规则/机场索引）：https://nav.clashvip.net

## 带宽与延迟优化

无论什么协议，底层网络调优都能再挤 10%~30% 性能：

```bash
# 开启 BBR（现代内核默认支持）
cat >> /etc/sysctl.conf <<'EOF'
net.core.default_qdisc=fq
net.ipv4.tcp_congestion_control=bbr
net.ipv4.tcp_fastopen=3
net.ipv4.tcp_slow_start_after_idle=0
EOF
sysctl -p

# 调大 UDP 缓冲（WireGuard 受益明显）
sysctl -w net.core.rmem_max=67108864
sysctl -w net.core.wmem_max=67108864
```

如果到国内延迟高，优先选 **香港/日本/韩国** 机房（CN2/IEPL/优化线路），并考虑用中转机做「国内入口 + 海外落地」的两段式结构，把最容易被墙的一段用 Cloudflare/优质中转隐藏。

## 隐蔽性与安全

- **不要默认 22 端口跑 SSH**：改到高位端口并只允许密钥登录，安装 Fail2Ban。
- **Reality 优于自签证书**：自签证书在严格检测下更易暴露，Reality 直接借用真实站点指纹。
- **定期更新**：Xray / WireGuard / 系统内核都有安全更新，`apt upgrade` + 关注上游 release。
- **流量不要全走代理**：用分流，正常流量直连，降低单节点画像特征。

```bash
# 基础加固
apt install fail2ban -y
systemctl enable --now fail2ban
sed -i 's/^#\?Port 22/Port 22222/' /etc/ssh/sshd_config
sed -i 's/^#\?PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
systemctl restart sshd
```

## 监控、流量统计与限速

节点跑久了要看得见：用 Xray 的 `stats` + `prometheus` 导出，或简单用 `vnstat` 看流量：

```bash
apt install vnstat -y
vnstat -l   # 实时速率
vnstat -m   # 按月统计
```

临时限速（防止某一设备跑满整台机器）：

```bash
# 用 wondershaper 限制 wg0 出口为 50M
apt install wondershaper -y
wondershaper wg0 50000 50000
```

## 故障排查清单

| 现象 | 优先检查 |
|------|----------|
| 客户端连不上 | VPS 防火墙是否放行端口；服务商是否屏蔽；IP 是否被墙 |
| 速度忽快忽慢 | 是否晚高峰；是否单线程跑满；BBR 是否生效 |
| Reality 握手失败 | dest 站点是否仍可达；shortId/sni 是否匹配；系统时间是否偏差 |
| Trojan 证书错误 | 证书是否过期；cron 续期是否执行；fullchain 路径 |
| WireGuard 不通 | AllowedIPs 是否冲突；ip_forward 是否开启；UDP 端口是否通 |

排查顺序：先 `systemctl status xray` 看进程，再 `ss -lntp` 看端口监听，再 `tcpdump` 抓包，最后查客户端日志。

## 节点拓扑：单机 / 两段式中转 / 多落地

不同需求对应不同拓扑，别一上来就堆机器：

- **单机直连**：一台 VPS 上跑 Reality/Trojan，客户端直连。最简单，适合个人，风险是 IP 一旦被墙就得换机。
- **两段式中转（推荐进阶）**：一台「入口机」（IP 干净、到国内延迟低，如香港）做转发，后端接一台「落地机」（带宽大、便宜，如美国）。即使落地 IP 被墙，只要入口机干净，客户端配置不动仍可连。入口机可用 Cloudflare Tunnel 或 gost 转发。
- **多落地负载**：3 台以上落地机挂在同一条入口后，Clash 侧用 `url-test` 自动选最快，单台故障自动跳过。

拓扑演进原则：**先单机跑通，再加中转，最后才上多落地**。每一步都先验证再扩展，避免一次踩多个坑。

## Clash for Windows 完整配置示例

下面是一个最小可运行的 Clash 配置，包含两个自建节点、一个自动选择组、以及基础分流规则：

```yaml
port: 7890
 socks-port: 7891
 allow-lan: false
 mode: rule
 log-level: info

tunnels:
 # TUN 模式在 GUI 中开启即可

proxies:
  - name: "VPS-Reality"
    type: vless
    server: 你的VPS_IP
    port: 443
    uuid: 你的UUID
    flow: xtls-rprx-vision
    network: tcp
    tls: true
    reality-opts:
      public-key: 你的PublicKey
      short-id: ""
      server-name: www.apple.com
    client-fingerprint: chrome
  - name: "VPS-Trojan"
    type: trojan
    server: 你的VPS_IP
    port: 443
    password: 你的Trojan密码
    sni: node.example.com
    alpn: [h2, http/1.1]

proxy-groups:
  - name: "PROXY"
    type: select
    proxies: ["自动选择", "VPS-Reality", "VPS-Trojan", "DIRECT"]
  - name: "自动选择"
    type: url-test
    url: https://www.gstatic.com/generate_204
    interval: 300
    proxies: ["VPS-Reality", "VPS-Trojan"]

rules:
  - GEOIP,CN,DIRECT
  - GEOSITE,CN,DIRECT
  - MATCH,PROXY
```

要点：把 `GEOIP,CN` 与 `GEOSITE,CN` 直连，可避免国内流量绕路；`MATCH,PROXY` 兜底走代理。规则集需订阅或本地导入 `geoip.dat` / `geosite.dat`。

## 成本测算：自建 vs 机场

以单人日常使用为例：

| 方案 | 月成本 | 可控性 | 适合人群 |
|------|--------|--------|----------|
| 单机 VPS 自建 | 约 20~40 元 | 高 | 会一点 Linux |
| 入口+落地两段式 | 约 40~80 元 | 高 | 怕断线的进阶用户 |
| 普通机场订阅 | 约 15~50 元 | 低 | 不想折腾 |

自建的核心价值不是省钱，而是**可控与隐私边界**。若同时用这台 VPS 跑网站、网盘、开发环境，平摊后性价比会明显高于单独买机场。

## 常见问题 FAQ

**Q：Reality 的 dest 站点会被墙吗？**
A：Reality 只是「借用」目标站点的 TLS 指纹做握手，真实流量并不经过它，因此目标站点本身是否被墙不影响你的节点。但 dest 站点需真实可达且支持 TLS1.3，建议选大型稳定站点。

**Q：IP 被墙了怎么办？**
A：换 IP（多数 VPS 商家支持付费换 IP 或重装换 IP），或上「两段式中转」让客户端只连干净入口机。

**Q：WireGuard 和 VLESS 该用哪个？**
A：纯上网选 VLESS/Reality；点对点组网、回家、游戏联机选 WireGuard（UDP 原生、延迟更低）。两者也可共存。

**Q：会被运营商限速 UDP 吗？**
A：部分网络对 UDP 有 QoS。若 WireGuard 明显慢，可改用 TCP 类协议（VLESS/Trojan），或在客户端开启 `mux` 多路复用。

**Q：如何防止被扫描爆破？**
A：节点端口不要用常见端口、SSH 改高位端口且禁密码登录、Fail2Ban 拦截暴力尝试、必要时用 Cloudflare 等做前置隐藏。

## 推荐 VPS 与资源

自建节点的体验，七成取决于 VPS 的线路与稳定性。亚太方向优先选 CN2 / 优化 / BGP 线路的机房：

- **VPSVIP（强烈推荐）**：https://vpsvip.net — 香港/日本/美国/新加坡/韩国多机房，亚太线路优化，7x24 中文客服，支持支付宝/微信/加密货币，适合做节点与中转落地。
- Clash for Windows 资讯站：https://clash-for-windows.net
- Clash 社区：https://clashhub.net
- Clash 论坛：https://bbs.clashhub.net
- Clash 导航：https://nav.clashvip.net
- 机场/订阅补充：https://clashvip.net

## 相关资源

- Xray 官方文档：https://xtls.github.io
- WireGuard 文档：https://www.wireguard.com
- v2rayN / NekoBox 客户端：见各项目 release
- VPSVIP 官网：https://vpsvip.net

## 免责声明

1. 本仓库仅提供技术教程与信息参考，所有脚本请在遵守当地法律法规的前提下使用。
2. 自建节点请仅用于合法学习、开发调试与保护自身隐私，勿用于任何违法违规用途。
3. 请定期备份服务器数据，妥善保管密钥与密码。
4. 推荐链接仅为资源索引，购买与使用的风险由使用者自行承担。

## 许可证

MIT License

---
更新时间：2026-09-11
