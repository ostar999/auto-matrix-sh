# Matrix 全栈一键部署脚本

在一台干净的 Ubuntu / Debian 服务器上，用一条命令部署：

- **Synapse** —— Matrix 服务端
- **Element Web** —— 网页客户端
- **LiveKit + lk-jwt** —— 群组语音/视频通话、手机端通话
- **coturn** —— 1:1 通话的 NAT 穿透
- **synapse-admin** —— 图形化管理后台
- **nginx + Let's Encrypt** —— 反向代理与自动 HTTPS

---

## 一、它做了什么

```
① 环境检查     root / 系统 / 架构 / 内存 / 磁盘 / 80·443 是否空闲
② 补 swap      内存 < 2GB 时自动建 2GB swap
③ 装 Docker    没有就装
④ 建 DNS      用 Cloudflare API 建 3 条 A 记录（幂等，冲突会拦）
⑤ 生成配置    渲染 compose / homeserver / element / nginx / livekit / coturn
              所有密钥随机生成，写进 .env
⑥ 起容器      7 个容器，等 Synapse 健康
⑦ 签证书      acme.sh + DNS-01，一张证书覆盖 3 个域名，自动续期
⑧ 配站点      nginx 三站点：网页 / API / 后台
⑨ 建管理员    register_new_matrix_user
⑩ 自检        6 项本机检查 + 公网可达性检查
```

## 二、前置条件

| 条件 | 说明 |
| --- | --- |
| 系统 | Ubuntu 20.04+ / Debian 11+，root 权限 |
| 域名 | 一个域名，NS 托管在 **Cloudflare** |
| CF Token | 需要 `Zone→DNS→Edit` + `Zone→Zone→Read` 两项权限 |
| 端口 | 云厂商安全组必须放行 80、443 以及通话用的 UDP 段（见 2.1） |
| 磁盘 | 建议 ≥ 20GB；单文件上限默认 2G |

> ⚠️ **最常见的失败原因**：VPS 默认只开放 SSH 端口，80/443 被**服务商安全组**挡住。
> 判断方法：`ufw status` 显示已放行、但 `tcpdump` 在服务器上抓不到任何包
> → 拦截在服务商网络层。部署完成后如果公网返回 522/超时，先去控制台放行：
>
> ```
> TCP 80, 443, 7881, 3478
> UDP 3478, 49160-49200, 50000-50100
> ```
>
> 具体到 RackSmart：产品详情 → 安全组 → 管理 → 添加（详见 PITFALLS 6.2）。

## 三、快速开始

### 3.1 上传脚本

```bash
scp -r 3-auto-matrix-sh root@你的服务器:/root/auto-matrix
ssh root@你的服务器
cd /root/auto-matrix
```

### 3.2 交互式运行（推荐第一次用）

```bash
bash install.sh
```

会依次询问：

```
? 主域名: example.com
? 网页前缀（用户 ID 会用这个）: chat
   → 接口域名 chatapi.example.com
   → 后台域名 chatad.example.com
? 接口前缀 [chatapi]:
? 后台前缀 [chatad]:
? 管理员用户名 [ostar]:
? 管理员密码: ********
? 证书邮箱 [admin@example.com]:
```

### 3.3 一条命令（全自动）

```bash
bash install.sh --non-interactive \
  --domain example.com \
  --web-host chat \
  --cf-token "你的 Cloudflare Token" \
  --admin-user ostar \
  --admin-pass "你的密码"
```

### 3.4 先看会生成什么（不启动）

```bash
bash install.sh --dry-run --skip-dns --non-interactive \
  --domain example.com --web-host chat --admin-pass "test12345"
# 配置会渲染到 /opt/matrix，但不启动任何服务
```

## 三·五、网站由谁托管（重要）

脚本会先判断这台机器上有没有 1Panel，再决定网站交给谁：

| 模式 | 触发条件 | 80/443 归谁 | 1Panel 里能看到站点吗 |
| --- | --- | --- | --- |
| `standalone` | 没有 1Panel | 脚本装的 nginx | ❌ 看不到 |
| `1panel` | 装了 1Panel | 1Panel 的 OpenResty | ✅ 能看到、能管理 |
| `install-1panel` | 想装 1Panel | 装完由你初始化 | ✅ |

选择逻辑：

```bash
# 交互式：检测到 1Panel 会问你要不要用它管理
bash install.sh

# 显式指定
bash install.sh --panel 1panel      # 用 1Panel 管理
bash install.sh --panel standalone  # 用独立 nginx
```

**1Panel 模式做什么 / 不做什么**

- ✅ 起容器、建 DNS、签证书、建管理员
- ✅ 把三个站点的 nginx 片段生成到 `/opt/matrix/nginx/`
- ❌ 不装宿主机 nginx（避免和 1Panel 抢 80/443）
- ⏸ 最后会打印「在 1Panel 里建 3 个站点」的步骤，由你点 3 次完成

**1Panel 模式建完站后，还要做三件事**（脚本会打印清单）：

1. 证书上传后，各站点 HTTPS 开关打开；
2. API 站点补 `/livekit/sfu/`、`/livekit/jwt/` 和 `client_max_body_size 0`；
3. 后台站点开「密码访问」，否则面板裸奔。

续期钩子 `renew-hook.sh` 会自动生成，把 acme.sh 的 reloadcmd 指向它即可。

> 为什么不能全自动？1Panel 的站点记录在它自己的数据库里，
> API Key 在库里是加密存储的（`api_keys.secret_ciphertext`），
> 脚本无法自行生成或读取。所以 1Panel 模式做成「半自动」：
> 脚本把该准备的都准备好，你在面板里点三下即可。

**从 standalone 切到 1panel**

