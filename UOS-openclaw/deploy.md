# OpenClaw 基于统信UOS+openEuler源部署手册
## 文档说明
本文档适用于**统信UOS服务器版**环境下，基于openEuler-20.03-LTS-SP3源部署OpenClaw服务的完整流程，包含环境准备、镜像构建、容器配置、服务启动及验证全步骤，操作命令均经过实测，可直接按步骤执行。

## 一、部署环境要求
### 1. 基础环境
- 操作系统：统信UOS服务器版（兼容aarch64架构）
- Docker环境：已安装Docker及Docker Compose（建议Docker 20.10+版本）
- 网络要求：服务器可访问外网（用于拉取基础镜像、安装依赖、下载OpenClaw）
- 权限要求：操作用户拥有**root/ sudo**权限，可执行Docker命令

### 2. 本地文件准备
在服务器任意目录（如`/opt/openclaw-deploy`）创建部署目录，并放入以下文件/目录：
```
/opt/openclaw-deploy/
├── Dockerfile       # 本手册提供的镜像构建文件
├── node22-bin/      # Node.js 22 可执行文件目录（含node、npm等二进制文件）
└── docker-compose.yml # 容器编排配置文件（手动创建，本手册提供模板）
|-- .env #环境变量
```
> 注：`node22-bin/`需为aarch64架构的Node.js 22完整可执行目录，确保文件有可执行权限。

## 二、核心配置文件准备
### 2.1 Dockerfile 文件
在部署目录`/opt/openclaw-deploy/`下创建`Dockerfile`，复制以下内容（已优化冗余命令、配置openEuler源、添加OpenClaw本地模式及跳过配置校验）：
```dockerfile
# 统信UOS服务器版基础镜像
FROM registry.uniontech.com/uos-server-base/uos-server-20-1070e:latest
LABEL maintainer="清风"
USER root

# 移除统信原生源，配置openEuler-20.03-LTS-SP3 aarch64源
RUN rm -rf /etc/yum.repos.d/UnionTechOS* && \
cat > /etc/yum.repos.d/openEuler.repo << 'EOF'
[OS]
name=openEuler-20.03-LTS-SP3 - OS
baseurl=https://repo.openeuler.org/openEuler-20.03-LTS-SP3/OS/aarch64/
enabled=1
gpgcheck=1
gpgkey=https://repo.openeuler.org/openEuler-20.03-LTS-SP3/OS/aarch64/RPM-GPG-KEY-openEuler
[update]
name=openEuler-20.03-LTS-SP3 - update
baseurl=https://repo.openeuler.org/openEuler-20.03-LTS-SP3/update/aarch64/
enabled=1
gpgcheck=1
gpgkey=https://repo.openeuler.org/openEuler-20.03-LTS-SP3/OS/aarch64/RPM-GPG-KEY-openEuler
EOF

# 安装基础依赖（python3、git）
RUN dnf install -y --skip-broken python3 && \
    dnf install -y git

# 安装Node.js 22（本地拷贝二进制文件，无需解压/下载）
RUN mkdir -p /usr/local/node22
COPY node22-bin/ /usr/local/node22/
RUN chmod +x /usr/local/node22/bin/node /usr/local/node22/bin/npm
ENV PATH=/usr/local/node22/bin:$PATH
# 验证Node.js安装
RUN node -v && npm -v

# 全局安装OpenClaw（国内镜像加速，避免网络问题）
RUN npm config set registry http://registry.npmmirror.com && \
    npm config set strict-ssl false && \
    npm install -g openclaw@latest && \
    # 恢复npm默认配置，避免后续依赖安装冲突
    npm config set strict-ssl true && \
    npm config set registry https://registry.npmjs.org && \
    # 验证OpenClaw安装
    openclaw -v

# 创建node用户（规避root运行风险，赋予家目录权限）
RUN id -u node >/dev/null 2>&1 || useradd -m -s /bin/bash node && \
    mkdir -p /home/node && \
    chown -R node:node /home/node

# 工作目录与端口暴露（与OpenClaw默认端口一致）
WORKDIR /home/node
EXPOSE 18789 18790

# OpenClaw核心配置：本地模式+环境变量
ENV OPENCLAW_GATEWAY_MODE=local
# 切换非root用户，启动OpenClaw（--allow-unconfigured跳过配置校验）
USER node
CMD ["openclaw", "gateway", "--bind", "0.0.0.0","--allow-unconfigured"]
```

