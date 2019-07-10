+++
title= "Webdevops Php Nginx"
date= 2019-07-10T20:51:08+07:00
draft= false
og_image = "2019/05/doggy2.jpg"
categories = ["docker"]
tags = [
  "docker",
  "webdevops",
  "php",
  ]
+++

## ENV ต่างๆที่ควรส่งไปตอนน ใช้ image webdevops/php-nginx

docker-compose.yaml
```
    mycat-web:
        container_name: test-web
        image: webdevops/php-nginx
        ports:
            - "8080:80"
        restart: always
        #networks:
        #    - test_net
        environment:
            TZ: Asia/Bangkok
            PHP_UPLOAD_MAX_FILESIZE: 500m
            PHP_POST_MAX_SIZE: 500m
            FPM_PM_MAX_CHILDREN: 100
            FPM_PM_START_SERVERS: 50
            FPM_PM_MIN_SPARE_SERVERS: 25
            FPM_PM_MAX_SPARE_SERVERS: 100
            FPM_PROCESS_IDLE_TIMEOUT: 10s
            fpm.pool.listen: 0.0.0.0:9000
            fpm.pool.pm.status_path: /status.php
        volumes:
            - /mydata:/app
```

หรือ

```
docker run -d --name nginx-php -p 8000:80 \
-e TZ="Asia/Bangkok" \
-e PHP_UPLOAD_MAX_FILESIZE=500m \
-e FPM_PM_MAX_CHILDREN=100 \
-e FPM_PM_START_SERVERS=50 \
-e FPM_PM_MIN_SPARE_SERVERS=25 \
-e FPM_PM_MAX_SPARE_SERVERS=100 \
-e FPM_PROCESS_IDLE_TIMEOUT=10s \
-e fpm.pool.listen=0.0.0.0:9000 \
-e fpm.pool.pm.status_path="/status.php" \
webdevops/php-nginx
```

สำหรับ ENV อื่น อ่านเพิ่มได้จาก  https://dockerfile.readthedocs.io/en/latest/content/DockerImages/dockerfiles/php-nginx.html

