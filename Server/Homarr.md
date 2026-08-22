# Homearr

----

- 服务器配置：

  ```shell
  #1、创建Homearr目录
  sudo mkdir -p /opt/homarr
  cd /opt/homarr
  sudo chown -R $USER:$USER /opt/homarr
  #2、生成加密密钥，复制备用
  openssl rand -hex 32
  #2、创建 Docker Compose
  sudo vim docker-compose.yml
  services:
    homarr:
      container_name: homarr
      image: ghcr.io/homarr-labs/homarr:latest
      restart: unless-stopped
  
      ports:
        - "7575:7575"
  
      volumes:
        - ./appdata:/appdata
        - /var/run/docker.sock:/var/run/docker.sock
  
      environment:
        TZ: Asia/Tokyo
        SECRET_ENCRYPTION_KEY: "加密密钥"
  #4、启动Docker
  sudo docker compose pull
  sudo docker compose up -d
  sudo docker compose ps
  #5、Docker日志查看
  sudo docker compose logs -f homarr
  ```

- Homarr配置：

  1. 打开http://服务器IP:7575
  2. 首次登录需创建用户和密码