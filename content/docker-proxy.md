Title: docker 代理
Date: 2024-6-11 21:30
Modified: 024-6-11 21:30
Category: docker
Tags: pelican, publishing
Slug: my-super-post
Authors: zero

## 背景: docker代理仓库各种都不太好使了、整理一下常见操作操作

1. 直接修改/etc/systemd/system/docker.service.d/http-proxy.conf, 具体可以详情参考https://docs.docker.com/config/daemon/systemd/,相关配置如下:
```sh
[Service]
Environment="HTTP_PROXY=http://proxy.example.com:3128"
Environment="HTTPS_PROXY=https://proxy.example.com:3129"
```

2. 服务器上运行registry(注意存储空间和过期时间), 客户端配置daemon.json配置
```yaml
version: "3.9"

services:
  
  redis:
    image: redis
    restart: always
    container_name: redis
    networks:
      - docker-registry
    volumes:
      - ./redis-data:/data
    command: ["redis-server", "--maxmemory", "512m"]

  dockerhub-mirror:
    image: registry:2
    container_name: registry
    restart: always
    networks:
      - docker-registry
    volumes:
      - /srv/docker/dockerhub/data:/var/lib/registry
      - /srv/docker/dockerhub/config.yml:/etc/docker/registry/config.yml:ro
    ports:
      - "5000:5000/tcp"
    logging:·
      driver: journald
      options:
        tag: "dockerd-dockerhub"

```
```sh
{
        "registry-mirrors": [
                "https://hub-cache.moelove.info"
        ]
}
```

3. nexus 

4. cloudflare 

