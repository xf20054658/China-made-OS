# 基于官方 Rocky Linux 最新稳定版镜像（当前为 Rocky Linux 9）
FROM rockylinux:9

# 维护者信息（可选）
LABEL maintainer="ives"

# 切换为 root 用户执行安装操作
USER root

# 1. 配置国内 yum/dnf 源（使用小写文件名匹配）
RUN sed -e 's|^mirrorlist=|#mirrorlist=|g' \
    -e 's|^#baseurl=http://dl.rockylinux.org/$contentdir|baseurl=https://mirrors.aliyun.com/rockylinux|g' \
    -i.bak /etc/yum.repos.d/rocky*.repo

# 2. 安装必要依赖（curl、wget、gcc 等，gcc 用于后续可能的 npm 编译）
RUN dnf install -y --allowerasing curl wget gcc-c++ make sudo git && \
    dnf clean all && \
    rm -rf /var/cache/dnf

# 3. 安装 Node.js 官方源（获取最新稳定版 Node）
RUN curl -fsSL https://rpm.nodesource.com/setup_current.x | bash -

# 4. 安装 Node.js（包含 npm）
RUN dnf install -y nodejs && \
    dnf clean all && \
    rm -rf /var/cache/dnf

# 5. 验证 Node 和 npm 版本（可选，用于构建时确认安装成功）
RUN node -v && npm -v

# 6. 全局安装 OpenClaw
# 注意：全局安装需要 root 权限，且建议配置 npm 国内镜像加速构建
RUN npm config set registry https://registry.npmmirror.com && \
    npm install -g openclaw@latest


# 7. 创建 node 用户并准备工作空间
RUN id -u node >/dev/null 2>&1 || useradd -m node

RUN mkdir -p /home/node
WORKDIR /home/node
RUN chown -R node:node /home/node


# 8. 设置工作目录（按要求指定为 /home/node）
WORKDIR /home/node

# 9. 暴露端口（在切换用户前或后均可，通常放在这里）
EXPOSE 18789 18790

# 切换为非 root 的 node 用户（生产环境推荐）
USER node

# 9. 启动命令（请确保 openclaw 命令已正确安装到 PATH）
# 如果是通过 npm -g 安装，通常可以直接使用 openclaw
CMD ["openclaw", "gateway", "--bind", "auto","--allow-unconfigured"]