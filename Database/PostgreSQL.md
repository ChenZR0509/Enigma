# PostgreSQL 

---

- 服务器配置：

  ```shell
  #1、创建Caddy目录
  sudo mkdir -p /opt/PostgreSQL
  sudo chown -R $USER:$USER /opt/PostgreSQL
  ls -ld /opt/PostgreSQL
  cd /opt/PostgreSQL
  #2、创建 Docker Compose
  sudo vim docker-compose.yml
  services:
  
    postgres:
      image: postgres:16
      container_name: postgres
      restart: always
  
      environment:
        POSTGRES_USER: postgres
        POSTGRES_PASSWORD: 管理员密码
        TZ: Asia/Shanghai
  
      ports:
        - "5432:5432"
  
      volumes:
        - ./data:/var/lib/postgresql/data
  #3、启动Docker
  sudo docker compose pull
  sudo docker compose up -d
  sudo docker compose ps
  #4、Docker日志查看
  sudo docker compose logs -f 
  ```
  
- PostgreSQL ：

  ```shell
  docker exec -it postgres bash
  psql -U postgres
  ```

  