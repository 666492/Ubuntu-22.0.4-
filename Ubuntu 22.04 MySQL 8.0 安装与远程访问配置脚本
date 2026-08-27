# Ubuntu 22.04 MySQL 8.0 安装与远程访问配置脚本

> 本文档提供一个完整的 Shell 脚本，用于在 Ubuntu 22.04.4 系统上安装 MySQL 8.0，完成基本安全配置，并开放远程访问权限。

---

## 一、脚本概述

本脚本将自动完成以下任务：
1. 检查是否以 root 用户执行。
2. 检测系统是否为 Ubuntu。
3. 安装 MySQL Server（如果尚未安装）。
4. 启动 MySQL 服务并设置开机自启。
5. 检查 MySQL 版本和服务状态。
6. 配置 root 用户密码（本地认证）。
7. 创建远程访问用户并授予权限。
8. 修改 MySQL 监听地址为 `0.0.0.0`。
9. 重启 MySQL 服务使配置生效。
10. 配置防火墙开放 3306 端口。
11. 验证端口监听状态。
12. 输出远程连接提示信息。

> **注意**：脚本中涉及密码设置，请根据实际需求修改密码强度。MySQL 8.0 默认启用密码策略，简单密码可能导致设置失败，建议使用包含大小写字母、数字和特殊字符的复杂密码。

---

## 二、完整脚本内容

将以下脚本保存为 `setup-mysql.sh`，并使用 `chmod +x setup-mysql.sh` 赋予执行权限，然后以 root 身份运行：`sudo ./setup-mysql.sh`。

```bash
#!/bin/bash

# ==========================================
# Ubuntu 22.04 MySQL 8.0 安装与远程访问配置脚本
# ==========================================

set -e  # 遇到错误立即退出

# -------------------------------
# 1. 检查是否以 root 用户执行
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

echo "开始 MySQL 8.0 安装与配置..."

# -------------------------------
# 3. 更新软件包列表
# -------------------------------
echo "更新软件包列表..."
apt update -y

# -------------------------------
# 4. 安装 MySQL Server
# -------------------------------
if ! dpkg -l | grep -q mysql-server; then
    echo "安装 MySQL Server..."
    apt install -y mysql-server
else
    echo "MySQL Server 已安装，跳过安装步骤。"
fi

# -------------------------------
# 5. 启动 MySQL 服务并设置开机自启
# -------------------------------
echo "启动 MySQL 服务并设置开机自启..."
systemctl start mysql
systemctl enable mysql

# -------------------------------
# 6. 检查 MySQL 服务状态
# -------------------------------
echo "检查 MySQL 服务状态..."
systemctl status mysql --no-pager | head -n 10

# -------------------------------
# 7. 显示 MySQL 版本
# -------------------------------
echo "MySQL 版本信息："
mysql --version

# -------------------------------
# 8. 配置 root 用户密码（本地认证）
# -------------------------------
# Ubuntu 默认 root 用户通过 auth_socket 认证，无需密码。
# 这里改为使用密码认证，并设置一个初始密码。
# 请将 'Root_2026@Local!' 替换为你要设置的密码（需符合密码策略）
ROOT_LOCAL_PASSWORD="Root_2026@Local!"

echo "设置 root 用户本地密码..."
mysql -u root <<EOF
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '${ROOT_LOCAL_PASSWORD}';
FLUSH PRIVILEGES;
EOF

# -------------------------------
# 9. 创建远程访问用户并授权
# -------------------------------
# 创建一个允许从任意主机连接的 root 用户（也可创建其他用户名）
# 请将 'Mysql_2026@Root!' 替换为远程用户的密码（需符合密码策略）
REMOTE_USER="root"
REMOTE_PASSWORD="Mysql_2026@Root!"

echo "创建远程访问用户 '${REMOTE_USER}'@'%' 并授予所有权限..."
mysql -u root -p"${ROOT_LOCAL_PASSWORD}" <<EOF
CREATE USER IF NOT EXISTS '${REMOTE_USER}'@'%' IDENTIFIED BY '${REMOTE_PASSWORD}';
GRANT ALL PRIVILEGES ON *.* TO '${REMOTE_USER}'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
EOF

# -------------------------------
# 10. 修改 MySQL 监听地址为 0.0.0.0
# -------------------------------
# 默认 MySQL 只监听 127.0.0.1，需要修改配置文件允许远程连接
MYSQL_CONF="/etc/mysql/mysql.conf.d/mysqld.cnf"
if [ -f "$MYSQL_CONF" ]; then
    echo "修改 MySQL 监听地址..."
    # 备份原配置文件
    cp "$MYSQL_CONF" "${MYSQL_CONF}.bak"
    # 使用 sed 替换 bind-address
    sed -i 's/^bind-address\s*=.*/bind-address = 0.0.0.0/' "$MYSQL_CONF"
    # 如果找不到 bind-address 行，则在 [mysqld] 段后添加
    if ! grep -q "^bind-address" "$MYSQL_CONF"; then
        sed -i '/\[mysqld\]/a bind-address = 0.0.0.0' "$MYSQL_CONF"
    fi
    echo "配置文件已更新，原文件备份为 ${MYSQL_CONF}.bak"
else
    echo "警告：未找到 MySQL 配置文件 $MYSQL_CONF，请手动修改。"
fi

# -------------------------------
# 11. 重启 MySQL 服务使配置生效
# -------------------------------
echo "重启 MySQL 服务..."
systemctl restart mysql

# -------------------------------
# 12. 配置防火墙开放 3306 端口
# -------------------------------
echo "开放防火墙 3306 端口..."
if command -v ufw >/dev/null 2>&1; then
    ufw allow 3306/tcp
    ufw reload
    ufw status
else
    echo "未检测到 ufw，如果使用其他防火墙请手动开放 3306 端口。"
fi

# -------------------------------
# 13. 验证端口监听状态
# -------------------------------
echo "检查 MySQL 端口监听状态："
ss -lntp | grep 3306 || echo "警告：端口 3306 未监听，请检查配置。"

# -------------------------------
# 14. 输出完成信息
# -------------------------------
echo ""
echo "========================================="
echo "MySQL 安装与远程访问配置完成！"
echo "本地 root 密码: ${ROOT_LOCAL_PASSWORD}"
echo "远程 root 密码: ${REMOTE_PASSWORD}"
echo "远程连接地址: 本机 IP:3306"
echo "请使用 Navicat 或其他客户端测试连接。"
echo "========================================="
```

