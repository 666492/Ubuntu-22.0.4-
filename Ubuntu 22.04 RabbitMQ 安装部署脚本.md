# Ubuntu 22.04 RabbitMQ 安装部署脚本

> 本文档提供一个自动化 Shell 脚本，用于在 Ubuntu 22.04.4 系统上安装 RabbitMQ，开启 Web 管理后台，创建管理员用户，并配置防火墙。
---

## 一、脚本概述

本脚本将自动完成以下任务：
1. 检查 root 权限和系统版本。
2. 更新软件包列表。
3. 安装 RabbitMQ Server。
4. 启动服务并设置开机自启。
5. 开启 RabbitMQ 的 Web 管理插件。
6. 重启服务使插件生效。
7. 创建远程管理用户（用户名：admin，密码可自定义）。
8. 设置用户为管理员角色并授予所有权限。
9. 配置防火墙放行 5672（消息服务）和 15672（管理后台）端口。
10. 检查端口监听状态。
11. 输出访问地址和登录信息。

> **注意**：脚本中默认密码为 `RabbitMQ_2026@Admin!`，请根据实际需求修改。

---

## 二、完整脚本

将以下脚本保存为 `setup-rabbitmq.sh`，执行 `chmod +x setup-rabbitmq.sh` 赋予权限，然后以 root 身份运行：

```bash
#!/bin/bash

# ==========================================
# Ubuntu 22.04 RabbitMQ 安装部署脚本
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

echo "开始 RabbitMQ 安装与配置..."

# -------------------------------
# 3. 更新软件包列表
# -------------------------------
echo "更新软件包列表..."
apt update -y

# -------------------------------
# 4. 安装 RabbitMQ Server
# -------------------------------
echo "安装 RabbitMQ Server..."
apt install -y rabbitmq-server

# -------------------------------
# 5. 启动服务并设置开机自启
# -------------------------------
echo "启动 RabbitMQ 服务并设置开机自启..."
systemctl start rabbitmq-server
systemctl enable --now rabbitmq-server

# -------------------------------
# 6. 检查服务状态
# -------------------------------
echo "RabbitMQ 服务状态："
systemctl status rabbitmq-server --no-pager | head -n 10

# -------------------------------
# 7. 开启 Web 管理后台插件
# -------------------------------
echo "开启 RabbitMQ Web 管理插件..."
rabbitmq-plugins enable rabbitmq_management

# -------------------------------
# 8. 重启服务使插件生效
# -------------------------------
echo "重启 RabbitMQ 服务..."
systemctl restart rabbitmq-server

# -------------------------------
# 9. 创建远程管理用户
# -------------------------------
ADMIN_USER="admin"
ADMIN_PASSWORD="RabbitMQ_2026@Admin!"  # 请按需修改

echo "检查用户 $ADMIN_USER 是否已存在..."
if rabbitmqctl list_users | grep -q "^$ADMIN_USER[[:space:]]"; then
    echo "用户 $ADMIN_USER 已存在，跳过创建。"
else
    echo "创建用户 $ADMIN_USER..."
    rabbitmqctl add_user "$ADMIN_USER" "$ADMIN_PASSWORD"
fi

echo "设置用户 $ADMIN_USER 为管理员角色..."
rabbitmqctl set_user_tags "$ADMIN_USER" administrator

echo "授予用户 $ADMIN_USER 所有权限..."
rabbitmqctl set_permissions -p / "$ADMIN_USER" ".*" ".*" ".*"

# -------------------------------
# 10. 配置防火墙
# -------------------------------
echo "配置防火墙放行 5672 和 15672 端口..."
if command -v ufw >/dev/null 2>&1; then
    ufw allow 5672/tcp
    ufw allow 15672/tcp
    ufw reload
    ufw status
else
    echo "未检测到 ufw，请手动开放 5672 和 15672 端口。"
fi

# -------------------------------
# 11. 检查端口监听
# -------------------------------
echo "检查 RabbitMQ 相关端口监听状态："
ss -lntp | grep -E ':5672\s|:15672\s' || echo "警告：未检测到 5672 或 15672 端口监听。"

# -------------------------------
# 12. 输出完成信息
# -------------------------------
IP_ADDR=$(hostname -I | awk '{print $1}')
echo ""
echo "========================================="
echo "RabbitMQ 安装与配置完成！"
echo "管理后台地址: http://${IP_ADDR}:15672"
echo "消息服务端口: 5672"
echo "用户名: $ADMIN_USER"
echo "密码: $ADMIN_PASSWORD"
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
- 确保脚本以 root 身份运行，否则无法执行系统级别的安装和配置。

```bash
if ! grep -qi "ubuntu" /etc/os-release; then
    echo "错误：此脚本仅适用于 Ubuntu 系统。"
    exit 1