```bash
systemctl disable --now nginx      # 让出 80/443
# 1Panel 应用商店装 OpenResty
bash install.sh --panel 1panel     # 重新跑一遍
```

数据卷不会丢，容器会自动重建。

## 四、参数

| 参数 | 说明 | 默认 |
| --- | --- | --- |
| `--domain` | 主域名（必填） | — |
| `--web-host` | 网页前缀，决定 server_name | `chat` |
| `--api-host` | API 前缀 | `<web>api` |
| `--admin-host` | 后台前缀 | `<web>ad` |
| `--cf-token` | Cloudflare Token | 环境变量 `CF_TOKEN` |
| `--admin-user` | 管理员用户名 | `ostar` |
| `--admin-pass` | 管理员密码 | 交互输入 |
| `--email` | 证书邮箱 | `admin@<domain>` |
| `--brand` | Element 显示名 | `Matrix Chat` |
| `--dir` | 安装目录 | `/opt/matrix` |
| `--max-upload` | 单文件上限 | `2G` |
| `--ip` | 公网 IP | 自动探测 |
| `--panel` | `auto` / `1panel` / `standalone` | `auto` |
| `--panel-api-key` | 1Panel API Key（预留，暂未用于自动建站） | — |
| `--skip-dns` | 不建 DNS 记录 | — |
| `--skip-rtc` | 不装 LiveKit/coturn | — |
| `--no-proxy-web` | 网页域名也不走 CF 代理 | — |
| `--non-interactive` | 不提问 | — |
| `--force` | 覆盖已有配置与密钥 | — |
| `--dry-run` | 只渲染不启动 | — |

## 五、域名与 Cloudflare 设置

脚本会建三条记录：

| 记录 | 用途 | 代理 |
| --- | --- | --- |
| `chat.example.com` | Element 网页 + 用户 ID | 橙云（CF 代理） |
| `chatapi.example.com` | API + 媒体 + LiveKit + TURN | **灰云（仅 DNS）** |
| `chatad.example.com` | 管理后台 | 橙云 |

**`chatapi` 必须是灰云**，否则大文件上传会被 Cloudflare 卡在 100MB。

三张都是一级子域，所以一张 `*.example.com` 通配符证书也能覆盖。

## 六、部署完成后

| 地址 | 说明 |
| --- | --- |
| `https://chat.example.com` | Element 网页 |
| `https://chatad.example.com` | 管理后台（Basic Auth + Matrix 账号两层） |
| `https://chatapi.example.com` | API（客户端自动发现，不用手填） |

管理后台登录：
1. 浏览器弹窗：Basic Auth 用户名/密码 = 安装时输入的管理员账号密码（htpasswd 里是同一个）
2. 面板内：服务器地址填 `https://chatapi.example.com`，用 Matrix 账号登录

## 七、常用运维

```bash
cd /opt/matrix

docker compose ps                          # 状态
docker compose logs -f synapse             # 日志
docker compose restart synapse             # 改配置后重启

# 加用户（去 -a 就是普通用户）
docker compose exec synapse register_new_matrix_user \
  -c /data/homeserver.yaml -a -u alice http://localhost:8008

# nginx
nginx -t && systemctl reload nginx

# 证书续期（acme.sh 已自动加 cron）
/root/.acme.sh/acme.sh --list
```

## 八、重跑与卸载

脚本是幂等的：重复执行会复用 `.env` 里的密钥、跳过已存在的 DNS 记录、
复用已签发的证书。

```bash
# 重跑（改配置）
bash install.sh --non-interactive --domain ... --web-host ...

# 想换密钥（会重新生成，数据库密码也变，需要重建库）
bash install.sh --force ...

# 卸载
cd /opt/matrix && docker compose down
rm -rf /opt/matrix
rm -f /etc/nginx/sites-enabled/{chat,chatapi,chatad}.example.com.conf
```

## 九、目录结构

```
3-auto-matrix-sh/
├── install.sh              # 主脚本，所有逻辑在这
├── templates/              # 配置模板（用 @占位符@）
│   ├── docker-compose.yaml.tmpl
│   ├── homeserver.yaml.tmpl
│   ├── element.config.json.tmpl
│   ├── nginx-web.conf.tmpl
│   ├── nginx-api.conf.tmpl
│   ├── nginx-admin.conf.tmpl
│   ├── livekit.yaml.tmpl
│   ├── lk-jwt-registration.yaml.tmpl
│   └── turnserver.conf.tmpl
├── PITFALLS.md             # 开发踩坑记录
└── README.md               # 本文件

部署后服务器上的布局：
/opt/matrix/
├── .env                    # 全部密钥（600）
├── docker-compose.yaml
├── synapse/homeserver.yaml
├── element/config.json
├── livekit/{livekit.yaml,lk-jwt-registration.yaml}
├── coturn/turnserver.conf
├── nginx/*.conf            # 站点配置副本
└── data/{postgres,synapse} # 数据卷
```

## 十、常见问题

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 公网返回 522 / 超时 | 云防火墙没放行 80/443 | 控制台放行端口 |
| `password authentication failed` | `data/postgres` 残留旧密码 | `docker compose down && rm -rf data/postgres` 重跑 |
| Element 登录转圈 | `.well-known` 没生效 | `curl https://chat.域名/.well-known/matrix/client` |
| 通话打不开 | lk-jwt 连 8448 超时 | 检查 `.well-known/matrix/server` 是否存在 |
| 大文件上传失败 | `chatapi` 走了橙云 | 改成灰云 |
| Synapse 反复重启 | `/data` 属主不是 991 | `chown -R 991:991 data/synapse` |

更多细节见 [PITFALLS.md](PITFALLS.md)。
