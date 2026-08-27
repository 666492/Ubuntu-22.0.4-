# Ubuntu 22.04 Tomcat 安装部署与配置脚本

> 本文档提供一个完整的 Shell 脚本，用于在 Ubuntu 22.04.4 系统上自动完成 JDK 安装、环境变量配置、Tomcat 9 安装、管理后台账号设置以及防火墙放行。
---

## 一、脚本概述

脚本将自动完成以下步骤：
1. 检查 root 权限和系统版本。
2. 更新软件包列表。
3. 安装 OpenJDK 11。
4. 配置 `JAVA_HOME` 环境变量（写入 `/etc/profile`）。
5. 安装 Tomcat 9 及管理组件。
6. 启动 Tomcat 服务并设置开机自启。
7. 配置防火墙开放 8080 端口。
8. 检查端口监听状态。
9. 配置 Tomcat 管理后台用户（用户名：admin，密码可自定义）。
10. 重启 Tomcat 使配置生效。
11. 输出访问地址和登录信息。

> **注意**：脚本中设置的 Tomcat 管理后台密码为 `Tomcat_2026@Admin!`，请根据实际需求修改。

---

## 二、完整脚本

将以下脚本保存为 `setup-tomcat.sh`，执行 `chmod +x setup-tomcat.sh` 赋予权限，然后以 root 身份运行：

```bash
#!/bin/bash

# ==========================================
# Ubuntu 22.04 Tomcat 9 安装部署脚本
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

echo "开始 Tomcat 安装与配置..."

# -------------------------------
# 3. 更新软件包列表
# -------------------------------
echo "更新软件包列表..."
apt update -y

# -------------------------------
# 4. 安装 OpenJDK 11
# -------------------------------
echo "安装 OpenJDK 11..."
apt install -y openjdk-11-jdk

# -------------------------------
# 5. 配置 JAVA_HOME 环境变量
# -------------------------------
echo "配置 JAVA_HOME 环境变量..."

# 获取 java 可执行文件的真实路径，例如 /usr/lib/jvm/java-11-openjdk-amd64/bin/java
JAVA_BIN=$(readlink -f "$(which java)")
# 获取 JAVA_HOME 路径（去掉末尾的 /bin/java）
JAVA_HOME=$(dirname "$(dirname "$JAVA_BIN")")

# 检查 /etc/profile 中是否已经设置了 JAVA_HOME
if grep -q "export JAVA_HOME=" /etc/profile; then
    echo "JAVA_HOME 已存在于 /etc/profile 中，跳过写入。"
else
    echo "写入 JAVA_HOME 到 /etc/profile..."
    # 备份原文件
    cp /etc/profile /etc/profile.bak
    # 追加环境变量配置
    cat >> /etc/profile <<EOF

# Java 环境变量
export JAVA_HOME=${JAVA_HOME}
export PATH=\$JAVA_HOME/bin:\$PATH
EOF
    echo "已添加 JAVA_HOME 配置。"
fi

# 立即生效（在当前 shell 中）
source /etc/profile

# 验证 JAVA_HOME
echo "当前 JAVA_HOME = $JAVA_HOME"
java -version

# -------------------------------
# 6. 安装 Tomcat 9 及管理组件
# -------------------------------
echo "安装 Tomcat 9 和 tomcat9-admin..."
apt install -y tomcat9 tomcat9-admin

# -------------------------------
# 7. 启动 Tomcat 并设置开机自启
# -------------------------------
echo "启动 Tomcat 服务并设置开机自启..."
systemctl start tomcat9
systemctl enable tomcat9

# -------------------------------
# 8. 检查 Tomcat 服务状态
# -------------------------------
echo "Tomcat 服务状态："
systemctl status tomcat9 --no-pager | head -n 10

# -------------------------------
# 9. 配置防火墙放行 8080 端口
# -------------------------------
echo "配置防火墙开放 8080 端口..."
if command -v ufw >/dev/null 2>&1; then
    ufw allow 8080/tcp
    ufw reload
    ufw status
else
    echo "未检测到 ufw，请手动开放 8080 端口。"
fi

# -------------------------------
# 10. 检查端口监听状态
# -------------------------------
echo "检查 8080 端口监听状态："
ss -ntpl | grep 8080 || echo "警告：8080 端口未监听，请检查 Tomcat 是否正常启动。"

# -------------------------------
# 11. 配置 Tomcat 管理后台账号
# -------------------------------
echo "配置 Tomcat 管理后台用户..."

TOMCAT_USERS_FILE="/etc/tomcat9/tomcat-users.xml"
ADMIN_USER="admin"
ADMIN_PASSWORD="Tomcat_2026@Admin!"  # 请按需修改

# 备份原文件
cp "$TOMCAT_USERS_FILE" "${TOMCAT_USERS_FILE}.bak"

# 检查是否已存在相同用户名
if grep -q "username=\"$ADMIN_USER\"" "$TOMCAT_USERS_FILE"; then
    echo "用户 $ADMIN_USER 已存在，跳过添加。"
else
    # 在 </tomcat-users> 前插入 role 和 user 定义
    sed -i "/<\/tomcat-users>/i \
  <role rolename=\"manager-gui\"/>\
  <role rolename=\"admin-gui\"/>\
  <user username=\"$ADMIN_USER\" password=\"$ADMIN_PASSWORD\" roles=\"manager-gui,admin-gui\"/>" "$TOMCAT_USERS_FILE"
    echo "已添加管理员用户 $ADMIN_USER。"
fi

# -------------------------------
# 12. 重启 Tomcat 使配置生效
# -------------------------------
echo "重启 Tomcat 服务..."
systemctl restart tomcat9

# -------------------------------
# 13. 输出完成信息
# -------------------------------
IP_ADDR=$(hostname -I | awk '{print $1}')
echo ""
echo "========================================="
echo "Tomcat 安装与配置完成！"
echo "访问地址: http://${IP_ADDR}:8080"
echo "管理后台账号: $ADMIN_USER"
echo "管理后台密码: $ADMIN_PASSWORD"
echo "========================================="
```

