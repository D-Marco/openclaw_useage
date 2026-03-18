# OpenClaw Docker 安装步骤

> 参考官方文档：https://docs.openclaw.ai/install/docker

---

## 系统要求

- Docker Desktop（或 Docker Engine）+ Docker Compose v2
- 至少 2GB RAM（用于镜像构建）
- 足够的磁盘空间（用于存储镜像和日志）

---

## 快速安装（推荐）

在仓库根目录下执行以下脚本：

```bash
./docker-setup.sh
```

该脚本会自动完成以下操作：

1. 构建或拉取网关镜像
2. 运行入职向导
3. 通过 Docker Compose 启动网关
4. 生成网关令牌并写入 `.env` 文件

安装完成后，打开浏览器访问：

```
http://127.0.0.1:18789/
```

将生成的令牌粘贴到控制 UI 中即可完成配置。

---

## 常用配置选项

### 使用远程预构建镜像

```bash
export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
./docker-setup.sh
```

### 启用代理沙箱

```bash
export OPENCLAW_SANDBOX=1
./docker-setup.sh
```

### 安装额外系统包

```bash
export OPENCLAW_DOCKER_APT_PACKAGES="ffmpeg build-essential"
./docker-setup.sh
```

### 预装扩展依赖

```bash
export OPENCLAW_EXTENSIONS="diagnostics-otel matrix"
./docker-setup.sh
```

---

## 手动安装

如需手动控制每个步骤，可按以下顺序执行：

```bash
# 第一步：构建镜像
docker build -t openclaw:local -f Dockerfile .

# 第二步：运行入职向导
docker compose run --rm openclaw-cli onboard

# 第三步：启动网关服务
docker compose up -d openclaw-gateway
```

---

## 故障排除

### 遇到"未授权"或配对错误

重新获取仪表板链接：

```bash
docker compose run --rm openclaw-cli dashboard --no-open
```

---

## 相关链接

- 官方文档：https://docs.openclaw.ai/install/docker
- 完整文档索引：https://docs.openclaw.ai/llms.txt
