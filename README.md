# sing-box-debian

Debian / Ubuntu 一键搭建 **vless-reality + hysteria2**，systemd 管理，支持 x86 与 ARM。

姊妹项目：[sing-box-onekey](https://github.com/ttbb1211/sing-box-onekey)（Alpine / OpenRC 专版）。
两者**命令名一致**（`sb`）、**目录与服务名完全隔离**（`/etc/sb-onekey` + `sb-onekey.service`），
可以和 fscarmen 的 `sing-box` 同机共存互不影响。

**版本与改动记录：[Releases](https://github.com/ttbb1211/sing-box-debian/releases)** · [全部提交](https://github.com/ttbb1211/sing-box-debian/commits/main)

---

## 为什么有这个项目

onekey 只能跑 Alpine，而实际手上的 VPS 大多是 Debian/Ubuntu。本脚本**复用 onekey 的全部业务逻辑**
（配置生成、证书、分享链接、交互菜单、doctor 体检、备份回滚），只替换平台层：

| 层 | onekey（Alpine） | 本项目（Debian） |
|---|---|---|
| 包管理 | `apk add` | `apt-get install -y` |
| init | OpenRC（`/etc/init.d` + `rc-service`） | systemd（`.service` unit + `systemctl`） |
| nginx 站点目录 | `/etc/nginx/http.d` | `/etc/nginx/conf.d` |
| 模块持久化 | `/etc/modules` + `rc-update` | `/etc/modules-load.d` + `systemd-sysctl` |
| 运行时依赖 | 需 `gcompat`（glibc 二进制跑在 musl 上） | 原生 glibc，**无需兼容层** |
| 日志 | `/var/log/sb-onekey.log` | `journalctl -u sb-onekey` |

适配层用**同名函数**接管 `rc-service` / `rc-update`，主体 1900 行代码一行未改 —— 减少引入新 bug 的面。

---

## 支持范围

| 项 | 范围 |
|---|---|
| 系统 | Debian 10 / 11 / 12 / 13、Ubuntu 20.04 / 22.04 / 24.04 |
| init | systemd |
| 架构 | `x86_64`(amd64)、`aarch64`(arm64)、`armv7`、`armv6`、`i386` |
| 协议 | vless-reality（xtls-rprx-vision）+ hysteria2 |
| 证书 | `steal`（借用大厂证书 + hy2 自签，默认）/ `own`（自有域名 + Let's Encrypt 真证书） |

---

## 安装

```bash
bash <(curl -fsSL https://raw.githubusercontent.com/ttbb1211/sing-box-debian/main/sing-box-debian.sh)
```

非交互：

```bash
# 只装 vless-reality
PROTOCOLS=vless bash sing-box-debian.sh

# 全参数
PROTOCOLS=vless,hy2 VLESS_PORT=443 HY2_PORT=8443 SNI=www.example.com NODE_NAME=hk-1 bash sing-box-debian.sh
```

安装完脚本自身会被装成命令 `sb`：

```bash
sb            # 交互菜单
sb -n         # 查看节点信息与分享链接
sb -e         # 修改配置（端口 / SNI / 节点名）
sb -p         # 增减协议
sb -v         # 升级 sing-box 内核
sb -b         # 开启 BBR（含持久化）
sb doctor     # 体检
sb domain     # 一键自建伪装站点（签 LE 证书 + nginx 回落 + 改配置 + 出链接）
sb -r         # 删除节点（保留 sb 命令）
sb -u         # 彻底卸载
```

---

## own 模式：让 hy2 链接不再需要 pin

默认 `steal` 模式用自签证书，hysteria2 的分享链接必须带 `pinSHA256`（或 `insecure=1`），
换客户端、换设备时容易漏参数导致连不上。

`sb domain` 走 `own` 模式：用自己的域名签 Let's Encrypt 真证书，nginx 提供本地回落站，
reality 的 `handshake` 指向 `127.0.0.1:8001`，hy2 直接读同一张 LE 证书。这样链接就是干净的：

```
hysteria2://<password>@<ip>:<port>?sni=你的域名#节点名
```

**不需要 `hpkp`，也不需要 `insecure=1`**，任何客户端导入即可用。

配套注意：

- 域名必须是**灰云（DNS only）**直接解析到本机，橙云（代理）会导致 ACME HTTP-01 失败
- 80 端口需放行（ACME 验证 + 回落站前置）
- `acme.sh` 续期钩子会自动调 `sb reload-cert`（nginx reload + 重启 sing-box），证书到期前自动换新

**443 归谁，按 `VLESS_PORT` 决定**（v1.3 起）：

| `VLESS_PORT` | 443 归 | 结果 |
|---|---|---|
| `443` | Reality | nginx 只监听 80 + `127.0.0.1:8001`，浏览器访问域名走 Reality 回落看到站点 |
| `≠443` | nginx | nginx 额外 `listen 443 ssl` 直接提供伪装站，浏览器访问域名正常打开 |

早期版本在 `VLESS_PORT≠443` 时把 80 无条件 301 到空着的 443，浏览器会卡在「此网站不支持安全连接」。升级后重跑 `sb domain` 即可修好；443 被别的进程占着会自动跳过，只影响伪装不影响节点。

---

## 从 fscarmen 迁移

两者可同机共存，但端口不能冲突。迁移步骤：

```bash
# 1. 装本脚本，用不同的端口避开现有节点
PROTOCOLS=vless,hy2 VLESS_PORT=443 HY2_PORT=8443 bash sing-box-debian.sh

# 2. 验证新节点可用后，停掉并卸载旧节点
sb -n                      # 确认新节点正常
# 旧节点按 fscarmen 的菜单卸载
```

⚠️ 如果 443 已被 fscarmen 的 reality 占用，先停旧服务再装新的，或反过来先装新的用别的端口。

---

## 安全说明

- 配置目录 `/etc/sb-onekey` 权限 700，`config.json` 与 `params.env` 权限 600
- 不写入任何第三方脚本的目录，卸载时只删自己创建的文件
- `steal` 模式的 reality 密钥、hy2 自签证书均在本地生成，不上传任何地方

---

## License

MIT
