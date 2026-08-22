# Grafana

---

- 服务器配置：

  ```shell
  #1、创建Caddy目录
  sudo mkdir -p /opt/grafana
  sudo chown -R $USER:$USER /opt/grafana
  ls -ld /opt/grafana
  cd /opt/grafana
  mkdir data
  sudo chown -R 472:472 data
  #2、创建 Docker Compose
  sudo vim docker-compose.yml
  services:
  
    grafana:
      image: grafana/grafana:latest
      container_name: grafana
      restart: always
  
      ports:
        - "3000:3000"
  
      environment:
        TZ: Asia/Shanghai
  
        GF_SECURITY_ADMIN_USER: admin
  
        GF_SECURITY_ADMIN_PASSWORD: 密码
  
      volumes:
        - ./data:/var/lib/grafana
  #3、启动Docker
  sudo docker compose pull
  sudo docker compose up -d
  sudo docker compose ps
  #4、Docker日志查看
  sudo docker compose logs -f 
  docker logs grafana --tail=50
  ```
  
- Grafana：http://你的数据库服务器IP:3000

  