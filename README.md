# vw-build

为自建 Vaultwarden 提供的二进制构建与发布仓库。

## 为什么需要这个仓库

| 问题 | 这里的解法 |
|---|---|
| Vaultwarden 官方**不发布独立二进制**，只有容器镜像 | 从官方镜像提取 `/vaultwarden`，作为 release 资产发布 |
| 服务器在内地，直连 GitHub release 经常被掐断 | 只依赖 API（通）+ 加速前缀下载（通），链路单一 |

## 技术栈

| 组件 | 方案 | 来源 |
|---|---|---|
| Vaultwarden | 从官方镜像提取二进制 | 本仓库自动构建 |
| 反向代理 | **nginx 1.26.3** | Debian 13 官方源，`apt install nginx` |
| TLS 证书 | **acme.sh + Cloudflare DNS-01** | 无需 80/443 对外可达 |
| 数据库 | SQLite | 内置 |

### 为什么用 nginx

需要做到"非目标域名/路径在服务器最外层直接断连"，nginx 是唯一能完整实现的：

- **`ssl_reject_handshake on`**（stream 层）—— SNI 不是目标域名时直接拒绝 TLS 握手，不返回证书
- **`$ssl_preread_server_name` 为空**（明文 HTTP 打到 443）—— 丢进黑洞端口，连接被 RST
- **`return 444`**（http 层）—— Host 伪造、路径不符、缺少校验头时直接关连接，零响应

对比：Caddy 对明文请求返回硬编码 400；lighttpd 无法在 HTTP 层断连；Apache 的 400 更啰嗦且内存占用大。

### 架构

```
客户端 :443
   └─ nginx stream（ssl_preread，不解密）
        ├─ SNI = bit.vaultwarden.ifyi.top → 127.0.0.1:8443
        └─ 其他 / 空（含明文 HTTP）      → 127.0.0.1:1 → 连接被拒
             └─ nginx http :8443 → return 444 / 反代 → Vaultwarden:8080
```

## 自动构建

| Workflow | 时间（北京时间） | 行为 |
|---|---|---|
| `build-vaultwarden.yml` | 每天 04:00 | 查 `dani-garcia/vaultwarden` 最新 release → 已发布则跳过，否则拉镜像提取二进制 → 发布 `vw-<版本>` |

也可以在 Actions 页面手动触发，勾选 `force` 可强制重建同名 release。

## Release 约定

| Release tag | 资产 |
|---|---|
| `vw-1.37.4` | `vaultwarden-linux-amd64`、`SHA256SUMS` |

下载地址格式：

```
https://github.com/<owner>/<repo>/releases/download/<tag>/<asset>
```

内地服务器可在前面加 `https://ghfast.top/` 走加速。

## 服务器侧使用

```bash
# 校验二进制完整性
sha256sum -c SHA256SUMS

# Vaultwarden 需要宿主机有 libmariadb3 libpq5
apt-get install -y libmariadb3 libpq5
```

## 证书续期

acme.sh 通过 Cloudflare DNS-01 签发，不依赖任何对外端口：

```bash
# 签发
export CF_Token="<token>"
acme.sh --issue --dns dns_cf -d bit.vaultwarden.ifyi.top

# 安装到 nginx 并在续期后自动 reload
acme.sh --install-cert -d bit.vaultwarden.ifyi.top \
  --key-file       /opt/vaultwarden/conf/ssl/privkey.pem \
  --fullchain-file /opt/vaultwarden/conf/ssl/fullchain.pem \
  --reloadcmd      "systemctl reload nginx"
```

cron 由 `acme.sh --install-cronjob` 安装，每天 4 次检查。

## 安全说明

- 二进制来源可追溯：Vaultwarden 来自官方镜像层
- 每次发布都附带 `SHA256SUMS`，更新脚本会校验后才替换
- 本仓库只用 `GITHUB_TOKEN`，无需配置任何 secret
- 反向代理与证书工具均走 Debian 官方源 / 官方安装脚本，不引入第三方构建产物
