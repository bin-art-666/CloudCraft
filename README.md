# CloudCraft · 从零搭建个人云端开发工作站

> 一台 2 核 2 GB 的云服务器，搭配浏览器、code-server、Caddy 和自己的域名，打造随处可访问的开发环境，并在同一台服务器上托管个人主页。

**关键词**：Ubuntu 24.04 / 阿里云 ECS / SSH / Node.js / code-server / systemd / Caddy / DNS / HTTPS / ICP 备案

**阶段进展**：个人主页 `https://wuweibi.xin` 已在服务器端得到 `HTTP/2 200` 响应；`code.wuweibi.xin` 既有服务进程为 `active`，新配置加载后仍建议再次进行浏览器端验证。

> 安全提示：本文的域名、IP 为真实项目的公开服务地址，重现时请替换；**不要提交** SSH 私钥、code-server 密码、令牌、`.env`、DNS API 密钥、证书私钥等。以下安装命令是复现参考，不表示所有步骤都在同一天执行。

## 1. 为什么要做这个项目？

目标是让 Windows PC、平板甚至手机，只要拥有现代浏览器，就能进入同一套开发环境：项目文件始终保存在服务器，终端、编辑器与扩展运行在云端，不必在每台设备上重新搭环境。

最终还希望搭建自己的技术博客与个人主页，让服务器承担两个角色：**私有开发工作站**和**公开的静态网站**，以不同域名区分服务。

## 2. 架构与选型

```mermaid
flowchart TB
    Client[Windows / 平板 / 手机浏览器] --> DNS[域名 DNS 解析]
    DNS --> HTTPS[Caddy :443 · HTTPS]
    HTTPS -->|code.wuweibi.xin| IDE[code-server · 127.0.0.1:8080]
    HTTPS -->|wuweibi.xin| Web[静态个人主页 · /var/www/home]
    IDE --> Files[/home/dev/projects]
```

| 项目 | 配置 / 职责 |
|---|---|
| 服务器 | 阿里云 ECS，2 vCPU / 2 GB RAM |
| 系统 | Ubuntu 24.04 LTS |
| 运行时 | Node.js 24 系列（本次环境中曾使用 v24.11.1） |
| 编辑器 | code-server（本次环境曾使用 4.143.1） |
| 进程管理 | systemd，服务名 `code-server-dev` |
| Web 服务 | Caddy，自动 HTTPS / 反向代理 / 静态文件 |
| 开发目录 | `/home/dev/projects` |
| 网站目录 | `/var/www/home` |
| 域名 | `code.wuweibi.xin` / `wuweibi.xin` |

### 关键设计决策

- code-server 仅监听 `127.0.0.1:8080`，**不对公网开放 8080**。
- 由 Caddy 统一接收公网 `80/443` 请求，通过域名分流。
- code-server 以普通用户 `dev` 运行；管理员仅负责部署与维护。
- 个人主页是独立静态站点，不破坏已有的开发服务。
- 修改 Caddy 前先备份、验证，再热加载。

## 3. 从零准备 ECS

创建 Ubuntu 24.04 ECS，建议配置 SSH 密钥认证（或配置强密码并逐步迁移到密钥）、防火墙和最小权限安全组。对公网仅按需开放 TCP `22`、`80`、`443`；SSH 最好限制可信来源 IP；**不要将 TCP 8080 对公网放开**。

登录服务器（将示例 IP 和用户名替换成自己的）：

```bash
ssh <your-user>@<your-ecs-public-ip>
```