---

## 三、脚本逐段解释

### 1. 脚本头部和权限检查
```bash
#!/bin/bash
set -e
if [ "$EUID" -ne 0 ]; then
    echo "错误：请使用 root 用户或 sudo 运行此脚本。"
    exit 1
fi
```
- `set -e`：当任何命令返回非零退出码时立即终止脚本，避免错误累积。
- `$EUID` 是当前用户的有效 UID，root 为 0，非 root 则退出。

### 2. 检测 Ubuntu 系统
```bash
if ! grep -qi "ubuntu" /etc/os-release; then
    echo "错误：此脚本仅适用于 Ubuntu 系统。"
    exit 1
fi
```
- `/etc/os-release` 包含系统标识，`grep -qi` 忽略大小写查找 "ubuntu"。

### 3. 更新软件包列表
```bash
apt update -y
```
- 刷新本地软件源索引，确保后续安装最新可用版本。

### 4. 安装 OpenJDK 11
```bash
apt install -y openjdk-11-jdk
```
- 安装 Java 开发工具包，Tomcat 需要 Java 环境才能运行。
- `-y` 自动确认安装。

### 5. 配置 JAVA_HOME
```bash
JAVA_BIN=$(readlink -f "$(which java)")
JAVA_HOME=$(dirname "$(dirname "$JAVA_BIN")")
```
- `which java` 找到 java 命令位置。
- `readlink -f` 解析符号链接，得到实际路径（例如 `/usr/lib/jvm/java-11-openjdk-amd64/bin/java`）。
- 连续两次 `dirname` 去掉 `/bin/java`，得到 `JAVA_HOME` 目录。

写入 `/etc/profile` 使用 `cat >> /etc/profile` 追加，注意 `\$JAVA_HOME` 中的反斜杠是为了在 profile 中保留 `$` 符号，避免当前 shell 展开。

### 6. 安装 Tomcat
```bash
apt install -y tomcat9 tomcat9-admin
```
- `tomcat9` 是主程序，`tomcat9-admin` 提供管理后台所需的额外组件。

### 7. 启动服务
```bash
systemctl start tomcat9
systemctl enable tomcat9
```
- `start` 立即启动，`enable` 设置开机自启。

### 8. 检查服务状态
```bash
systemctl status tomcat9 --no-pager | head -n 10
```
- 显示前 10 行状态信息，确认服务是否运行正常。

### 9. 防火墙配置
```bash
if command -v ufw >/dev/null 2>&1; then
    ufw allow 8080/tcp
    ufw reload
    ufw status
fi
```
- 检查系统是否安装 `ufw`，若存在则开放 8080 端口并重载规则。

### 10. 检查监听
```bash
ss -ntpl | grep 8080
```
- 显示监听 8080 端口的进程，若输出 `0.0.0.0:8080` 表示外部可访问。

### 11. 配置管理后台用户
```bash
sed -i "/<\/tomcat-users>/i \
  <role rolename=\"manager-gui\"/>\
  <role rolename=\"admin-gui\"/>\
  <user username=\"$ADMIN_USER\" password=\"$ADMIN_PASSWORD\" roles=\"manager-gui,admin-gui\"/>" "$TOMCAT_USERS_FILE"
```
- `sed` 在包含 `</tomcat-users>` 的行之前插入三行 XML 内容。
- 插入的 role 和 user 用于访问 Tomcat 的 Manager 和 Host Manager 界面。
- 先使用 `grep` 检查用户是否已存在，避免重复插入。

### 12. 重启 Tomcat
```bash
systemctl restart tomcat9
```
- 重新加载配置，使 `tomcat-users.xml` 生效。

### 13. 输出信息
```bash
IP_ADDR=$(hostname -I | awk '{print $1}')
```
- 获取本机第一个 IP 地址，用于提示访问 URL。

---

## 四、使用说明

1. 将脚本保存为 `setup-tomcat.sh`。
2. 赋予执行权限：`chmod +x setup-tomcat.sh`。
3. 以 root 身份运行：`sudo ./setup-tomcat.sh`。
4. 等待脚本执行完毕，根据输出访问 `http://本机IP:8080`。
5. 访问管理后台 `http://本机IP:8080/manager/html`，使用配置的用户名和密码登录。

> **安全提示**：生产环境中请勿使用简单密码，并建议限制管理后台的访问来源 IP。

---

## 五、常见问题与排错

- **8080 端口被占用**：使用 `ss -ntpl | grep 8080` 查看占用进程，可停止旧服务或修改 Tomcat 监听端口（位于 `/etc/tomcat9/server.xml`）。
- **JAVA_HOME 不生效**：重新登录终端或执行 `source /etc/profile`。
- **管理后台无法访问**：确保 `tomcat9-admin` 已安装，且 `tomcat-users.xml` 中配置正确，重启 Tomcat 后重试。

---

## 六、总结

本脚本自动化完成了 Tomcat 在 Ubuntu 上的安装和基础配置，涵盖了从 JDK 安装到管理后台用户设置的全过程。
