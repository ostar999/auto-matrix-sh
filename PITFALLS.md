# 开发踩坑记录

写这个脚本时踩到的坑，按「现象 / 原因 / 修复」记录，避免下次再掉进去。
每一条都在这台测试服务器上真实发生过。

---

## 一、bash 本身

### 1.1 `set -u` 下 `local` 同行引用会报 unbound

**现象**

```
install.sh: line 240: m: unbound variable
```

**出问题的写法**

```bash
cf_api() {
  local m=$1 p=$2 args=(-X "$m" "https://...")
  #                    ^^^^ 这里展开时 m 还是 unset
}
```

**原因**：`local` 会先把所有变量名声明为局部变量（值为空），
再执行赋值。同一条 `local` 语句后面引用的 `$m`，展开发生在赋值之前，
配合 `set -u` 就直接报错。

**修复**：拆成多条语句。

```bash
cf_api() {
  local m=$1
  local p=$2
  local -a args=( -X "$m" "https://..." )
}
```

### 1.2 `pkill -f` 会杀掉自己的 SSH 会话

**现象**

```bash
ssh root@host 'pkill -f "bash install.sh"; ...其余命令...'
# 连接直接断开，后面的命令一条都没执行
```

**原因**：`ssh` 把整条远程命令作为参数传给远端 shell，
这条 `bash -c '...pkill -f "bash install.sh"...'` 的命令行本身就包含
`bash install.sh` 字样，`pkill -f` 匹配到了自己。

**修复**：用 PID 杀，或把匹配串写成不会匹配自己的形式（如 `[i]nstall.sh`）。

```bash
ps -eo pid,cmd | awk '/[b]ash install.sh/{print $1}' | xargs -r kill
```

### 1.3 远程跑长任务要用 `setsid nohup`

**现象**：安装到一半 SSH 超时断开，脚本跟着被杀。

**修复**

```bash
setsid nohup bash install.sh ... > /root/install.log 2>&1 < /dev/null &
```

`< /dev/null` 和 `setsid` 缺一不可，否则要么被 SIGHUP 杀掉，要么
SSH 一直挂着不返回（stdout 被后台进程继承）。

### 1.4 heredoc 里的 `$` 与反引号

**现象**：nginx 模板渲染后 `$http_upgrade` 变成了空。

**原因**：用未加引号的 heredoc 写模板，`$` 被 shell 展开。

**修复**：模板走独立文件 + Python 替换 `@占位符@`，
或者 heredoc 用 `<<'EOF'`（引号阻止展开），需要变量时再单独处理。

### 1.5 `set -e` 配 `trap ERR` 报的行号是 trap 的

**现象**：报错说「第 254 行执行失败」，但 254 行是 `trap` 定义处。

**原因**：`$LINENO` 在 trap 里取到的是 trap 内部的行号。

**修复**：trap 里用 `$BASH_LINENO[0]`，或者干脆在 trap 里打印 `$BASH_COMMAND`。

```bash
trap 'die "执行失败: $BASH_COMMAND (行 ${BASH_LINENO[0]})"' ERR
```

---

## 二、幂等与状态

### 2.1 密钥没全写进 `.env`，重跑就 unbound

**现象**：第一次跑成功，重跑时报

```
install.sh: line 375: SECRET_TURN: unbound variable
```

**原因**：`.env` 只存了数据库密码和 LiveKit 密钥，
`SECRET_TURN` / `SECRET_AS` / `SECRET_HS` 这些只存在于内存里。
重跑时从 `.env` 加载 → 缺项 → 模板替换时变量为空 → `set -u` 报错。

**修复**：所有密钥都必须落盘，并且加载后逐个检查补齐。

```bash
for v in SECRET_DB SECRET_REG SECRET_TURN SECRET_AS SECRET_HS LIVEKIT_KEY LIVEKIT_SECRET; do
  [[ -n ${!v:-} ]] && continue
  printf -v "$v" '%s' "$(openssl rand -hex 32)"
done
```

`${!v:-}` 是**间接引用**，配合 `:-` 才不会在 `set -u` 下炸。

### 2.2 DNS 记录「已存在」不一定是好事

**现象**：第二次部署时，脚本发现 `chat.域名` 已存在，
但指向的是另一台服务器。