### 2.2 docker-compose.yml 配置文件
在部署目录下创建`docker-compose.yml`，使用以下模板（配置端口映射、容器持久化、自动重启）：
```yaml
version: '3.8'
services:
  openclaw:
    build: .
    image: uos-openclaw:v1.0  # 自定义镜像名+标签
    container_name: openclaw-service-uos  # 容器名称，固定便于管理
    ports:
      - "19791:18789"  # 宿主机API端口:容器内API端口
      - "19792:18790"  # 宿主机控制台端口:容器内控制台端口
    volumes:
      - /opt/openclaw/data:/home/node  # 宿主机数据目录:容器工作目录，持久化配置/数据
    restart: unless-stopped  # 容器异常退出时自动重启
    stdin_open: true
    tty: true
    user: node  # 以node用户运行容器，与Dockerfile一致
```
> 注：宿主机端口`19791/19792`可根据实际情况修改，需保证未被占用；`/opt/openclaw/data`为宿主机持久化目录，会自动创建。

### 2.3 .env 环境变量文件（可选）
若需灵活修改端口，可在部署目录下创建`.env`文件，替代`docker-compose.yml`中的固定端口：
```env
# OpenClaw服务端口配置
OPENCLAW_API_PORT=19791
OPENCLAW_CTRL_PORT=19792
# 宿主机持久化数据目录
OPENCLAW_DATA_PATH=/opt/openclaw/data
```
修改`docker-compose.yml`的`ports`和`volumes`节点，适配环境变量：
```yaml
ports:
  - "${OPENCLAW_API_PORT:-18789}:18789"
  - "${OPENCLAW_CTRL_PORT:-18790}:18790"
volumes:
  - "${OPENCLAW_DATA_PATH:-/opt/openclaw/data}:/home/node"
```

## 三、镜像构建
1. 进入部署目录：
```bash
cd /opt/openclaw-deploy
```
2. 执行Docker镜像构建命令（首次构建需下载基础镜像、安装依赖，耗时约5-10分钟，视网络情况而定）：
```bash
docker-compose build
```
> 若未安装Docker Compose，可使用原生Docker命令构建：`docker build -t uos-openclaw:v1.0 .`
3. 构建完成后，验证镜像是否生成：
```bash
docker images | grep uos-openclaw
```
若输出镜像名、标签、ID等信息，说明镜像构建成功。

## 四、容器启动与初始化配置
### 4.1 启动临时容器（交互式配置OpenClaw）
因首次启动无配置文件会导致容器启动失败，需先启动**临时交互式容器**，执行`openclaw setup`完成核心配置：
1. 停止/删除可能存在的失效容器（清理环境）：
```bash
docker stop openclaw-service-uos 2>/dev/null || true
docker rm openclaw-service-uos 2>/dev/null || true
```
2. 启动临时交互式容器（绕过默认启动命令，直接进入bash）：
```bash
docker run -it \
  --name openclaw-temp \
  -u node \
  --entrypoint /bin/bash \
  -v /opt/openclaw/data:/home/node \
  uos-openclaw:v1.0
```
执行成功后，终端提示符变为`node@xxxx:/home/node$`，代表已进入容器。

### 4.2 容器内执行OpenClaw初始化配置
在容器内的交互式终端中，执行以下命令开始配置，**全程按回车使用默认配置**即可（本地模式推荐默认）：
```bash
openclaw setup
```
#### 关键配置项提示（默认即可）
- **Gateway mode**：本地模式直接回车（默认`local`）
- **Workspace directory**：工作空间默认`/home/node/.openclaw`，回车
- **API/Control Port**：默认`18789/18790`，回车（与Dockerfile暴露端口一致）
- 其余配置（日志级别、缓存路径等）：全部回车使用默认

#### 配置成功标志
终端输出`Setup completed successfully!`，无任何报错，代表配置文件已生成在`/home/node/.openclaw`目录。

### 4.3 退出临时容器并保留配置
1. 在容器内执行以下命令，回到宿主机终端：
```bash
exit
```
2. 临时容器的配置已自动同步到宿主机持久化目录（因`docker-compose.yml`已配置挂载），无需额外拷贝，直接删除临时容器即可：
```bash
docker rm openclaw-temp
```

## 五、正式启动OpenClaw服务
1. 回到部署目录，执行Docker Compose启动命令，后台运行容器：
```bash
cd /opt/openclaw-deploy
docker-compose up -d
```
2. 验证容器启动状态：
```bash
docker ps | grep openclaw-service-uos
```
若输出中`STATUS`列显示`Up`，代表容器启动成功。

