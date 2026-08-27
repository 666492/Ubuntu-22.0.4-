# Ubuntu 22.04 Elasticsearch 安装部署脚本

> 本文档提供一个自动化 Shell 脚本，用于在 Ubuntu 22.04.4 系统上安装 Elasticsearch 8.14.3，完成基础配置、服务启动、防火墙放行及访问验证。
---

## 一、脚本概述

本脚本将自动完成以下任务：
1. 检查 root 权限和系统版本。
2. 安装基础依赖（wget、gnupg 等）。
3. 下载 Elasticsearch 8.14.3 的 deb 安装包。
4. 使用 `dpkg` 安装 Elasticsearch。
5. 在配置文件中追加单节点模式、允许外部访问等配置。
6. 启动服务并设置开机自启。
7. 配置防火墙放行 9200 端口。
8. 验证本机 HTTPS 访问（使用默认 elastic 用户和密码）。
9. 输出远程访问地址及登录信息。

> **注意**：Elasticsearch 8 默认启用 HTTPS 和账号认证，安装时会自动生成 `elastic` 用户的密码。本脚本使用用户预设的密码（`RO7W*f8PgEAwBZ1vHabW`），若实际密码不同，请修改脚本中的变量。

---

## 二、完整脚本

将以下脚本保存为 `setup-elasticsearch.sh`，执行 `chmod +x setup-elasticsearch.sh` 赋予权限，然后以 root 身份运行：

```bash
#!/bin/bash

# ==========================================
# Ubuntu 22.04 Elasticsearch 8.14.3 安装部署脚本
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

echo "开始 Elasticsearch 安装与配置..."

# -------------------------------
# 3. 定义变量
# -------------------------------
ES_VERSION="8.14.3"
ES_DEB="elasticsearch-${ES_VERSION}-amd64.deb"
ES_DOWNLOAD_URL="https://artifacts.elastic.co/downloads/elasticsearch/${ES_DEB}"
ES_CONF="/etc/elasticsearch/elasticsearch.yml"
ES_PASSWORD="RO7W*f8PgEAwBZ1vHabW"   # 请根据实际安装时生成的密码修改

# -------------------------------
# 4. 安装基础依赖
# -------------------------------
echo "安装基础依赖..."
apt update -y
apt install -y wget gnupg ca-certificates apt-transport-https

# -------------------------------
# 5. 下载 Elasticsearch 安装包
# -------------------------------
if [ -f "$ES_DEB" ]; then
    echo "检测到安装包 $ES_DEB 已存在，跳过下载。"
else
    echo "下载 Elasticsearch ${ES_VERSION} 安装包..."
    wget "$ES_DOWNLOAD_URL"
fi

# -------------------------------
# 6. 安装 Elasticsearch
# -------------------------------
echo "安装 Elasticsearch..."
dpkg -i "$ES_DEB"

# 安装过程中可能因为依赖问题报错，尝试修复
if [ $? -ne 0 ]; then
    echo "检测到依赖问题，尝试自动修复..."
    apt install -f -y
    dpkg -i "$ES_DEB"
fi

# -------------------------------
# 7. 修改 Elasticsearch 配置
# -------------------------------
echo "备份原配置文件..."
cp "$ES_CONF" "${ES_CONF}.bak"

echo "追加自定义配置到 elasticsearch.yml..."
# 使用 cat 追加配置，注意去掉用户示例中的多余 # 符号
cat >> "$ES_CONF" <<EOF

# ===== 自定义配置 =====
cluster.name: my-es
node.name: node-1
network.host: 0.0.0.0
http.port: 9200
discovery.type: single-node
EOF

# -------------------------------
# 8. 启动 Elasticsearch 服务
# -------------------------------
echo "重新加载 systemd 配置..."
systemctl daemon-reload

echo "启动 Elasticsearch 服务..."
systemctl start elasticsearch.service

echo "设置开机自启..."
systemctl enable elasticsearch.service

echo "检查服务状态..."
systemctl status elasticsearch.service --no-pager | head -n 15

# -------------------------------
# 9. 配置防火墙
# -------------------------------
echo "配置防火墙放行 9200 端口..."
if command -v ufw >/dev/null 2>&1; then
    ufw allow 9200/tcp
    ufw reload
    ufw status
else
    echo "未检测到 ufw，请手动开放 9200 端口。"
fi

# -------------------------------
# 10. 验证访问
# -------------------------------
echo "验证本机 HTTPS 访问..."
# 使用 wget 忽略证书验证并携带账号密码
wget --no-check-certificate --user=elastic --password="$ES_PASSWORD" -qO- https://localhost:9200 && echo "" || echo "本机访问失败，请检查服务和密码。"

# -------------------------------
# 11. 输出完成信息
# -------------------------------
IP_ADDR=$(hostname -I | awk '{print $1}')
echo ""
echo "========================================="
echo "Elasticsearch 安装与配置完成！"
echo "访问地址: https://${IP_ADDR}:9200"
echo "用户名: elastic"
echo "密码: $ES_PASSWORD"
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
- 确保以 root 身份运行，否则无法安装软件和修改系统配置。

```bash
if ! grep -qi "ubuntu" /etc/os-release; then
    echo "错误：此脚本仅适用于 Ubuntu 系统。"
    exit 1