初始化：

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl ca-certificates git
```

## 4. 安装 Node.js 与 code-server

本项目使用了 Node.js 24 环境，通过 npm 安装 code-server。使用稳定版 Node.js 并确认 code-server 官方兼容性；以下命令假设 `node` / `npm` 已正确安装：

```bash
node -v
npm -v
sudo npm install -g code-server
code-server --version
```

如果使用自定义路径的 Node.js（项目中曾使用 `/opt/nodejs/...`），必须确保 systemd 服务能找到正确的可执行文件。可通过 `command -v code-server` 找到实际路径。

建立开发用户与工作目录（若已存在则跳过）：

```bash
sudo adduser dev
sudo mkdir -p /home/dev/projects
sudo chown -R dev:dev /home/dev/projects
```

以 `dev` 用户配置 `~/.config/code-server/config.yaml`（不要将真实密码上传 GitHub）：

```yaml
bind-addr: 127.0.0.1:8080
auth: password
password: REPLACE_WITH_A_LONG_RANDOM_SECRET
cert: false
```

文件仅允许该用户读取：

```bash
sudo chmod 600 /home/dev/.config/code-server/config.yaml
```

可以使用 code-server 自己的密码散列配置方式（`hashed-password`）减少明文密码暴露，并按官方文档验证配置。不要将上方占位密码直接用于公网服务。

### systemd 自启动

项目实际使用 `code-server-dev` 服务名。下面是参考单元，`ExecStart` 应按 `command -v code-server` 得到的路径修改：

```ini
# /etc/systemd/system/code-server-dev.service
[Unit]
Description=code-server for dev
After=network.target

[Service]
Type=simple
User=dev
WorkingDirectory=/home/dev/projects
ExecStart=/usr/local/bin/code-server /home/dev/projects
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now code-server-dev
systemctl status code-server-dev --no-pager
ss -lntp | grep 8080
```

预期监听地址是 `127.0.0.1:8080`，而不是 `0.0.0.0:8080`。在尚未配置域名之前，可以使用临时 SSH 隧道在本地浏览器测试：

```bash
ssh -N -L 8080:127.0.0.1:8080 <your-user>@<your-ecs-public-ip>
```

然后打开 `http://127.0.0.1:8080`。这一步无需开放 8080 公网端口。

## 5. 域名与 DNS

在域名解析控制台设置 A 记录：

| 主机记录 | 类型 | 指向 | 作用 |
|---|---|---|---|
| `code` | A | ECS 公网 IPv4 | 浏览器 IDE |
| `@` | A | ECS 公网 IPv4 | 个人主页 |

本项目的公开域名为 `code.wuweibi.xin` 和 `wuweibi.xin`，都指向同一台 ECS。由 Caddy 按 Host 区分服务，不需要两台服务器。

验证：

```bash
getent ahostsv4 wuweibi.xin
getent ahostsv4 code.wuweibi.xin
```

**踩坑：代理软件 Fake-IP。** 在 Windows 的代理/TUN 环境中，即使执行 `nslookup domain 223.5.5.5`，也可能收到 `198.18.x.x`，这通常是代理生成的虚拟地址，并不代表权威 DNS 配错。曾在服务器端通过 `getent ahostsv4` 查到正确公网 IP。也可对比独立网络或可信 DNS-over-HTTPS 查询。排障时要明确区分「系统解析」与「公开 DNS 记录」。

## 6. 用 Caddy 配置 HTTPS 反向代理

服务器安装并启动 Caddy 后，编辑 `/etc/caddy/Caddyfile`。本次最终使用两个独立站点：

```caddyfile
# 公开个人主页
wuweibi.xin {
    encode gzip
    root * /var/www/home
    file_server
}

# 私人开发环境
code.wuweibi.xin {
    encode gzip
    reverse_proxy 127.0.0.1:8080
}
```

Caddy 根据域名自动处理 HTTPS 证书和 HTTP→HTTPS 跳转（前提是解析正确，80/443 通畅且符合证书签发条件）。

**重要：保留旧配置，不盲目覆盖。** 操作次序：

```bash
sudo cp /etc/caddy/Caddyfile /etc/caddy/Caddyfile.bak
sudo caddy validate --config /etc/caddy/Caddyfile
sudo caddy fmt --overwrite /etc/caddy/Caddyfile  # 可选：规范格式
sudo systemctl reload caddy
sudo systemctl is-active caddy
sudo systemctl is-active code-server-dev
```

`Caddyfile input is not formatted` 是格式告警；`Valid configuration` 才表示语法验证通过。若验证失败，不执行 reload，先修复配置。

安全提醒：code-server 的密码登录只能作为基本保护。对于长期公网访问的开发工作站，更建议在 Caddy 之前或网络层增加 VPN / Zero Trust 身份访问控制、MFA、访问审计与定期更新，保护源码与终端权限。