**原因**：如果无脑 PATCH 覆盖，会把别人正在跑的服务顶掉。

**修复**：先比对 IP，相同才跳过；不同则停下来问用户，
非交互模式直接失败。

```bash
if [[ $old_ip == "$PUBLIC_IP" ]]; then ok "已存在且指向本机"; continue; fi
warn "$fqdn 指向 $old_ip（不是本机）"
confirm "强制覆盖？" n || die "已中止，请换前缀"
```

### 2.4 Postgres 数据目录残留旧密码

**现象**：Synapse 起不来，日志

```
psycopg2.OperationalError: FATAL: password authentication failed for user "synapse"
```

**原因**：`data/postgres` 是第一次初始化时的密码，之后 `.env` 里的密码变了
（比如重跑时补生成了缺失的密钥），两边对不上。`docker compose up` 不会
因为环境变量变了就重建已有数据目录。

**修复**

```bash
cd /opt/matrix
docker compose down
rm -rf data/postgres
# 重新运行脚本；它会用 .env 里的当前密码初始化
```

脚本在等待 Synapse 超时时会自动 grep 这个错误并打印提示。

### 2.3 `docker compose pull -q` 失败也要能看见日志

`-q` 会吞掉进度但保留错误，不过错误常常只有最后几行有用，
所以统一 `> /tmp/xxx.log 2>&1`，失败时 `tail -20`。

---

## 三、文件权限（macOS → Linux 必踩）

### 3.1 容器读不到挂载的配置文件

**现象**：`matrix-element-1` 一直 `Restarting (1)`，日志

```
cp: can't open '/app/config.json': Permission denied
```

**原因**：在 macOS 上打的 tar 包保留了 `uid=501` 和 `600` 权限，
解压到 Linux 后文件属主是 501，容器里的 nginx（uid 101）读不了。

**修复**

```bash
chown -R root:root element synapse livekit coturn
chmod -R a+rX element synapse livekit coturn
```

脚本里渲染完配置统一 `chmod 644`。

### 3.2 Synapse 的 `/data` 必须是 uid 991

**现象**：Synapse 反复重启，`PermissionError: '/data/homeserver.pid'`。

**原因**：`matrixdotorg/synapse` 镜像在 `run` 模式下用 `gosu 991:991`
降权运行，而且**只有 `generate` 模式才会 chown**。

**修复**

```bash
chown -R 991:991 /opt/matrix/data/synapse
```

---

## 四、证书

### 4.1 1Panel 申请的证书不落盘，站点绑不上

**现象**：1Panel 证书列表显示「正常」，但站点 → HTTPS → 证书下拉框
显示「无数据」。

**原因**：1Panel 2.5.0 通过 DNS 账号申请成功后，证书只在数据库里，
没有写到 `conf/ssl/` 目录，站点绑定列表因此是空的。

**绕过**：用 acme.sh 在宿主机重新签一张，直接写进 OpenResty 的 ssl 目录，
再手写站点配置。代价是 1Panel 界面显示 HTTP（状态不同步）。

### 4.2 通配符证书不覆盖三级子域

`*.example.com` **不覆盖** `livekit.im.example.com`（通配符只匹配一级）。
所以最初方案里 `livekit.im.` 必须单独列进 SAN。

**改进**：改成 3 个子域（`chat` / `chatapi` / `chatad`），
全是一级子域，一张证书全包，问题消失。

### 4.3 DNS-01 需要 Zone:Read

只给 `DNS:Edit` 权限时，`_get_root` 拿不到 zone id，签发失败。
Cloudflare Token 必须同时有：

- `Zone → DNS → Edit`
- `Zone → Zone → Read`

### 4.4 用 `--install-cert` 当幂等判断

```bash
if ~/.acme.sh/acme.sh --install-cert -d "$primary" --key-file ... ; then
  ok "复用已有证书"
else
  ~/.acme.sh/acme.sh --issue ...
fi
```

比解析 `--list` 的输出可靠，也不会因为重复签发触发 Let's Encrypt 的
每周 5 张同名证书限制。

---

## 五、反向代理

### 5.1 大文件上传被挡：两个参数缺一不可

