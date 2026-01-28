### 🚀 Skopeo 静态编译版

本项目提供 Skopeo 的静态编译二进制文件。无需复杂的依赖安装（如 Podman 或 Containers-Common），下载即用，特别适用于 **离线环境**、**容器镜像迁移** 或 **轻量级 CI/CD 构建节点**。

> **核心优势**：不依赖系统库，无需 Root 权限也可运行（仅操作文件时需要权限），自带默认安全策略。

[![GitHub Stars](https://img.shields.io/github/stars/williammmmmmmmm/skopeo-static-builder?style=for-the-badge&logo=github)](https://github.com/williammmmmmmmm/skopeo-static-builder)
[![GitHub Release](https://img.shields.io/github/v/release/williammmmmmmmm/skopeo-static-builder?style=for-the-badge)](https://github.com/williammmmmmmmm/skopeo-static-builder/releases)
[![GitHub All Releases](https://img.shields.io/github/downloads/williammmmmmmmm/skopeo-static-builder/total?style=for-the-badge)](https://github.com/williammmmmmmmm/skopeo-static-builder/releases)

---

### 📦 下载与安装

1. **下载二进制**
   请在 [Releases 页面]([替换成你的 Releases 链接]) 下载适用于您系统的版本（通常为 `skopeo-v1.21.0-amd64.tar.gz ` 或 `skopeo-v1.21.0-arm64.tar.gz `）。
2. **安装到系统路径**

   ```bash
   # 1. 赋予执行权限
   chmod +x skopeo

   # 2. 移动到系统路径（可选，如果不移动，则使用 ./skopeo 调用）
   sudo mv skopeo /usr/local/bin/
   ```
3. **配置默认信任策略**
   为了让 Skopeo 能够正常拉取和操作镜像，需要创建默认的策略文件。请依次执行以下命令：

   ```bash
   # ① 创建配置目录
   sudo mkdir -p /etc/containers

   # ② 创建 policy.json 文件
   cat << 'EOF' | sudo tee /etc/containers/policy.json
   {
       "default": [
           {
               "type": "insecureAcceptAnything"
           }
       ],
       "transports": {
           "docker-daemon": {
               "": [
                   {
                       "type": "insecureAcceptAnything"
                   }
               ]
           }
       }
   }
   EOF
   ```

   *注：此配置允许拉取任何来源的镜像，适用于离线内网环境。*

---

### 🛠️ 快速开始

#### 1. 检查版本

```bash
skopeo --version
```

#### 2. 常用命令示例

- **拉取镜像并保存为离线包**

  ```bash
  # 将远程镜像保存为本地 tar 文件
  skopeo copy docker://nginx:latest docker-archive:/tmp/nginx.tar
  ```
- **将离线包加载到 Docker**

  ```bash
  # 需要 Docker 服务已启动
  skopeo copy docker-archive:/tmp/nginx.tar docker-daemon:nginx:latest
  ```
- **查看镜像信息（不下载）**

  ```bash
  skopeo inspect docker://nginx:latest
  ```

---

### 📂 目录结构建议

为了保持环境整洁，建议在服务器上建立如下结构：

```bash
/opt/skopeo/
├── bin/          # 存放 skopeo 二进制
├── images/       # 存放下载的镜像 tar 包
└── config/       # 存放配置文件 (软链到 /etc/containers)
```

---

### 🔄 后台运行与监控 (可选)

虽然 Skopeo 本身是命令行工具（执行完即退出），但如果您需要将其集成到**长期运行的守护进程**中（例如监听某个目录自动导入镜像），您可以使用以下简单的 `systemd` 服务模板或监控脚本。

#### 方案 A：简单的监控脚本 (monitor.sh)

您可以编写一个脚本，监控某个目录，当有新的 `.tar` 文件放入时自动加载到 Docker。

```bash
#!/bin/bash
# monitor.sh - 监控目录并自动导入镜像

WATCH_DIR="/opt/skopeo/images"
LOG_FILE="/var/log/skopeo-monitor.log"

echo "开始监控目录: $WATCH_DIR" >> $LOG_FILE

# 使用 inotifywait 监听文件创建事件 (需安装 inotify-tools)
inotifywait -m -e create --format '%f' $WATCH_DIR | while read FILENAME; do
    FULL_PATH="$WATCH_DIR/$FILENAME"
  
    # 等待文件写入完成 (简单延迟)
    sleep 2
  
    if [[ "$FILENAME" == *.tar ]]; then
        echo "[$(date)] 检测到新镜像: $FILENAME，正在导入..." >> $LOG_FILE
      
        # 使用 skopeo 导入
        skopeo copy "docker-archive:$FULL_PATH" "docker-daemon:" >> $LOG_FILE 2>&1
      
        if [ $? -eq 0 ]; then
            echo "[$(date)] 成功导入: $FILENAME" >> $LOG_FILE
            # 可选：导入成功后移动或删除文件
            # mv "$FULL_PATH" "/opt/skopeo/images/done/"
        else
            echo "[$(date)] 导入失败: $FILENAME" >> $LOG_FILE
        fi
    fi
done
```

#### 方案 B：Systemd 服务配置

如果您希望上述脚本开机自启，请创建 systemd 服务。

1. 创建服务文件：`sudo vim /etc/systemd/system/skopeo-monitor.service`

```ini
[Unit]
Description=Skopeo Image Import Monitor
After=docker.service
Requires=docker.service

[Service]
Type=simple
ExecStart=/bin/bash /path/to/monitor.sh
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

2. 启用服务：

```bash
sudo systemctl daemon-reload
sudo systemctl enable skopeo-monitor.service
sudo systemctl start skopeo-monitor.service
```

---

### ⚠️ 安全提示

1. **关于 Policy.json**
   本项目默认配置为 `insecureAcceptAnything`，这意味着它会信任任何来源的镜像。这在**离线内网**是安全的，但在**公网环境**使用时，请务必修改 `policy.json` 以限制可信的 CA 证书或仓库。
2. **权限管理**
   运行 Skopeo 操作 `docker-daemon` 时，通常需要与 Docker 守护进程通信，因此可能需要将运行用户加入 `docker` 用户组。

---

### 📬 反馈与贡献

如果遇到问题或有改进建议，欢迎提交 Issue 或 Pull Request。

祝您的镜像分发工作更加顺畅！
