# Ubuntu 22.04 Redis 安装部署脚本

> 本文档提供一个自动化 Shell 脚本，用于在 Ubuntu 22.04.4 系统上安装 Redis，配置远程访问和密码，并放行防火墙端口。

---

## 一、脚本概述

本脚本将自动完成以下任务：
1. 检查 root 权限和系统版本。
2. 更新软件包列表。
3. 安装 Redis Server。
4. 启动 Redis 服务并设置开机自启。
5. 修改 Redis 配置文件，允许远程访问并设置密码。
6. 重启 Redis 服务使配置生效。
7. 配置防火墙放行 6379 端口。
8. 验证本机连接（使用密码执行 PING 命令）。
9. 输出远程连接信息。

> **注意**：脚本中设置的 Redis 密码为 `Redis_2026@Admin!`，请根据实际需求修改。安装前请确保 6379 端口未被占用。

---

## 二、完整脚本

将以下脚本保存为 `setup-redis.sh`，执行 `chmod +x setup-redis.sh` 赋予权限，然后以 root 身份运行：

```bash
#!/bin/bash

# ==========================================
# Ubuntu 22.04 Redis 安装部署脚本
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

echo "开始 Redis 安装与配置..."

# -------------------------------
# 3. 更新软件包列表
# -------------------------------
echo "更新软件包列表..."
apt update -y

# -------------------------------
# 4. 安装 Redis Server
# -------------------------------
echo "安装 Redis Server..."
apt install -y redis-server

# -------------------------------
# 5. 查看 Redis 版本
# -------------------------------
echo "Redis 版本信息："
redis-server --version

# -------------------------------
# 6. 启动服务并设置开机自启
# -------------------------------
echo "启动 Redis 服务并设置开机自启..."
systemctl start redis-server
systemctl enable redis-server

# -------------------------------
# 7. 检查服务状态
# -------------------------------
echo "Redis 服务状态："
systemctl status redis-server --no-pager | head -n 10

# -------------------------------
# 8. 配置远程访问和密码
# -------------------------------
REDIS_CONF="/etc/redis/redis.conf"
REDIS_PASSWORD="Redis_2026@Admin!"  # 请按需修改

echo "备份原配置文件..."
cp "$REDIS_CONF" "${REDIS_CONF}.bak"

# --- 修改 bind 地址为 0.0.0.0 ---
echo "修改 bind 地址为 0.0.0.0..."
# 如果存在以 bind 开头的行，替换为 bind 0.0.0.0
if grep -q "^bind " "$REDIS_CONF"; then
    sed -i 's/^bind .*/bind 0.0.0.0/' "$REDIS_CONF"
else
    # 如果不存在，则在文件末尾添加
    echo "bind 0.0.0.0" >> "$REDIS_CONF"
fi

# --- 设置 requirepass 密码 ---
echo "设置 Redis 访问密码..."
# 检查是否存在被注释的 requirepass 行（# requirepass ...）
if grep -q "^# requirepass " "$REDIS_CONF"; then
    sed -i "s/^# requirepass .*/requirepass $REDIS_PASSWORD/" "$REDIS_CONF"
# 检查是否存在未注释的 requirepass 行
elif grep -q "^requirepass " "$REDIS_CONF"; then
    sed -i "s/^requirepass .*/requirepass $REDIS_PASSWORD/" "$REDIS_CONF"
else
    # 如果都不存在，则在文件末尾添加
    echo "requirepass $REDIS_PASSWORD" >> "$REDIS_CONF"
fi

echo "配置完成。protected-mode 保持 yes，密码已设置。"

# -------------------------------
# 9. 重启 Redis 服务
# -------------------------------
echo "重启 Redis 服务..."
systemctl restart redis-server

# -------------------------------
# 10. 配置防火墙
# -------------------------------
echo "配置防火墙放行 6379 端口..."
if command -v ufw >/dev/null 2>&1; then
    ufw allow 6379/tcp
    ufw reload
    ufw status
else
    echo "未检测到 ufw，请手动开放 6379 端口。"
fi

# -------------------------------
# 11. 检查端口监听
# -------------------------------
echo "检查 6379 端口监听状态："
ss -lntp | grep 6379 || echo "警告：6379 端口未监听，请检查 Redis 是否正常启动。"

# -------------------------------
# 12. 本机验证连接
# -------------------------------
echo "验证本机连接（使用密码执行 PING）..."
if redis-cli -a "$REDIS_PASSWORD" ping | grep -q PONG; then
    echo "本机连接验证成功：PONG"
else
    echo "警告：本机连接验证失败，请检查密码和配置。"
fi

# -------------------------------
# 13. 输出完成信息
# -------------------------------
IP_ADDR=$(hostname -I | awk '{print $1}')
echo ""
echo "========================================="
echo "Redis 安装与配置完成！"
echo "本机 IP: $IP_ADDR"
echo "端口: 6379"
echo "密码: $REDIS_PASSWORD"
echo "远程连接命令示例:"
echo "  redis-cli -h $IP_ADDR -p 6379 -a '$REDIS_PASSWORD'"
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
- 确保脚本以 root 身份运行，否则无法修改系统配置。

```bash
if ! grep -qi "ubuntu" /etc/os-release; then
    echo "错误：此脚本仅适用于 Ubuntu 系统。"
    exit 1