```nginx
client_max_body_size 0;        # 不限体积
proxy_request_buffering off;   # 边收边转，不先写 /tmp
```

只加第一行，20 GB 的文件会先把 `/tmp` 写满；
两条都在 server 层，location 里也行。

### 5.2 LiveKit 的路径前缀必须 strip

```nginx
location /livekit/sfu/ {
    proxy_pass http://127.0.0.1:7880/;   # 结尾的 / 就是 strip
}
```

少写结尾斜杠，请求会变成 `http://127.0.0.1:7880/livekit/sfu/rtc`，
LiveKit 只认 `/rtc`，直接 404。

### 5.3 通话建立不起来：lk-jwt 去连 8448

**现象**：Element Call 打开通话界面，双方看不到彼此，LiveKit 无参与者。

**日志**

```
OpenIDUserInfo: Failed to look up user info
err=... https://chat.example.com:8448/_matrix/federation/v1/openid/userinfo
```

**原因**：lk-jwt 校验用户身份时默认拼 `server_name:8448`（联邦端口）。
只开了 443 就必然超时。

**修复**：在网页域名上补 federation 委派

```nginx
location = /.well-known/matrix/server {
    return 200 '{"m.server":"chatapi.example.com:443"}';
}
```

改完 `reload nginx` + 重启 lk-jwt。

### 5.4 证书安装早于 nginx 安装会中断整个流程

**现象**

```
Installing full chain to: /etc/nginx/ssl/matrix/fullchain.pem
Running reload cmd: systemctl reload nginx
Failed to reload nginx.service: Unit nginx.service not found.
✗ 证书安装失败
```

**原因**：`acme.sh --install-cert` 的 `--reloadcmd` 返回非 0，
acme.sh 认为安装失败，脚本 `set -e` 直接中止。
而此时 nginx 还没装（脚本原来的顺序是：签证书 → 装 nginx）。

**修复**（两处都要改）

1. 把「安装 nginx 包」提前到签证书之前，只装包不写站点；
2. `--reloadcmd` 加容错：`systemctl reload nginx 2>/dev/null || true`。

```bash
ensure_nginx     # 只 apt install，不碰站点配置
start_stack
issue_cert       # 此时 nginx 已存在，reload 不会失败
setup_nginx      # 写站点配置 + reload
```

### 5.5 1Panel 的 conf.d 会被面板覆盖

手写 `/opt/1panel/www/conf.d/*.conf` 后，只要在 1Panel 界面
改一次该站点的域名/HTTPS，配置就会被重新生成、自定义部分丢失。

脚本里检测到 1Panel 时应优先走它的 API，或者明确提示用户
「不要在该站点的面板界面里改设置」。

---

## 六、网络

### 6.1 MTU 黑洞：大包 100% 丢

**现象**：服务器本地 `curl` 毫秒级，客户端访问时快时慢，
手机端直接 `ERR_INTERNET_DISCONNECTED`。

**诊断**

```bash
ping -c 20 -D -s 1472 <ip>   # 100% 丢包
ping -c 20 -D -s 1200 <ip>   # 40% 丢包但能通
```

**修复**：把服务器网卡 MTU 降到 1300，并持久化。

```bash
ip link set dev eth0 mtu 1300
```

默认的 1460 字节分段会被中间设备静默丢弃，TCP 只能无限重传。

### 6.2 云厂商默认只开 SSH 端口（RackSmart 实测）

**现象**：脚本自检 6 项全部通过，但公网访问 Element 返回 **522**，
直连 `chatapi` 域名 TLS 握手失败（curl exit 35）。

**三步定位**

1. 先排除主机防火墙 —— ufw 规则明明是对的：

```bash
ufw status verbose
# 80/tcp  ALLOW IN  Anywhere
# 443/tcp ALLOW IN  Anywhere
```

2. 再抓包，看数据包到底有没有进来：

```bash
# 服务器上
tcpdump -ni ens7 -c 10 "tcp port 80 or tcp port 443"

# 本机同时发起访问
curl -m 4 http://<公网IP>/
```

结果是 **`0 packets captured`** —— 请求根本没到达网卡。

3. 结论：拦截在服务商的网络层，服务器内部改不了。

**RackSmart 的具体位置**

