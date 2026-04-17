# 1panel-proxy-complete-guide
1Panel-挂载代理完整指南

适用场景：腾讯云 / 阿里云等国内服务器，需要访问 Telegram、Docker Hub、Brave Search 等境外服务。

---

## 环境要求

- Ubuntu 20.04 / 22.04
- 已安装 1Panel
- 有一个可用的机场订阅链接
- 服务器架构：x86_64（amd64）

---

## 第一步：安装 Mihomo（Clash Meta）

```bash
# 查询最新版本下载链接
wget -qO- https://api.github.com/repos/MetaCubeX/mihomo/releases/latest | grep "browser_download_url.*linux-amd64-v1\." 

# 下载标准版（替换为上面查到的实际版本号）
wget https://github.com/MetaCubeX/mihomo/releases/download/v1.19.21/mihomo-linux-amd64-v1.19.21.gz

# 解压、移动、赋权
gunzip mihomo-linux-amd64-v1.19.21.gz
sudo mv mihomo-linux-amd64-v1.19.21 /usr/local/bin/clash
sudo chmod +x /usr/local/bin/clash

# 验证安装
clash -v
```

---

## 第二步：准备配置文件

```bash
sudo mkdir -p /etc/clash
```

**方式 A：直接下载订阅（如果服务器能访问）**

```bash
sudo wget -O /etc/clash/config.yaml "你的订阅链接"
```

**方式 B：本地下载后上传（推荐，机场一般屏蔽服务器IP）**

1. 在本地电脑的 Clash Verge / Clash for Windows 中找到对应订阅
2. 右键 → 在文件夹中显示，找到 `.yaml` 配置文件
3. 通过 **1Panel → 文件管理** 上传到 `/etc/clash/config.yaml`

**确认配置文件正确：**

```bash
head -5 /etc/clash/config.yaml
# 正确的文件开头应包含 port、allow-lan、dns 等字段
```

**确保配置文件中开启了 allow-lan：**

```yaml
allow-lan: true
```

---

## 第三步：下载 MMDB 地理数据库

Clash 启动需要这个文件，服务器无法自动下载时手动处理：

在本地浏览器访问以下链接下载：

```
https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/country.mmdb
```

通过 **1Panel → 文件管理** 上传到 `/etc/clash/country.mmdb`

---

## 第四步：配置 systemd 开机自启

```bash
sudo tee /etc/systemd/system/clash.service > /dev/null << 'EOF'
[Unit]
Description=Clash Proxy
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/clash -d /etc/clash
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable clash
sudo systemctl start clash

# 确认运行状态
sudo systemctl status clash
```

正常启动后日志应包含：

```
Mixed(http+socks) proxy listening at: [::]:7890
RESTful API listening at: 127.0.0.1:9090
```

---

## 第五步：给 Docker 容器配置代理

查询 Docker 网关 IP（通常是 `172.17.0.1`）：

```bash
ip route | grep docker
```

编辑 docker-compose.yml（以 OpenClaw 为例）：

```bash
sudo nano /opt/1panel/apps/openclaw/OpenClaw/docker-compose.yml
```

在 `services.openclaw` 下添加 environment 块：

```yaml
services:
    openclaw:
        environment:
            - HTTP_PROXY=http://172.17.0.1:7890
            - HTTPS_PROXY=http://172.17.0.1:7890
            - NO_PROXY=localhost,127.0.0.1
```

> 注意 YAML 缩进，environment 与 image、ports 等字段平级

重启容器：

```bash
cd /opt/1panel/apps/openclaw/OpenClaw
sudo docker compose down && sudo docker compose up -d
```

---

## 第六步：给 Docker 守护进程配置代理（用于拉取镜像）

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo tee /etc/systemd/system/docker.service.d/proxy.conf > /dev/null << 'EOF'
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:7890"
Environment="HTTPS_PROXY=http://127.0.0.1:7890"
Environment="NO_PROXY=localhost,127.0.0.1"
EOF

sudo systemctl daemon-reload
sudo systemctl restart docker
```

---

## 验证

**验证容器内网络是否通畅：**

```bash
sudo docker exec -it <容器名> sh -c "curl -s https://api.telegram.org"
# 返回 302 或 JSON 即为成功
```

**验证 Docker 拉取镜像是否正常：**

```bash
sudo docker pull hello-world
```

---

## 常见问题

| 问题 | 原因 | 解决方法 |
|------|------|----------|
| Clash 启动失败：`can't download MMDB` | 无法自动下载地理数据库 | 手动上传 country.mmdb |
| 订阅链接 403 | 机场屏蔽服务器 IP | 本地下载后上传 |
| Docker 拉取超时 | Docker 守护进程未配置代理 | 执行第六步 |
| 容器内超时但宿主机正常 | 代理 IP 写成了 127.0.0.1 | 改为 Docker 网关 IP（172.17.0.1） |
| Clash 启动后端口未监听 | 配置文件格式错误 | 检查 config.yaml 开头字段 |

---

## 目录结构

```
/etc/clash/
├── config.yaml      # 机场订阅配置
└── country.mmdb     # 地理数据库

/etc/systemd/system/
├── clash.service                    # Clash 服务
└── docker.service.d/proxy.conf      # Docker 代理配置
```

---

## 相关链接

- [Mihomo 发布页](https://github.com/MetaCubeX/mihomo/releases)
- [meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat/releases)
