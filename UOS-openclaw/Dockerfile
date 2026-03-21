# 统信UOS服务器版基础镜像
FROM registry.uniontech.com/uos-server-base/uos-server-20-1070e:latest
LABEL maintainer="lxf20054658@163.com"
USER root


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

RUN dnf install -y --skip-broken python3
RUN dnf install -y  git


# 第一步：创建Node.js安装目录（统信原生mkdir命令均支持）
RUN mkdir -p /usr/local/node22

# 第二步：拷贝本地Node.js 22可执行文件到镜像内（仅用COPY，规避解压/下载命令）
COPY node22-bin/ /usr/local/node22/

# 第三步：赋予可执行权限（统信原生chmod命令支持）
RUN chmod +x /usr/local/node22/bin/node /usr/local/node22/bin/npm

# 第四步：配置环境变量，将node/npm加入系统PATH，全局可用
ENV PATH=/usr/local/node22/bin:$PATH

# 验证安装（可选，基础镜像若支持echo则可添加）
RUN node -v && npm -v

# 全局安装OpenClaw（国内镜像加速，关闭SSL，纯二进制安装无需编译）
RUN npm config set registry http://registry.npmmirror.com && \
    npm config set strict-ssl false && \
    npm install -g openclaw@latest && \
    # 恢复npm默认配置，避免后续冲突
    npm config set strict-ssl true && \
    npm config set registry https://registry.npmjs.org && \
    # 验证OpenClaw安装
    openclaw -v

# 用镜像内置useradd/id/chown创建node用户（无sudo，直接root创建后切换，规避sudo依赖）
RUN id -u node >/dev/null 2>&1 || useradd -m -s /bin/bash node && \
    mkdir -p /home/node && \
    chown -R node:node /home/node
RUN mkdir -p /home/node && chown -R node:node /home/node

# 工作目录与端口暴露
WORKDIR /home/node
EXPOSE 18789 18790

# 设置OpenClaw本地模式，跳过配置校验
ENV OPENCLAW_GATEWAY_MODE=local

# 切换非root用户，启动OpenClaw
USER node
CMD ["openclaw", "gateway", "--bind", "0.0.0.0","--allow-unconfigured"]