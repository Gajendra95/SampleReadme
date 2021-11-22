WiproDWI Application with Grafana, Loki, Prometheus, Promtail and MySQL with backup script
========

A monitoring solution for Docker hosts and containers with [Prometheus](https://prometheus.io/), [Grafana](http://grafana.org/), 
[NodeExporter](https://github.com/prometheus/node_exporter) and alerting.  


## Install

### Create .env:
```
MYSQL_DATABASE_HOST=mysqldb
MYSQL_DATABASE_USER=root
MYSQL_DATABASE=admin_wiprodwi
MYSQL_ROOT_PASSWORD=123456
```

Store the .env file inside mysql folder.

### Create backup.sh:
```
#!/bin/sh
# Dump DBs
#timestamp=`date +%Y%m%d_%H%M`
now=$(date +"%d-%m-%Y_%H-%M")
mysqldump -h mysqldb -u root --password="123456" --routines admin_wiprodwi > /home/backups/admin_wiprodwi-$now.sql

# remove all files (type f) modified longer than 7 days ago under /db_backups/backups
find /home/backups -name "*.sql" -type f -mtime +7 -delete

exit 0
```

Store the backup.sh file inside mysql folder.

### Create Dockerfile:
```
FROM mysql:8.0
RUN apt-get update && apt-get install -y cron
RUN mkdir /home/backups
ADD crontab.conf /crontab.conf
COPY backup.sh /home/backup.sh
COPY entry.sh /entry.sh
CMD cat crontab.conf >> /etc/crontab
RUN chmod 755 /home/backups
RUN chmod 755 /entry.sh
RUN chmod 755 /home/backup.sh 
RUN chmod +x /home/backup.sh 
RUN /usr/bin/crontab /etc/crontab
CMD ["cron","-f"]
```

Store the Dockerfile inside mysql folder.

### Create nginx.conf:
```
upstream ssl.is.dev.dwi.com {
    server server:8086 max_fails=0;
    server server:3005 max_fails=0;
    ip_hash;
}

server {
        # this server listens on port 80
        listen 443 ssl;
        #listen [::]:80 default_server;

        # name this server "nodeserver", but we can call it whatever we like
        server_name dwi.ax;

        ssl_certificate /etc/nginx/security/certificate.crt;
        ssl_certificate_key /etc/nginx/security/privatekey.key;

        # the location / means that when we visit the root url (localhost:80/), we use this configuration
        location / {
                # a bunch of boilerplate proxy configuration
                proxy_set_header X-Forwarded-Host $host;
                proxy_set_header X-Forwarded-Server $host;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header Host $http_host;
                proxy_read_timeout 5m;
                proxy_send_timeout 5m;
                proxy_pass http://ssl.is.dev.dwi.com/;

                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection "upgrade";
        }
}
```

Store the nginx.conf inside nginx folder.

### Create Dockerfile:
```
FROM ubuntu/nginx:latest
RUN apt-get update -y
COPY ./default.conf /etc/nginx/conf.d/
COPY ./security/ /etc/nginx/security/
```

Create self-signed(ssl) certificates and place it inside security folder in nginx.
Store the Dockerfile inside nginx folder.

### Create prometheus.yml:
```
global:
  scrape_interval:     15s

scrape_configs:
  - job_name: "prometheus"
    scrape_interval: 5s
    static_configs:
    - targets: ["localhost:9090"]

  - job_name: "node"
    static_configs:
    - targets: ["node-exporter:9100"]

remote_write:
  - url: "http://prometheus:9090"
    basic_auth:
      username: "admin"
      password: "admin"
```

Store the prometheus.yml inside prometheus folder.

### Create supervisord.conf:
```
[supervisord]
nodaemon=true

[program:apiserver]
directory=/app
command=node /app/server.js

[program:app]
directory=/app
command=node /app/appserver.js

```

Store the supervisord.conf file inside dwi folder.


### Edit dwi/app/config/db.config.js:
```
module.exports = {
  HOST: "mysqldb",             //Hostname of the db server
  USER: "root",                //Username of the mysql server
  PASSWORD: "123456",          //Password of the mysql server
  DB: "admin_wiprodwi",        //Database of the mysql server
  dialect: "mysql",
  pool: {
    max: 5,
    min: 0,
    acquire: 30000,
    idle: 10000
  }
};

```

### Create Dockerfile:
```
FROM node:16.11.1
RUN apt-get -y update
RUN apt-get install -y supervisor
RUN apt install default-mysql-client -y
WORKDIR /app
COPY package.json .
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

COPY . .

RUN npm install

RUN npm run build
EXPOSE 3005
EXPOSE 8086
RUN chmod +x server.js
CMD ["/usr/bin/supervisord"]
```

Store the Dockerfile inside dwi folder.

### Create docker-compose.yml:
```
version: "3.8"

services:

  mysqldb:
    image: mysql:8.0
    restart: unless-stopped
    networks:
      - antiz_net 
    volumes:
      - /var/lib/mysql
    env_file:
      - ./mysql/MySQL.env
    ports:
      - 3306:3306
    volumes:
      - dbdata:/var/lib/mysql
      - ./MySQL.env:/docker-entrypoint-initdb.d/MySQL.env
      - ./mysql/dwi.sql:/docker-entrypoint-initdb.d/dwi.sql
      
  cron_mysql:
    build:
      context: ./mysql/
    image: cron_mysql
    container_name: cron_mysql
    restart: unless-stopped
    user: root
    tty: true
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - cron_dbdata:/home/backups
      - C:\docker\Volumes\mysql:/home/backups
    networks:
      - antiz_net
    depends_on:
      - mysqldb

  server:
    depends_on:
      - mysqldb
    image: server
    build: ./dwi/
    tty: true
    container_name: wiprodwi
    env_file: 
      - ./dwi/.env.development
      - ./MySQL/MySQL.env
    networks:
      - antiz_net
    volumes:
      - wiprodata:/app
    ports:
      - '8086:8086'
      - '3005:3005'
    logging:
      driver: loki
      options:
        loki-url: http://host.docker.internal:3100/loki/api/v1/push
    
  nginx:
    depends_on:
      - server
    image: nginx
    build:
      context: ./nginx
    container_name: nginx
    hostname: nginx
    tty: true
    ports:
      - "80:80"
      - "443:443"
    networks:
      - antiz_net
    logging:
      driver: loki
      options:
        loki-url: http://host.docker.internal:3100/loki/api/v1/push

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    expose:
      - 9100
    networks:
      - antiz_net

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--web.enable-lifecycle'
    expose:
      - 9090
    networks:
      - antiz_net

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    env_file:
      - ./grafana/.env.grafana
    restart: unless-stopped
    expose:
      - 3000
    ports:
      - "3000:3000"
    networks:
      - antiz_net
  
  loki:
    image: grafana/loki:2.3.0
    container_name: loki
    ports:
      - 127.0.0.1:3100:3100
    command: -config.file=/etc/loki/local-config.yaml
    volumes:
      - ./loki/config.yaml:/etc/loki/config.yaml
      - loki_data:/data/loki
    networks:
      - antiz_net

  promtail:
    image: grafana/promtail:2.3.0
    container_name: promtail
    volumes:
      - /var/log:/var/log
      - /var/lib/docker/containers:/var/lib/docker/containers
      - ./promtail:/etc/promtail-config/
    command: -config.file=/etc/promtail/config.yml
    networks:
      - antiz_net


networks:
  antiz_net:

volumes:
  dbdata:
  prometheus_data: {}
  grafana_data: {}
  loki_data: {}
  wiprodata:
  cron_dbdata:
```

Store the Dockerfile inside dwi folder.

### Clone this repository on your Docker host, cd into test directory and run compose up:

```
git clone https://github.com/
cd Docker-Compose-Prometheus-and-Grafana
docker-compose up -d
```

## Prerequisites:

* Docker Engine >= 1.13
* Docker Compose >= 1.11

## Containers:

* Prometheus (metrics database) `http://<host-ip>:9090`
* Prometheus-Pushgateway (push acceptor for ephemeral and batch jobs) `http://<host-ip>:9091`
* mysql (database) 
* Grafana (visualize metrics) `http://<host-ip>:3000`
* NodeExporter (host metrics collector)
* wiprodwi (application) for the application: `localhost:3005` for the server: `localhost:8086`
* nginx 'https://dwi.ax'

## Setup Grafana

Navigate to `http://<host-ip>:3000` and login with user ***admin*** password ***admin***. You can change the credentials in the compose file or by supplying the `ADMIN_USER` and `ADMIN_PASSWORD` environment variables via .env file on compose up. The config file can be added directly in grafana part like this
```
grafana:
  image: grafana/grafana:5.2.4
  env_file:
    - config

```
and the config file format should have this content
```
GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=changeme
GF_USERS_ALLOW_SIGN_UP=false
```
If you want to change the password, you have to remove this entry, otherwise the change will not take effect
```
- grafana_data:/var/lib/grafana
```

Grafana is preconfigured with dashboards and Prometheus as the default data source:

* Name: Prometheus
* Type: Prometheus
* Url: http://prometheus:9090
* Access: proxy

***Docker Host Dashboard***

![Host](https://raw.githubusercontent.com/Einsteinish/Docker-Compose-Prometheus-and-Grafana/master/screens/Grafana_Docker_Host.png)

The Docker Host Dashboard shows key metrics for monitoring the resource usage of your server:

* Server uptime, CPU idle percent, number of CPU cores, available memory, swap and storage
* System load average graph, running and blocked by IO processes graph, interrupts graph
* CPU usage graph by mode (guest, idle, iowait, irq, nice, softirq, steal, system, user)
* Memory usage graph by distribution (used, free, buffers, cached)
* IO usage graph (read Bps, read Bps and IO time)
* Network usage graph by device (inbound Bps, Outbound Bps)
* Swap usage and activity graphs

For storage and particularly Free Storage graph, you have to specify the fstype in grafana graph request.
You can find it in `grafana/dashboards/docker_host.json`, at line 480 :

      "expr": "sum(node_filesystem_free_bytes{fstype=\"btrfs\"})",

I work on BTRFS, so i need to change `aufs` to `btrfs`.

You can find right value for your system in Prometheus `http://<host-ip>:9090` launching this request :

      node_filesystem_free_bytes

***Docker Containers Dashboard***

![Containers](https://raw.githubusercontent.com/Einsteinish/Docker-Compose-Prometheus-and-Grafana/master/screens/Grafana_Docker_Containers.png)

The Docker Containers Dashboard shows key metrics for monitoring running containers:

* Total containers CPU load, memory and storage usage
* Running containers graph, system load graph, IO usage graph
* Container CPU usage graph
* Container memory usage graph
* Container cached memory usage graph
* Container network inbound usage graph
* Container network outbound usage graph

Note that this dashboard doesn't show the containers that are part of the monitoring stack.

***Monitor Services Dashboard***

![Monitor Services](https://raw.githubusercontent.com/Einsteinish/Docker-Compose-Prometheus-and-Grafana/master/screens/Grafana_Prometheus.png)
![Monitor Services](https://raw.githubusercontent.com/Einsteinish/Docker-Compose-Prometheus-and-Grafana/master/screens/Grafana_Prometheus2.png)
![Monitor Services](https://raw.githubusercontent.com/Einsteinish/Docker-Compose-Prometheus-and-Grafana/master/screens/Grafana_Prometheus3.png)

The Monitor Services Dashboard shows key metrics for monitoring the containers that make up the monitoring stack:

* Prometheus container uptime, monitoring stack total memory usage, Prometheus local storage memory chunks and series
* Container CPU usage graph
* Container memory usage graph
* Prometheus chunks to persist and persistence urgency graphs
* Prometheus chunks ops and checkpoint duration graphs
* Prometheus samples ingested rate, target scrapes and scrape duration graphs
* Prometheus HTTP requests graph
* Prometheus alerts graph

