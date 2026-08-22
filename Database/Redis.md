# Redis

---

- 服务器配置：

  ```shell
  #1、创建Caddy目录
  sudo mkdir -p /opt/redis
  sudo chown -R $USER:$USER /opt/redis
  ls -ld /opt/redis
  cd /opt/redis
  mkdir data
  #2、创建 Docker Compose
  sudo vim docker-compose.yml
  services:
  
    redis:
      image: redis:7
      container_name: redis
      restart: always
  
      command:
        redis-server
        --requirepass 密码
  
      ports:
        - "6379:6379"
  
      volumes:
        - ./data:/data
  
      environment:
        TZ: Asia/Shanghai
  #3、启动Docker
  sudo docker compose pull
  sudo docker compose up -d
  sudo docker compose ps
  #4、Docker日志查看
  sudo docker compose logs -f 
  docker logs redis --tail=50
  ```
  
- Redis：

  ```shell
  docker exec -it redis bash
  redis-cli
  AUTH 密码
  ```
  
  