## 7. 发布独立个人主页

```bash
sudo mkdir -p /var/www/home
sudo tee /var/www/home/index.html >/dev/null <<'HTML'
<!doctype html>
<html lang="zh-CN">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <title>个人主页</title>
</head>
<body>
  <h1>你好，世界！</h1>
  <p>这里记录技术学习与项目实践。</p>
  <footer><a href="https://beian.miit.gov.cn/">请替换成真实ICP备案号</a></footer>
</body>
</html>
HTML
sudo chmod -R a+rX /var/www/home
```

请在正式对外运行时把备案占位内容替换为真实、完整的ICP备案编号；公安联网备案按当地适用要求办理并展示。

此时 Caddy 直接读取 `/var/www/home/index.html`，原先的 code-server 仍运行在 `127.0.0.1:8080`，互不干扰。

## 8. 验证结果

```bash
curl -I https://wuweibi.xin
curl -I https://code.wuweibi.xin
systemctl is-active caddy code-server-dev
```

本次部署过程中的真实检查输出（节选）：

```text
$ caddy validate --config /etc/caddy/Caddyfile
Valid configuration

$ curl -I https://wuweibi.xin
HTTP/2 200
content-type: text/html; charset=utf-8
server: Caddy
```

这证明**服务器端的个人主页 HTTPS 请求已正常返回 HTML**。开发服务的 systemd 状态此前为 `active`；对于新增站点热加载之后的 code-server 浏览器访问，还应再做一次独立验收，不能只凭主站返回 200 认定全部完成。

## 9. 实际遇到的问题

| 问题 | 原因 / 处理 |
|---|---|
| Windows `nslookup` 结果是 `198.18.x.x` | 代理 Fake-IP；从 ECS 端独立核查解析 |
| code-server 要远程访问却不想暴露 8080 | 仅监听 loopback，Caddy 转发 443 |
| 服务器重启后进程可能丢失 | 使用 systemd 并 `enable` |
| Caddy 验证时有 `WARN` | 区分格式警告和实际语法错误 |
| 需要加个人主页又怕影响 IDE | 备份配置、独立站点块、validate 后 reload |
| 2 GB 内存偏紧 | 控制扩展数量，监控资源；慎装大规模开发依赖 |
| 中国内地服务器公开托管网站 | 按适用要求完成 ICP、公安联网备案与页面展示 |

## 10. 日常运维备忘

```bash
# 进程状态
systemctl status caddy code-server-dev --no-pager

# 服务日志
journalctl -u caddy -n 50 --no-pager
journalctl -u code-server-dev -n 50 --no-pager

# 端口核查
ss -lntp | grep -E ':(22|80|443|8080)\b'

# CPU / 内存 / 磁盘
free -h
df -h
ps -eo pid,ppid,%mem,%cpu,args --sort=-%mem | head -20
```

运维要点：定期更新系统与 code-server；保留可恢复的项目备份（备份应加密并隔离）；SSH 优先密钥登录，禁用不必要的 root 远程口令登录；每次改 Caddy 都应 validate；如需修改公开域名或用途，及时核实备案信息要求。

## 11. 下一步规划

- [x] 阿里云 ECS + Ubuntu 24.04
- [x] code-server 以 systemd 管理
- [x] Caddy 提供域名 HTTPS 入口
- [x] 个人主页通过 `HTTP/2 200` 验证
- [ ] 再次验证 code-server 浏览器端登录
- [ ] 完善站点备案号及公安联网备案展示
- [ ] 部署 Hugo / Astro 等静态博客
- [ ] 配置 Git 自动部署、版本化备份和监控
- [ ] 为远程 IDE 增加更强的访问控制

## 12. 结语

这个项目最重要的收获，不只是“在浏览器里打开 VS Code”，而是完整串联了 **Linux 服务器运维、端口监听、反向代理、DNS、TLS、systemd、安全组、ICP备案与静态网站托管**。把每个组件的职责弄清楚后，一台普通的低配云服务器就能成为持续迭代的个人开发基础设施。

---

记录日期：2026-11-01。本文是实践复盘与复现教程；版本和平台规则未来可能变化，请以官方文档为准。