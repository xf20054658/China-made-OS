docker run -it \
  --name openclaw-data-agent \
  -u node \
  --entrypoint /bin/bash \
  -v ~/.openclaw-rockylinux-data:/home/node \
  rockylinux-openclaw-data:v1.0

  openclaw setup