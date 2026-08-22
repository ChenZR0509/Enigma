# MinIO

---

- 服务器配置：

  ```shell
  #1、创建MinIO目录
  sudo mkdir -p /opt/minio
  sudo chown -R $USER:$USER /opt/minio
  ls -ld /opt/minio
  cd /opt/minio
  mkdir data
  #2、创建 Docker Compose
  sudo vim docker-compose.yml
  services:
  
    minio:
      image: minio/minio:latest
      container_name: minio
      restart: always
  
      command:
        server /data --console-address ":9001"
  
      ports:
        - "9000:9000"
        - "9001:9001"
  
      environment:
        TZ: Asia/Shanghai
  
        MINIO_ROOT_USER: admin
  
        MINIO_ROOT_PASSWORD: 密码
  
      volumes:
        - ./data:/data
  #3、启动Docker
  sudo docker compose pull
  sudo docker compose up -d
  sudo docker compose ps
  #4、Docker日志查看
  sudo docker compose logs -f 
  docker logs minio --tail=50
  ```
  
- MinIO：http://你的数据库服务器IP:9001

  