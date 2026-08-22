# InfluxDB

---

- 服务器配置：

  ```shell
  #1、创建Caddy目录
  sudo mkdir -p /opt/influxdb
  sudo chown -R $USER:$USER /opt/influxdb
  ls -ld /opt/influxdb
  cd /opt/influxdb
  mkdir data config
  #2、创建 Docker Compose
  sudo vim docker-compose.yml
  services:
  
    influxdb:
      image: influxdb:2
      container_name: influxdb
      restart: always
  
      ports:
        - "8086:8086"
  
      environment:
        TZ: Asia/Shanghai
  
        DOCKER_INFLUXDB_INIT_MODE: setup
  
        DOCKER_INFLUXDB_INIT_USERNAME: admin
  
        DOCKER_INFLUXDB_INIT_PASSWORD: 密码
  
        DOCKER_INFLUXDB_INIT_ORG: home
  
        DOCKER_INFLUXDB_INIT_BUCKET: default
  
        DOCKER_INFLUXDB_INIT_RETENTION: 365d
  
      volumes:
        - ./data:/var/lib/influxdb2
        - ./config:/etc/influxdb2
  #3、启动Docker
  sudo docker compose pull
  sudo docker compose up -d
  sudo docker compose ps
  #4、Docker日志查看
  sudo docker compose logs -f 
  docker logs redis --tail=50
  ```
  
- Redis：http://服务器IP:8086

  ```shell
  docker exec -it influxdb bash
  influx
  influx auth list
  ```
  
  