---

## 三、脚本逐段解释

### 1. 检查 root 权限
```bash
if [ "$EUID" -ne 0 ]; then
    echo "错误：请使用 root 用户或 sudo 运行此脚本。"
    exit 1
fi
```
- `$EUID` 是当前用户的 UID，root 的 UID 为 0。如果不是 root，则退出。

### 2. 检查系统是否为 Ubuntu
```bash
if ! grep -qi "ubuntu" /etc/os-release; then
    echo "错误：此脚本仅适用于 Ubuntu 系统。"
    exit 1
fi
```
- `/etc/os-release` 文件包含系统信息，`grep -qi` 忽略大小写查找 "ubuntu"。

### 3. 更新软件包列表
```bash
apt update -y
```
- 更新本地软件包索引，确保安装的是最新版本。

### 4. 安装 MySQL Server
```bash
if ! dpkg -l | grep -q mysql-server; then
    apt install -y mysql-server
else
    echo "MySQL Server 已安装，跳过安装步骤。"
fi
```
- `dpkg -l` 列出已安装的软件包，`grep -q mysql-server` 检查是否已安装。
- 如果未安装，则使用 `apt install -y` 自动安装。

### 5. 启动并设置开机自启
```bash
systemctl start mysql
systemctl enable mysql
```
- `start` 立即启动服务，`enable` 设置开机自启。

### 6. 查看服务状态
```bash
systemctl status mysql --no-pager | head -n 10
```
- 显示 MySQL 服务状态的前 10 行，`--no-pager` 避免分页阻塞。

### 7. 显示版本
```bash
mysql --version
```
- 输出 MySQL 客户端版本，可用于确认安装成功。

### 8. 配置 root 本地密码
```bash
mysql -u root <<EOF
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '${ROOT_LOCAL_PASSWORD}';
FLUSH PRIVILEGES;
EOF
```
- Ubuntu 默认 root 用户使用 `auth_socket` 认证，无需密码即可通过 `sudo mysql` 登录。这里将认证方式改为 `mysql_native_password` 并设置密码。
- `<<EOF ... EOF` 是 here-doc 语法，将多行 SQL 命令传递给 mysql。

### 9. 创建远程访问用户
```bash
mysql -u root -p"${ROOT_LOCAL_PASSWORD}" <<EOF
CREATE USER IF NOT EXISTS '${REMOTE_USER}'@'%' IDENTIFIED BY '${REMOTE_PASSWORD}';
GRANT ALL PRIVILEGES ON *.* TO '${REMOTE_USER}'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
EOF
```
- 创建允许从任意主机（`%`）连接的 root 用户，并授予所有权限。
- `-p"${ROOT_LOCAL_PASSWORD}"` 直接在命令行提供密码（注意：密码可能会出现在 shell 历史中，生产环境建议用其他方式）。

### 10. 修改监听地址
```bash
sed -i 's/^bind-address\s*=.*/bind-address = 0.0.0.0/' "$MYSQL_CONF"
```
- `sed -i` 直接替换配置文件中的 `bind-address` 行。
- 如果文件中不存在该行，则在 `[mysqld]` 段后插入。

### 11. 重启服务
```bash
systemctl restart mysql
```
- 重启使配置生效。

### 12. 配置防火墙
```bash
ufw allow 3306/tcp
ufw reload
ufw status
```
- 开放 3306 TCP 端口，重载防火墙规则，并显示状态。

### 13. 检查监听
```bash
ss -lntp | grep 3306
```
- 如果显示 `0.0.0.0:3306` 表示 MySQL 正在监听所有网络接口。

---

## 四、使用方法

1. 将脚本保存为 `setup-mysql.sh`。
2. 赋予执行权限：`chmod +x setup-mysql.sh`。
3. 以 root 身份执行：`sudo ./setup-mysql.sh`。
4. 根据输出提示，记录密码信息。
5. 使用 Navicat 或其他 MySQL 客户端测试连接。

---

## 五、注意事项

- **密码策略**：MySQL 8.0 默认启用 `validate_password` 组件，要求密码至少 8 位，包含大小写字母、数字和特殊字符。请使用强密码，否则脚本可能执行失败。
- **安全警告**：脚本中直接使用 root 用户远程访问存在安全风险，建议创建专用的远程用户并限制来源 IP。
- **配置文件路径**：如果 MySQL 配置文件路径不同，请手动修改脚本中的 `MYSQL_CONF` 变量。
- **防火墙工具**：本脚本使用 `ufw`，如果系统未安装，请自行使用 `iptables` 或云安全组开放端口。

---

## 六、总结

通过这个脚本，可以快速在 Ubuntu 系统上完成 MySQL 的安装、基本配置和远程访问设置。
