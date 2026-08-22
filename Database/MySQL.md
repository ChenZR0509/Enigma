# MySQL

---

- 服务器配置：

  ```shell
  #1、创建Caddy目录
  sudo mkdir -p /opt/mysql
  sudo chown -R $USER:$USER /opt/mysql
  ls -ld /opt/mysql
  cd /opt/mysql
  #2、创建 Docker Compose
  sudo vim docker-compose.yml
  services:
    mysql:
      image: mysql:8.4
      container_name: mysql
      restart: always
  
      environment:
        MYSQL_ROOT_PASSWORD: 密码
        TZ: Asia/Shanghai
  
      ports:
        - "3306:3306"
  
      volumes:
        - ./data:/var/lib/mysql
        - ./conf:/etc/mysql/conf.d
        - ./logs:/var/log/mysql
  #3、创建配置目录
  mkdir conf logs data
  #4、添加优化配置
  sudo vim conf/my.cnf
  [mysqld]
  
  # 字符集
  character-set-server=utf8mb4
  collation-server=utf8mb4_unicode_ci
  
  # 时区
  default-time-zone='+08:00'
  
  # InnoDB优化
  innodb_buffer_pool_size=2G
  innodb_log_file_size=512M
  
  # 连接
  max_connections=200
  
  # 日志
  slow_query_log=1
  long_query_time=2
  
  
  [client]
  
  default-character-set=utf8mb4
  #4、启动Docker
  sudo docker compose pull
  sudo docker compose up -d
  sudo docker compose ps
  #5、Docker日志查看
  sudo docker compose logs -f 
  ```

- MySQL：

  ```shell
  docker exec -it mysql bash
  mysql -uroot -p
  ```

  