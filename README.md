# vw-build

为自建 Vaultwarden 提供的二进制构建与发布仓库。

## 为什么需要这个仓库

| 问题 | 这里的解法 |
|---|---|
| Vaultwarden 官方**不发布独立二进制**，只有容器镜像 | 从官方镜像提取 `/vaultwarden`，作为 release 资产发布 |
| 服务器在内地，直连 GitHub release 经常被掐断 | 只依赖 API（通）+ 加速前缀下载（通），链路单一 |

## 关于反向代理

前置反向代理已从 **Caddy 换成 Debian 官方 `lighttpd`**，因此**不再需要自建构建**：

| 项目 | 说明 |
|---|---|
| 来源 | Debian 13 (trixie) 官方源，`apt install lighttpd lighttpd-mod-openssl lighttpd-mod-magnet` |
| 版本 | 1.4.79-2（安全更新由 Debian 安全团队跟进，`apt upgrade` 即可） |
| 为什么换 | Caddy 对"明文 HTTP 打到 443"会返回硬编码的 `400 Client sent an HTTP request to an HTTPS server.`，无法配置消除；lighttpd 在 SSL 握手阶段**直接断开连接，零响应** |
| 证书 | 由 acme.sh 通过阿里云 DNS-01 签发（不依赖 80/443 对外可达） |

所以本仓库现在只保留 Vaultwarden 的构建流水线。

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

## 安全说明

- 二进制来源可追溯：Vaultwarden 来自官方镜像层
- 每次发布都附带 `SHA256SUMS`，更新脚本会校验后才替换
- 本仓库只用 `GITHUB_TOKEN`，无需配置任何 secret
- 反向代理 lighttpd 走 Debian 官方源，不引入第三方构建产物