## 六、服务验证
### 6.1 查看容器启动日志（核心验证）
执行以下命令，查看OpenClaw启动日志，**无`Missing config`报错即代表配置生效**：
```bash
docker logs openclaw-service-uos
```
#### 成功日志特征
```
 OpenClaw 2026.3.13 (61d171a)
   I'll refactor your busywork like it owes me money.

10:20:27 Config overwrite: /home/node/.openclaw/openclaw.json (sha256 dc7a7982d2bef7ad147542798f4f74a230494e8f02fb13d6caca287ef4322041 -> 97c3c63d64ca0a96b8996c364a94d0e51c09f1921380a2be3aaa7b4497cff32d, backup=/home/node/.openclaw/openclaw.json.bak)
10:20:27 [gateway] auth token was missing. Generated a new token and saved it to config (gateway.auth.token).
10:20:27 [canvas] host mounted at http://127.0.0.1:18789/__openclaw__/canvas/ (root /home/node/.openclaw/canvas)
10:20:27 [heartbeat] started
10:20:27 [health-monitor] started (interval: 300s, startup-grace: 60s, channel-connect-grace: 120s)
10:20:27 [gateway] agent model: anthropic/claude-opus-4-6
10:20:27 [gateway] listening on ws://127.0.0.1:18789, ws://[::1]:18789 (PID 13)
10:20:27 [gateway] log file: /tmp/openclaw-1000/openclaw-2026-03-21.log
10:20:27 [browser/server] Browser control listening on http://127.0.0.1:18791/ (auth=token)
```
仅显示OpenClaw版本和`Gateway online`提示，无持续的配置缺失报错。

### 6.2 验证容器内服务运行
进入运行中的容器，验证OpenClaw进程是否正常：
```bash
# 进入容器交互式终端
docker exec -it -u node openclaw-service-uos /bin/bash
# 验证OpenClaw命令是否可用
openclaw -v
# 退出容器
exit
```

### 6.3 端口连通性验证
在服务器本地，执行以下命令验证端口是否监听（替换为实际配置的宿主机端口）：
```bash
# 验证19791端口
telnet 127.0.0.1 19791
# 或使用netstat
netstat -tulpn | grep 19791
```
若显示`Connected to 127.0.0.1`或端口处于`LISTEN`状态，代表端口映射成功。

## 七、日常运维命令
### 7.1 容器基础操作
```bash
# 启动服务
docker-compose up -d
# 停止服务
docker-compose down
# 重启服务
docker-compose restart
# 查看实时日志
docker logs -f openclaw-service-uos
# 进入容器操作
docker exec -it -u node openclaw-service-uos /bin/bash
```

### 7.2 镜像/容器清理
```bash
# 删除失效容器
docker rm $(docker ps -a -f status=exited -q)
# 删除无用镜像
docker rmi $(docker images -f dangling=true -q)
```

### 7.3 数据备份
宿主机`/opt/openclaw/data`目录为OpenClaw的核心数据目录，包含配置、工作空间、会话数据，备份时直接打包该目录即可：
```bash
tar -zcvf openclaw-data-backup-$(date +%Y%m%d).tar.gz /opt/openclaw/data
```

## 八、常见问题排查
### 8.1 容器启动失败，提示`port is already allocated`
**原因**：宿主机配置的端口（如19791/19792）被其他进程/容器占用。
**解决**：
1. 查找占用端口的进程：
```bash
netstat -tulpn | grep 19791
```
2. 杀死占用进程（替换`<PID>`为实际进程ID）：
```bash
kill -9 <PID>
```
3. 或修改`docker-compose.yml`/`.env`中的宿主机端口，重新启动。

### 8.2 日志持续报`Missing config`
**原因**：未执行`openclaw setup`初始化配置，或配置文件未持久化。
**解决**：重新执行**第四章**的临时容器配置步骤，确保配置文件生成后再启动正式容器。

### 8.3 镜像构建失败，提示`dnf: command not found`
**原因**：统信UOS基础镜像未自带dnf，或源配置失败。
**解决**：检查Dockerfile中`openEuler.repo`的baseurl是否正确，确保服务器可访问openEuler源，或替换为`yum`命令（部分UOS版本为yum）。

### 8.4 容器启动后，端口无法访问
**原因**：服务器防火墙/安全组未开放端口，或端口映射配置错误。
**解决**：
1. 开放宿主机端口（以firewalld为例）：
```bash
firewall-cmd --add-port=19791/tcp --permanent
firewall-cmd --add-port=19792/tcp --permanent
firewall-cmd --reload
```
2. 检查云服务器安全组，放行对应的端口。

## 九、部署完成说明
OpenClaw服务部署完成后，可通过**宿主机IP+配置端口**访问服务，默认本地访问地址为`http://127.0.0.1:19791`（API端口）、`http://127.0.0.1:19792`（控制台端口）。

服务为**后台常驻运行**，容器配置了`restart: unless-stopped`，服务器重启后会自动启动，无需手动操作。