fi
```
- 检查系统是否为 Ubuntu。

### 2. 变量定义
```bash
ES_VERSION="8.14.3"
ES_DEB="elasticsearch-${ES_VERSION}-amd64.deb"
ES_DOWNLOAD_URL="https://artifacts.elastic.co/downloads/elasticsearch/${ES_DEB}"
ES_CONF="/etc/elasticsearch/elasticsearch.yml"
ES_PASSWORD="RO7W*f8PgEAwBZ1vHabW"
```
- 集中定义版本、下载地址、配置文件和密码，便于后续修改。

### 3. 安装依赖
```bash
apt update -y
apt install -y wget gnupg ca-certificates apt-transport-https
```
- 确保系统具有下载和传输所需的基本工具。

### 4. 下载安装包
```bash
if [ -f "$ES_DEB" ]; then
    echo "检测到安装包 $ES_DEB 已存在，跳过下载。"
else
    wget "$ES_DOWNLOAD_URL"
fi
```
- 检查当前目录是否已有该文件，避免重复下载。

### 5. 安装与依赖修复
```bash
dpkg -i "$ES_DEB"
if [ $? -ne 0 ]; then
    apt install -f -y
    dpkg -i "$ES_DEB"
fi
```
- `dpkg -i` 安装 deb 包。如果因为缺少依赖而失败，则使用 `apt install -f -y` 自动修复依赖，然后再次安装。

### 6. 配置修改
```bash
cat >> "$ES_CONF" <<EOF
# ===== 自定义配置 =====
cluster.name: my-es
node.name: node-1
network.host: 0.0.0.0
http.port: 9200
discovery.type: single-node
EOF
```
- 将自定义配置追加到 Elasticsearch 配置文件末尾，不会覆盖原有内容。
- `network.host: 0.0.0.0` 允许外部访问；`discovery.type: single-node` 指定为单节点模式（适合测试环境）。

### 7. 启动服务
```bash
systemctl daemon-reload
systemctl start elasticsearch.service
systemctl enable elasticsearch.service
```
- 重新加载 systemd 配置，启动服务并设置开机自启。

### 8. 防火墙
```bash
ufw allow 9200/tcp
ufw reload
```
- 开放 9200 端口供外部访问。

### 9. 验证
```bash
wget --no-check-certificate --user=elastic --password="$ES_PASSWORD" -qO- https://localhost:9200
```
- 使用 `wget` 忽略证书校验（因为自签名证书），携带用户名密码请求本机 9200 端口，输出 JSON 即代表成功。

### 10. 输出信息
```bash
IP_ADDR=$(hostname -I | awk '{print $1}')
```
- 获取本机 IP 用于提示访问地址。

---

## 四、使用方法

1. 将脚本保存为 `setup-elasticsearch.sh`。
2. 赋予执行权限：`chmod +x setup-elasticsearch.sh`。
3. 以 root 身份运行：`sudo ./setup-elasticsearch.sh`。
4. 等待脚本执行完毕，记录输出的密码（如果与实际生成的不同，请手动修改脚本中的 `ES_PASSWORD`）。
5. 在浏览器访问 `https://本机IP:9200`，输入用户名 `elastic` 和密码。

---

## 五、常见问题与排错

- **下载失败**：检查网络是否能访问 `artifacts.elastic.co`，可尝试手动下载或更换镜像。
- **安装依赖错误**：脚本已包含自动修复依赖的步骤，通常可以解决。
- **服务启动失败**：查看日志 `journalctl -u elasticsearch.service` 或 `/var/log/elasticsearch/` 下的日志文件，常见原因有内存不足（默认需要 4GB 堆内存）、配置格式错误等。
- **忘记密码**：可执行 `/usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic` 重置。
- **防火墙未放行**：确认 9200 端口已开放，且 Elasticsearch 监听在 `0.0.0.0:9200`。

---

## 六、总结

本脚本自动化完成了 Elasticsearch 的下载、安装、基础配置和验证流程。