```
客户中心 → 产品管理 → VPS → 产品详情
  → 右侧标签「安全组」→ 管理 → 添加
```

默认策略只有这些入站规则，没有 80/443：

```
tcp 22      tcp 29038    tcp 3389    udp 68    icmp
```

需要补的 7 条：

| 方向 | 协议 | 端口 | 用途 |
| --- | --- | --- | --- |
| 进 | TCP | 80 | HTTP / ACME |
| 进 | TCP | 443 | HTTPS |
| 进 | TCP | 7881 | LiveKit TCP 兜底 |
| 进 | TCP | 3478 | TURN |
| 进 | UDP | 3478 | TURN |
| 进 | UDP | 49160-49200 | TURN 中继 |
| 进 | UDP | 50000-50100 | LiveKit 媒体 |

补完后 `nc -z <ip> 443` 立刻变 OPEN，页面可用。

> 台湾/香港这类 VPS 常见形态：网卡上是公网 IP、主机防火墙空、
> 但上层安全组只放行 SSH。判断口诀：
> **ufw 放行 + tcpdump 收不到包 = 服务商安全组**。

### 6.3 云安全组 ≠ 主机防火墙

`ufw status` 是 inactive 不代表端口通——云厂商的安全组在机房那层。
脚本只管 ufw，安全组必须在云控制台单独放行，所以脚本最后会打印端口清单。

---

## 七、容器

### 7.1 element-web 镜像会改写 config.json

`/docker-entrypoint.d/18-load-element-modules.sh` 会把 `/app/config.json`
复制到 `/tmp` 再写回。挂载成 `:ro` 且没有读权限时就会失败。

**教训**：只读挂载 + 权限 600 是双重雷，先确保 `a+r`。

### 7.2 LiveKit 必须 `use_external_ip`

云服务器的公网 IP 不在网卡上，`rtc.use_external_ip: true` 让 LiveKit
通过 STUN 发现对外地址，否则客户端拿到的 ICE 候选是内网 IP，连不上。

### 7.3 coturn 必须 host 网络

TURN 要直接对公网提供 UDP 并占用大范围中继端口，
在 bridge 网络里映射几十个端口不现实，所以 `network_mode: host`。

---

## 八、Cloudflare

### 8.1 橙云对请求体有硬上限

Free/Pro：100 MB，Business 200 MB，5 GB 要 Enterprise。
「无限上传」和「橙云代理」互斥，所以 API 域名必须走灰云。

### 8.2 代理只支持固定端口

80 / 8080 / 8880 / 2052 / 2082 / 2086 / 2095、443 / 2053 / 2083 / 2087 / 2096 / 8443。
自定义端口（如 9108）开橙云是连不上的。

### 8.3 源服务器证书只被 Cloudflare 信任

浏览器不认，灰云域名用它会直接证书错误。
要么全站用 Let's Encrypt，要么橙云 + 源证书。

---

## 九、测试方法

### 9.1 先 `--dry-run` 再真跑

```bash
bash install.sh --dry-run --skip-dns --non-interactive --domain example.com ...
```

只渲染配置不启动，渲染结果和线上工作的配置逐字 diff，
能提前发现 90% 的模板错误。

### 9.2 自检用 `--resolve` 绕过 DNS

```bash
curl --resolve chat.example.com:443:<ip> https://chat.example.com/
```

避免 DNS 缓存/Cloudflare 代理干扰，直接验证源站。

### 9.3 远端排查优先看三处日志

```bash
docker compose logs --tail=50 synapse
docker logs --since 3m matrix-lk-jwt-1
docker logs --since 3m matrix-livekit-1
```

通话问题看后两个；登录、联邦、媒体看第一个。

---

## 十、如果重来一次，我会这样做

1. 一开始就用 **3 个子域**（chat / chatapi / chatad），不要 5 个，
   证书和 nginx 都少一半。
2. 所有密钥第一版就写进 `.env`，别在内存里传来传去。
3. 模板独立成文件，用 `@占位符@` + Python 替换，不要用 shell heredoc。
4. 部署前先 `ping -s 1472` 测一下链路 MTU，省得后面查半天。
5. 证书直接用 acme.sh，别依赖面板的证书管理。
