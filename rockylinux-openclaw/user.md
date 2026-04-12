# 导出 image
docker save -o rockylinux-openclaw-v2.0.0.tar rockylinux-openclaw:v2.0.0

- 运行容器，使用挂载目录：~/.openclaw-rockylinux
docker run -it --rm -p 18789:18789 -p 18790:18790 -v ~/.openclaw-rockylinux:/home/node rockylinux-openclaw:9

docker compose up -d

如果报错，先交互式运行：
docker run -it --rm \
  -v ~/.openclaw-rockylinux:/home/node \
  rockylinux-openclaw:9 \
  openclaw setup

查看日志
docker logs openclaw-service

登录容器
docker exec -it openclaw-service /bin/bash


cat .openclaw/openclaw.json
{
  "meta": {
    "lastTouchedVersion": "2026.3.13",
    "lastTouchedAt": "2026-03-17T00:02:54.546Z"
  },
  "agents": {
    "defaults": {
      "workspace": "/home/node/.openclaw/workspace",
      "compaction": {
        "mode": "safeguard"
      }
    }
  },
  "commands": {
    "native": "auto",
    "nativeSkills": "auto",
    "restart": true,
    "ownerDisplay": "raw"
  },
  "gateway": {
    "mode": "local",
    "auth": {
      "mode": "token",
      "token": "ab545a7eaf44934865eb240d1f7340f83e3dea44e525a5fa"
    }
  }
}
