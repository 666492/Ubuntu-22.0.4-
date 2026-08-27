# Ubuntu 22.04 Nginx 安装部署脚本

> 本文档提供一个自动化 Shell 脚本，用于在 Ubuntu 22.04.4 系统上安装 Nginx，并处理可能出现的软件源问题。

---

## 一、脚本概述

本脚本将自动完成以下任务：
1. 检查是否以 root 用户执行。
2. 检测系统是否为 Ubuntu。
3. 更新软件包列表。
4. 尝试安装 Nginx，若失败则自动切换到清华镜像源并重新安装。
5. 检查 Nginx 版本。
6. 启动 Nginx 服务并设置开机自启。
7. 配置防火墙永久放行 80 和 443 端口。
8. 检查端口监听状态。
9. 输出访问地址。

> **说明**：脚本包含软件源自动修复逻辑，当默认源安装失败时，会备份原有 `sources.list` 并替换为清华大学镜像源。

---

## 二、完整脚本

将以下脚本保存为 `setup-nginx.sh`，执行 `chmod +x setup-nginx.sh` 赋予权限，然后以 root 身份运行：

```bash
#!/bin/bash

# ==========================================
# Ubuntu 22.04 Nginx 安装部署脚本
# ==========================================

set -e  # 遇到错误立即退出

# -------------------------------
# 1. 检查是否以 root 用户运行
# -------------------------------
if [ "$EUID" -ne 0 ]; then
    echo "错误：请使用 root 用户或 sudo 运行此脚本。"
    exit 1
fi

# -------------------------------
# 2. 检测系统是否为 Ubuntu
# -------------------------------
if ! grep -qi "ubuntu" /etc/os-release; then
    echo "错误：此脚本仅适用于 Ubuntu 系统。"
    exit 1
fi

echo "开始 Nginx 安装与配置..."

# -------------------------------
# 3. 更新软件包列表
# -------------------------------
echo "更新软件包列表..."
apt update -y

# -------------------------------
# 4. 尝试安装 Nginx
# -------------------------------
echo "尝试安装 Nginx..."
if apt install -y nginx; then
    echo "Nginx 安装成功。"
else
    echo "安装失败，正在尝试更换软件源为清华大学镜像..."

    # 备份原 sources.list
    cp /etc/apt/sources.list /etc/apt/sources.list.bak

    # 写入清华源
    cat > /etc/apt/sources.list <<EOF
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-security main restricted universe multiverse
EOF

    echo "清理缓存并更新..."
    apt clean
    apt update -y

    echo "重新安装 Nginx..."
    apt install -y nginx
    echo "Nginx 安装成功。"
fi

# -------------------------------
# 5. 检查 Nginx 版本
# -------------------------------
echo "Nginx 版本信息："
nginx -v

# -------------------------------
# 6. 启动 Nginx 并设置开机自启
# -------------------------------
echo "启动 Nginx 服务并设置开机自启..."
systemctl start nginx
systemctl enable nginx

# -------------------------------
# 7. 检查服务状态
# -------------------------------
echo "Nginx 服务状态："
systemctl status nginx --no-pager | head -n 10

# -------------------------------
# 8. 配置防火墙
# -------------------------------
echo "配置防火墙放行 80 和 443 端口..."
if command -v ufw >/dev/null 2>&1; then
    ufw allow 80/tcp
    ufw allow 443/tcp
    ufw reload
    ufw status
else
    echo "未检测到 ufw，请手动开放 80 和 443 端口。"
fi

# -------------------------------
# 9. 检查端口监听
# -------------------------------
echo "检查端口监听状态："
ss -lntp | grep -E ':80\s|:443\s' || echo "警告：未检测到 80 或 443 端口监听。"

# -------------------------------
# 10. 获取本机 IP 并输出信息
# -------------------------------
IP_ADDR=$(hostname -I | awk '{print $1}')
echo ""
echo "========================================="
echo "Nginx 安装与配置完成！"
echo "访问地址: http://${IP_ADDR}"
echo "配置 HTTPS 后可使用: https://${IP_ADDR}"
echo "========================================="
```

---

## 三、脚本逐段解释

### 1. 权限与系统检测
```bash
if [ "$EUID" -ne 0 ]; then
    echo "错误：请使用 root 用户或 sudo 运行此脚本。"
    exit 1
fi
```
- 确保脚本以 root 身份运行，否则没有权限修改系统文件。

```bash
if ! grep -qi "ubuntu" /etc/os-release; then
    echo "错误：此脚本仅适用于 Ubuntu 系统。"
    exit 1
fi
```
- 检查系统是否为 Ubuntu，避免在错误系统上执行。

### 2. 更新软件包列表
```bash
apt update -y
```
- 刷新本地软件源索引，让系统知道哪些包可用。

### 3. 尝试安装 Nginx 并处理失败
```bash
if apt install -y nginx; then
    echo "Nginx 安装成功。"
else
    # ... 备份、替换源、重试
fi
```
- 首先尝试使用当前源安装，如果失败（可能因为源过期或不可用），则进入错误处理流程。

### 4. 替换为清华源
```bash
cp /etc/apt/sources.list /etc/apt/sources.list.bak
cat > /etc/apt/sources.list <<EOF
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy main restricted universe multiverse
...
EOF
```
- 备份原文件后，使用 `cat` 和 here-doc 将新的源写入 `sources.list`。
- `jammy` 对应 Ubuntu 22.04 的代号。

### 5. 清理并更新
```bash
apt clean
apt update -y
```
- `apt clean` 清空已下载的包缓存，避免残留旧信息。
- 更新索引以使用新源。

### 6. 重新安装
```bash
apt install -y nginx
```
- 再次安装，此时应该能成功。

### 7. 启动服务
```bash
systemctl start nginx
systemctl enable nginx
```
- 立即启动并设置开机自启。

### 8. 防火墙
```bash
ufw allow 80/tcp
ufw allow 443/tcp
ufw reload
```
- 开放 HTTP 和 HTTPS 端口，并重载规则。

### 9. 端口检查
```bash
ss -lntp | grep -E ':80\s|:443\s'
```
- 查看 80 和 443 端口是否被监听。若输出包含 `0.0.0.0:80` 表示正常。

### 10. 输出信息
```bash
IP_ADDR=$(hostname -I | awk '{print $1}')
```
- 获取本机第一个 IP 地址用于提示。

---

## 四、使用方法

1. 将脚本保存为 `setup-nginx.sh`。
2. 赋予执行权限：`chmod +x setup-nginx.sh`。
3. 以 root 身份运行：`sudo ./setup-nginx.sh`。
4. 脚本执行完毕后，在浏览器访问 `http://本机IP` 应看到 Nginx 欢迎页。

---

## 五、常见问题与排错

- **安装失败且脚本未能修复**：检查网络是否能访问清华源，或手动编辑 `/etc/apt/sources.list` 后重新运行脚本。
- **80 端口被占用**：使用 `ss -lntp | grep :80` 查看占用进程，停止该服务或修改 Nginx 监听端口。
- **ufw 未安装**：脚本会提示手动开放端口，可执行 `apt install ufw` 安装，或使用其他防火墙工具。
- **访问不到页面**：确认防火墙已放行，且 Nginx 服务正在运行。

---

## 六、总结

本脚本通过自动处理软件源问题，提高了 Nginx 安装的成功率，并完成了基础的服务管理和防火墙配置。