fi
```
- 检查系统是否为 Ubuntu，避免在错误系统上执行。

### 2. 安装 Redis
```bash
apt update -y
apt install -y redis-server
```
- 更新软件源索引并安装 Redis。

### 3. 启动与管理
```bash
systemctl start redis-server
systemctl enable redis-server
```
- 立即启动服务并设置开机自启。

### 4. 查看状态
```bash
systemctl status redis-server --no-pager | head -n 10
```
- 显示服务状态前 10 行，确认正常运行。

### 5. 修改配置文件
```bash
if grep -q "^bind " "$REDIS_CONF"; then
    sed -i 's/^bind .*/bind 0.0.0.0/' "$REDIS_CONF"
else
    echo "bind 0.0.0.0" >> "$REDIS_CONF"
fi
```
- 查找以 `bind` 开头的行，若存在则替换为 `bind 0.0.0.0`，不存在则追加到文件末尾。这样 Redis 就会监听所有网络接口，允许远程连接。

```bash
if grep -q "^# requirepass " "$REDIS_CONF"; then
    sed -i "s/^# requirepass .*/requirepass $REDIS_PASSWORD/" "$REDIS_CONF"
elif grep -q "^requirepass " "$REDIS_CONF"; then
    sed -i "s/^requirepass .*/requirepass $REDIS_PASSWORD/" "$REDIS_CONF"
else
    echo "requirepass $REDIS_PASSWORD" >> "$REDIS_CONF"
fi
```
- 检查是否存在被注释的 `# requirepass` 行，若有则取消注释并设置密码；若已有未注释的 `requirepass` 则直接替换；若都不存在则在文件末尾添加。这样确保密码被正确设置。

### 6. 重启服务
```bash
systemctl restart redis-server
```
- 重新加载配置，使修改生效。

### 7. 防火墙
```bash
ufw allow 6379/tcp
ufw reload
```
- 开放 Redis 默认端口，并重载规则。

### 8. 验证连接
```bash
if redis-cli -a "$REDIS_PASSWORD" ping | grep -q PONG; then
    echo "本机连接验证成功：PONG"
else
    echo "警告：本机连接验证失败，请检查密码和配置。"
fi
```
- 使用 `redis-cli` 带密码执行 `ping`，若返回 `PONG` 则说明密码正确、服务正常。

### 9. 输出信息
```bash
IP_ADDR=$(hostname -I | awk '{print $1}')
```
- 获取本机 IP 用于提示远程连接命令。

---

## 四、使用方法

1. 将脚本保存为 `setup-redis.sh`。
2. 赋予执行权限：`chmod +x setup-redis.sh`。
3. 以 root 身份运行：`sudo ./setup-redis.sh`。
4. 等待执行完毕，根据输出信息进行远程连接测试。

---

## 五、常见问题与排错

- **6379 端口被占用**：如果之前已安装 Redis 或其他服务占用 6379，请先停止旧服务或修改新 Redis 的端口。
- **连接被拒绝**：检查防火墙是否放行 6379，以及 Redis 是否已监听 `0.0.0.0:6379`。
- **密码错误**：确认配置文件中 `requirepass` 已正确设置，且重启服务后生效。
- **protected-mode 干扰**：由于我们保留了 `protected-mode yes`，但设置了密码，远程连接时需要密码才能访问，否则会被拒绝。

---

## 六、总结

本脚本自动化完成了 Redis 在 Ubuntu 上的安装、远程访问配置、密码设置和防火墙放行。
