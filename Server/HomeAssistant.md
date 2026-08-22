# HomeAssistant

---

- 服务器配置：

  ```shell
  #1、创建Homearr目录
  sudo mkdir -p /opt/homeassistant/config
  cd /opt/homeassistant
  sudo chown -R $USER:$USER /opt/homeassistant
  #2、创建 Docker Compose
  sudo vim docker-compose.yml
  services:
    homeassistant:
      container_name: homeassistant
      image: ghcr.io/home-assistant/home-assistant:stable
      restart: unless-stopped
  
      network_mode: host
  
      environment:
        - TZ=Asia/Shanghai
  
      volumes:
        - /opt/homeassistant/config:/config
  #3、启动Docker
  sudo docker compose pull
  sudo docker compose up -d
  sudo docker compose ps
  #4、Docker日志查看
  sudo docker compose logs -f
  #5、安装Hacs，下载完成后在集成中添加HACS即可
  cd /opt/homeassistant
  sudo docker exec -it homeassistant bash
  wget -O - https://get.hacs.xyz | bash -
  exit
  sudo docker restart homeassistant
  ```

  