fi
```
- 检测系统是否为 Ubuntu，避免误操作。

### 2. 更新与安装
```bash
apt update -y
apt install -y rabbitmq-server
```
- 刷新软件源索引，然后安装 RabbitMQ Server 包。

### 3. 启动与开机自启
```bash
systemctl start rabbitmq-server
systemctl enable --now rabbitmq-server
```
- `start` 立即启动服务，`enable --now` 同时设置开机自启并确保服务当前正在运行。

### 4. 查看服务状态
```bash
systemctl status rabbitmq-server --no-pager | head -n 10
```
- 显示服务状态前 10 行，确认是否正常运行。

### 5. 开启 Web 管理插件
```bash
rabbitmq-plugins enable rabbitmq_management
```
- RabbitMQ 默认不带 Web 管理界面，需要通过此命令启用 `rabbitmq_management` 插件。
- 启用后需要重启服务才能生效，因此脚本中紧接着执行了 `systemctl restart rabbitmq-server`。

### 6. 创建管理员用户
```bash
if rabbitmqctl list_users | grep -q "^$ADMIN_USER[[:space:]]"; then
    echo "用户 $ADMIN_USER 已存在，跳过创建。"
else
    rabbitmqctl add_user "$ADMIN_USER" "$ADMIN_PASSWORD"
fi
```
- 使用 `rabbitmqctl list_users` 列出所有用户，通过 `grep` 检查目标用户是否已存在。
- 若不存在，则执行 `add_user` 创建用户。

```bash
rabbitmqctl set_user_tags "$ADMIN_USER" administrator
```
- `set_user_tags` 设置用户角色为 `administrator`，拥有管理后台的完整权限。

```bash
rabbitmqctl set_permissions -p / "$ADMIN_USER" ".*" ".*" ".*"
```
- 授予用户在默认虚拟主机 `/` 上的配置、写、读权限，`.*` 表示匹配所有资源。

### 7. 防火墙放行
```bash
ufw allow 5672/tcp
ufw allow 15672/tcp
ufw reload
```
- 开放消息服务端口 5672 和管理后台端口 15672，然后重载防火墙规则。

### 8. 检查端口监听
```bash
ss -lntp | grep -E ':5672\s|:15672\s'
```
- 使用 `ss` 查看端口监听情况，确保 RabbitMQ 正常监听这两个端口。

### 9. 输出访问信息
```bash
IP_ADDR=$(hostname -I | awk '{print $1}')
```
- 获取本机第一个 IP 地址，用于显示访问 URL。

---

## 四、使用方法

1. 将脚本保存为 `setup-rabbitmq.sh`。
2. 赋予执行权限：`chmod +x setup-rabbitmq.sh`。
3. 以 root 身份运行：`sudo ./setup-rabbitmq.sh`。
4. 等待脚本执行完毕，根据输出信息在浏览器访问 `http://本机IP:15672`，使用 admin 用户和密码登录。

---

## 五、常见问题与排错

- **服务启动失败**：检查 `/var/log/rabbitmq/` 下的日志文件，常见原因有端口冲突、Erlang 版本不匹配等。
- **无法访问管理后台**：确认防火墙已放行 15672，且 `rabbitmq_management` 插件已启用并重启过服务。
- **用户创建报错**：如果用户已存在，脚本会跳过创建，但后续 `set_user_tags` 和 `set_permissions` 仍会执行，确保用户权限正确。
- **ufw 未安装**：脚本会提示手动开放端口，可执行 `apt install ufw` 安装防火墙工具。

---

## 六、总结

本脚本自动化完成了 RabbitMQ 在 Ubuntu 上的安装、Web 管理后台开启、管理员用户创建和防火墙配置。
