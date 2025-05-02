# 美化版 Uptime Kuma Docker 镜像

这是一个经过美化的 Uptime Kuma Docker 镜像，体积约为 300MB，比官方镜像更小，同时保留了基础功能。

## 特点

-   基于 Alpine Linux，体积小，资源占用少
-   保留了监控等基本功能
-   优化了依赖，减少了不必要的文件
-   提供了简单的部署方式

## 界面展示

### 登录页面

![login](https://image.cet.buzz/2025/05/login.png)

### 仪表盘页面

![dashboard](https://image.cet.buzz/2025/05/dashboard.png)

### 添加监控

![add-monitor](https://image.cet.buzz/2025/05/add_monitor.png)

### 状态页面

![status-page](https://image.cet.buzz/2025/05/status_page_test.png)

### 设置页面

![settings](https://image.cet.buzz/2025/05/setting.png)

## 快速开始

### 使用 Docker Compose（推荐）

1. 创建一个新目录并进入：

    ```bash
    mkdir uptime-kuma-mod && cd uptime-kuma-mod
    ```

2. 创建 docker-compose.yaml 文件：

    ```bash
    curl -o docker-compose.yaml https://raw.githubusercontent.com/Digital-z/uptime-kuma-mod/main/docker/docker-compose.dist.yaml
    ```

    或手动创建文件，内容如下：

    ```yaml
    version: "3.8"

    services:
        uptime-kuma:
            image: gfxy/uptime-kuma-mod:v1.0-mod
            container_name: uptime-kuma-mod
            volumes:
                - ./data:/app/data
            ports:
                - "3001:3001"
            restart: unless-stopped
            environment:
                - NODE_ENV=production
                # 设置时区
                - TZ=Asia/Shanghai
                # 禁用内置MariaDB
                - UPTIME_KUMA_ENABLE_EMBEDDED_MARIADB=0
    ```

3. 启动服务：

    ```bash
    docker-compose up -d
    ```

4. 访问 Uptime Kuma：
   打开浏览器，访问 http://your-server-ip:3001

### 使用 Docker 命令

```bash
docker run -d \
  --name uptime-kuma-mod \
  -p 3001:3001 \
  -v ./data:/app/data \
  -e TZ=Asia/Shanghai \
  -e UPTIME_KUMA_ENABLE_EMBEDDED_MARIADB=0 \
  --restart unless-stopped \
  gfxy/uptime-kuma-mod:v1.0-mod
```

## 数据备份

所有数据都存储在 `./data` 目录中。要备份数据，只需复制此目录即可。

## 版本信息

-   **v1.0-mod**: 基于 Uptime Kuma 2.0.0-beta.2 构建的美化版本

## 限制

-   此镜像只包含基本功能，某些高级功能可能受限

## 许可证

与原始 Uptime Kuma 项目相同，采用 MIT 许可证。
