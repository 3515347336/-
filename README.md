# （key not match一定要看！！！）RustDesk 自建服务器完整教程（WSL/云服务器 + 快速配置法）

> 本教程适用于 Windows 环境，支持本地部署（WSL）和云服务器部署，包含常规配置和快速配置两种方法

## 📋 目录
- [0. 准备工作](#0-准备工作)
- [1. 配置与部署](#1-配置与部署)
- [2. 配置 RustDesk 客户端](#2-配置-rustdesk-客户端)
- [3. 常见问题解决](#3-常见问题解决，key not match直接看这个就好)

---

## 0. 准备工作

### 环境要求
- **本地部署**：Ubuntu 22.04 + Docker Desktop + 非公网/校园网环境
- **云服务器部署**：任意云服务商（腾讯云、阿里云等）+ Ubuntu 系统

### 说明
- 如果没有公网 IP 或需要从外网访问，建议购买云服务器
- 本地部署仅适合局域网内使用（如家庭、公司内网）

---

## 1. 配置与部署

### 1.1 Linux 系统的配置

> ⚠️ **如果你使用云服务器，可以直接跳到 [1.2 节](#12-hbbs-与-hbbr-的配置)**

#### 1.1.1 安装 WSL

打开 PowerShell（管理员权限），检查是否已安装 WSL：

```powershell
wsl --version
```

如果显示版本信息则已安装，否则运行：

```powershell
wsl --install
```

#### 1.1.2 安装 Docker Desktop

1. 前往 [Docker 官网](https://www.docker.com/products/docker-desktop/) 下载 Docker Desktop
2. 安装过程一路默认设置即可
3. 安装完成后验证：

```powershell
docker --version
```

#### 1.1.3 安装 Ubuntu 22.04

1. 打开 Microsoft Store
2. 搜索 "Ubuntu 22.04 LTS"
3. 点击安装

#### 1.1.4 配置 Docker 与 WSL 集成

1. 启动 Docker Desktop
2. 点击右上角 **⚙️ 设置图标**
3. 选择 **Resources** → **WSL Integration**
4. 勾选 **Enable integration with my default WSL distro**
5. 勾选你刚安装的 **Ubuntu 22.04**
6. 点击 **Apply & Restart**

---

### 1.2 hbbs 与 hbbr 的配置

#### 1.2.1 创建项目目录

启动 Ubuntu 终端（本地）或 SSH 连接云服务器，然后运行：

```bash
mkdir -p ~/rustdesk-server && cd ~/rustdesk-server
```

#### 1.2.2 编写 Docker Compose 配置文件

```bash
nano docker-compose.yml
```

将以下内容复制进去：

```yaml
version: '3'

services:
  hbbs:
    image: rustdesk/rustdesk-server:latest
    container_name: hbbs
    command: hbbs -r <YOUR_IP>:21117
    ports:
      - "21115:21115"
      - "21116:21116"
      - "21116:21116/udp"
    volumes:
      - ./data:/root
    restart: unless-stopped
    networks:
      - rustdesk-net

  hbbr:
    image: rustdesk/rustdesk-server:latest
    container_name: hbbr
    command: hbbr
    ports:
      - "21117:21117"
      - "21118:21118"
      - "21119:21119"
    volumes:
      - ./data:/root
    restart: unless-stopped
    networks:
      - rustdesk-net

networks:
  rustdesk-net:
    driver: bridge
```

**重要配置说明：**

- `<YOUR_IP>` 替换为：
  - **云服务器**：填写你的公网 IP（从云服务商控制台获取）
  - **本地部署（局域网）**：运行 `ipconfig`（Windows）或 `ip addr`（Linux），找到本机局域网 IP（如 192.168.1.100）

**保存文件：**
- 按 `Ctrl + O` 保存
- 按 `Enter` 确认
- 按 `Ctrl + X` 退出

#### 1.2.3 启动服务

```bash
docker-compose up -d
```

#### 1.2.4 检查服务状态

```bash
docker ps
```

如果看到 `hbbs` 和 `hbbr` 两个容器都在运行（状态为 `Up`），说明部署成功！

#### 1.2.5 获取 Public Key

**方法 1：从公钥文件读取（推荐）**
```bash
docker exec hbbs cat /root/id_ed25519.pub
```

**方法 2：从日志获取**
```bash
docker logs hbbs 2>&1 | grep "Public Key"
```

**⚠️ 重要：保存完整的 Key，不要截取！**

示例 Key：
```
AbCdEf1234567890XyZwxyz+ABCDEFG1234567890==
```

---

### 1.3 开放必要端口

#### 云服务器（以腾讯云为例）

1. 登录云服务商控制台
2. 找到 **安全组** 或 **防火墙** 设置
3. 添加以下入站规则：

| 端口 | 协议 | 说明 |
|------|------|------|
| 21115 | TCP | hbbs |
| 21116 | TCP/UDP | hbbs |
| 21117 | TCP | hbbr (relay) |
| 21118 | TCP | hbbr |
| 21119 | TCP | hbbr |

> 💡 **提示**：阿里云叫"安全组规则"，华为云叫"网络 ACL"，其他云服务商类似

#### 本地部署（路由器端口转发）

如果需要从外网访问本地服务器：

1. 浏览器输入路由器管理地址（通常是 `192.168.1.1` 或 `192.168.0.1`）
2. 登录路由器后台
3. 找到 **端口转发** / **虚拟服务器** / **NAT 设置**（不同品牌名称不同）
4. 添加上述 5 个端口的转发规则，指向运行 RustDesk 服务器的内网 IP

---

## 2. 配置 RustDesk 客户端

### 2.1 常规配置方法

#### 步骤 1：准备信息
- **服务器 IP**：`<YOUR_IP>`（前面配置的 IP）
- **Public Key**：完整的 Key（前面获取的）

#### 步骤 2：配置客户端

1. 打开 RustDesk
2. 点击右上角 **三条横线图标** → **设置**
3. 选择 **网络** 标签
4. 点击 **ID/中继服务器**
5. 填写配置：
   - **ID 服务器**：`<YOUR_IP>`
   - **中继服务器**：`<YOUR_IP>`
   - **Key**：粘贴完整的 Public Key
6. 点击 **确定**
7. **重启 RustDesk**

#### 步骤 3：验证连接

重启后，RustDesk 右下角应显示：
- ✅ **就绪**（绿色） - 说明连接成功
- ❌ **正在连接...** - 说明配置有问题

---

### 2.2 快速配置法（强烈推荐！）

> 💡 **适用场景**：批量部署、避免手动配置、解决 Key 不匹配问题

#### 原理
RustDesk 会自动解析可执行文件名中的服务器配置，无需手动在软件中设置。

#### 步骤

**1. 清除旧配置（如果之前配置过）**

- 按 `Win + R`，输入 `%AppData%\RustDesk`
- 删除该文件夹内所有文件
- 卸载旧版本 RustDesk（如果有）

**2. 下载并重命名 RustDesk**

从 [RustDesk 官网](https://rustdesk.com/) 下载最新版本，将文件名改为：

```
rustdesk-host=<YOUR_IP>,key=<COMPLETE_KEY>.exe
```

**示例：**
```
rustdesk-host=140.143.234.241,key=AbCdEf1234567890XyZwxyz+ABCDEFG1234567890==.exe
```

**⚠️ 注意事项：**
- 文件名中不要有空格
- Key 必须是完整的（不要截取）
- 确保 `.exe` 后缀保留

**3. 分发并运行**

- 将改名后的 RustDesk 发给需要连接的电脑
- 对方直接运行即可，**无需任何手动配置**
- 双方都使用同一个改名版本

**4. 关键要点**

✅ **DO（正确做法）：**
- 使用改名版 RustDesk
- 不要在软件设置中手动填写任何服务器信息
- 保持文件名不变

❌ **DON'T（错误做法）：**
- 改名后又在软件里手动配置（会冲突）
- 修改文件名
- 截取 Key 的部分字符

---

## 3. 常见问题解决

### 问题 1：Key 不匹配

**症状：** 连接时提示 "Key mismatch" 或 "密钥不匹配"

**原因：**
- 软件内手动配置与文件名配置冲突
- 使用了错误或截取的 Key
- 客户端缓存了旧配置

**解决方案：**

**方法 A：使用快速配置法**
1. 按 `Win + R` → 输入 `%AppData%\RustDesk` → 删除所有文件
2. 检查并删除：`C:\Windows\ServiceProfiles\LocalService\AppData\Roaming\RustDesk\`
3. 卸载 RustDesk
4. 使用 [2.2 节](#22-快速配置法强烈推荐)的改名方法重新配置

**方法 B：完全清除后重新手动配置**
1. 清除配置文件（同方法 A）
2. 卸载 RustDesk
3. 重启电脑
4. 重新安装并按 [2.1 节](#21-常规配置方法)配置
5. 确保使用**完整的** Public Key

---

### 问题 2：远程电脑离线

**症状：** 输入对方 ID 后显示 "离线" 或 "Offline"

**排查步骤：**

**1. 确认对方配置正确**
- 对方也需要配置相同的服务器地址和 Key
- 检查对方 RustDesk 右下角是否显示 "就绪"

**2. 检查服务器状态**
```bash
docker ps | grep rustdesk
```
确保 hbbs 和 hbbr 都在运行

**3. 查看日志排查问题**
```bash
# 查看 hbbs 日志
docker logs --tail 50 hbbs

# 查看 hbbr 日志
docker logs --tail 50 hbbr
```

**4. 确认防火墙/安全组配置**
- 云服务器：检查安全组规则
- 本地部署：检查路由器端口转发、Windows 防火墙

**5. 测试端口连通性**
```bash
# 在客户端电脑上测试（Windows PowerShell）
Test-NetConnection -ComputerName <YOUR_IP> -Port 21116
```

---

### 问题 3：连接速度慢或卡顿

**原因：**
- 中继服务器带宽不足
- 网络延迟高

**解决方案：**
1. 选择离用户更近的云服务器区域
2. 升级云服务器带宽
3. 考虑使用 P2P 直连（需要双方网络支持打洞）

---

### 问题 4：Docker 容器无法启动

**症状：** `docker-compose up -d` 后容器立即退出

**排查方法：**
```bash
# 查看详细日志
docker logs hbbs
docker logs hbbr

# 检查端口占用
netstat -tuln | grep 2111
```

**常见原因：**
- 端口被占用：修改 docker-compose.yml 中的端口映射
- IP 配置错误：检查 `<YOUR_IP>` 是否正确

---

## 4. 进阶配置

### 4.1 设置固定密码

在被控端 RustDesk 中：
1. 点击 **设置** → **安全**
2. 找到 **固定密码** 或 **永久密码**
3. 输入密码（建议 8 位以上）
4. 保存

### 4.2 Docker 容器管理命令

```bash
# 停止服务
docker-compose down

# 重启服务
docker-compose restart

# 查看日志（实时）
docker logs -f hbbs

# 更新到最新版本
docker-compose pull
docker-compose up -d
```

### 4.3 备份配置

定期备份 `~/rustdesk-server/data/` 目录，其中包含：
- `id_ed25519`：私钥
- `id_ed25519.pub`：公钥

```bash
# 备份
tar -czf rustdesk-backup-$(date +%Y%m%d).tar.gz ~/rustdesk-server/data/

# 恢复
tar -xzf rustdesk-backup-YYYYMMDD.tar.gz -C ~/rustdesk-server/
```

---

## 5. 安全建议

1. **更改默认端口**：避免使用默认端口（可在 docker-compose.yml 修改）
2. **使用强密码**：为 RustDesk 连接设置复杂的固定密码
3. **定期更新**：及时更新 RustDesk Server 版本
4. **限制 IP 访问**：在安全组/防火墙中只允许特定 IP 访问（可选）
5. **启用加密**：RustDesk 设置中勾选 "只允许加密连接"

---

## 6. 参考资料

- [RustDesk 官方网站](https://rustdesk.com/)
- [RustDesk GitHub](https://github.com/rustdesk/rustdesk)
- [RustDesk Server GitHub](https://github.com/rustdesk/rustdesk-server)
- [官方文档](https://rustdesk.com/docs/)

---

## 📝 更新日志

- **2024-10-01**：初始版本，包含常规配置和快速配置法

---

## 💬 反馈与贡献

如果你在使用过程中遇到问题，或有更好的解决方案，欢迎：
- 提交 Issue
- 发起 Pull Request
- 在评论区讨论

---

**祝你部署顺利！🎉**