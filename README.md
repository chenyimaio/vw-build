# vw-build

为自建 Vaultwarden 提供的二进制构建与发布仓库。

## 为什么需要这个仓库

| 问题 | 这里的解法 |
|---|---|
| Vaultwarden 官方**不发布独立二进制**，只有容器镜像 | 从官方镜像提取 `/vaultwarden`，作为 release 资产发布 |
| Caddy 官方预编译版**用的 Go 版本落后**，可能受 CVE-2026-111943 影响 | 用最新稳定版 Go 从源码编译，并在构建时校验 Go 版本 |
| 服务器在内地，直连 GitHub release 经常被掐断 | 只依赖 API（通）+ 加速前缀下载（通），链路单一 |

## 自动构建

| Workflow | 时间（北京时间） | 行为 |
|---|---|---|
| `build-vaultwarden.yml` | 每天 04:00 | 查 `dani-garcia/vaultwarden` 最新 release → 已发布则跳过，否则拉镜像提取二进制 → 发布 `vw-<版本>` |
| `build-caddy.yml` | 每天 05:00 | 查 `caddyserver/caddy` 最新 release → 用最新稳定 Go 编译（构建前校验 Go ≥ 1.26.10 且不在 1.27.0~1.27.2）→ 发布 `caddy-<版本>` |

也可以在 Actions 页面手动触发，勾选 `force` 可强制重建同名 release。

## Release 约定

| Release tag | 资产 |
|---|---|
| `vw-1.37.4` | `vaultwarden-linux-amd64`、`SHA256SUMS` |
| `caddy-v2.11.7` | `caddy-linux-amd64`、`SHA256SUMS` |

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

- 二进制来源可追溯：Vaultwarden 来自官方镜像层，Caddy 来自官方源码 + 官方 Go 工具链
- 每次发布都附带 `SHA256SUMS`，更新脚本会校验后才替换
- 本仓库只用 `GITHUB_TOKEN`，无需配置任何 secret
