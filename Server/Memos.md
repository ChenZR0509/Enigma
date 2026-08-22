# Memos

----

- 服务器配置：

  ```shell
  #1、创建Memos目录
  sudo mkdir -p /opt/memos
  cd /opt/memos
  sudo chown -R $USER:$USER /opt/memos
  #2、创建 Docker Compose
  sudo vim docker-compose.yml
  services:
    memos:
      image: neosmemo/memos:stable
      container_name: memos
      restart: unless-stopped
      ports:
        - "5230:5230"
      volumes:
        - ./data:/var/opt/memos
  #4、启动Docker
  sudo docker compose pull
  sudo docker compose up -d
  sudo docker compose ps
  #5、Docker日志查看
  sudo docker compose logs -f homarr
  ```

  