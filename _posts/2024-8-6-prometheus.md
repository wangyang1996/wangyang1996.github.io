---
layout:     post                    # 使用的布局（不需要改）
title:      prometheus + grafana            # 标题 
subtitle:   prometheus、grafana                     #副标题
date:       2024-08-06             # 时间
author:     wangyang                     # 作者
header-img: img/post-bg-desk.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - prometheus
    - grafana
---


Prometheus+Grafana
=================
本篇文章记录prometheus、grafana安装部署流程，相关使用操作

1.prometheus
--------------------------

### 1.1 prometheus采集方式

> prometheus数据采集方式分为pull和push方式

```
pull    
客户端安装各式exporter插件，服务端拉取metrics采样数据
push
通过自定义脚本推送数据到PushGateWay服务，PushGateWay再推送采样数据到prometheus
```


### 1.2 prometheus安装

```
wget prometheus-2.52.0.linux-amd64.tar.gz
tar -zxvf prometheus-2.52.0.linux-amd64.tar.gz
cd prometheus-2.52.0.linux-amd64
##使用daemonize方式放入后台运行
git clone https://gitcode.com/bmc/daemonize.git
sh configure && make && sudo make install
echo '/usr/local/prometheus/prometheus-2.52.0.linux-amd64/prometheus --web.enable-lifecycle --config.file=prometheus.yml'  >  up.sh
daemonize -c /usr/local/prometheus/prometheus-2.52.0.linux-amd64/ /usr/local/prometheus/prometheus-2.52.0.linux-amd64/up.sh
##写入服务器启动文件
vi /etc/rc.local
daemonize -c /usr/local/prometheus/prometheus-2.52.0.linux-amd64/ /usr/local/prometheus/prometheus-2.52.0.linux-amd64/up.sh
chmod +x /etc/rc.local
```

访问地址：`http://xx.xx.xx.xx:9090/graph`

### 1.3 node_exporter安装

node_exporter 安装到被监视节点，prometheus通过接口`http://xx.xx.xx.xx:9100/metrics`获取采集数据

	wget node_exporter-1.8.1.linux-amd64.tar.gz
	tar -zxvf node_exporter-1.8.1.linux-amd64.tar.gz
	cd node_exporter-1.8.1.linux-amd64
	##使用daemonize方式放入后台运行
	git clone https://gitcode.com/bmc/daemonize.git
	sh configure && make && sudo make install
	echo '/usr/local/node_exporter/node_exporter-1.8.1.linux-amd64/node_exporter'  >  up.sh
	daemonize -c /usr/local/node_exporter/node_exporter-1.8.1.linux-amd64/ /usr/local/node_exporter/node_exporter-1.8.1.linux-amd64/up.sh
	##写入服务器启动文件
	vi /etc/rc.local
	daemonize -c /usr/local/node_exporter/node_exporter-1.8.1.linux-amd64/ /usr/local/node_exporter/node_exporter-1.8.1.linux-amd64/up.sh
	chmod +x /etc/rc.local

### 1.4 prometheus配置

prometheus.yml 新增监控组

```
scrape_configs:
  - job_name: "node_exporter"
    static_configs:
      - targets: ["localhost:9100","localhost:9200"]
```

prometheus.yml 配置alert.yml文件地址(相对路径)

```
rule_files:
  - "alert.yml"
```

alert.yml配置告警

```
- name: redis
  rules:
  - alert: RedisDown
    expr: redis_up == 0
    for: 0m
    labels:
      severity: critical
    annotations:
      summary: "redis异常，实例：\{\{$labels.instance\}\}"
      description: "\{\{$labels.job\}\} redis已关闭"
```

检查配置

```
./promtool check config prometheus.yml
```

prometheus重载配置生效

```
curl -X POST http://localhost:9090/-/reload 
```



2.grafana
--------------------------

### 2.1 grafana安装

    sudo yum install -y https://dl.grafana.com/enterprise/release/grafana-enterprise-11.0.0-1.x86_64.rpm
    service grafana-server start
    service grafana-server status

访问地址：`http://xx.xx.xx.xx:3000`

- 设置prometheus数据源 `Home > Connections > Data sources > Add data source`
- 建立dashboard视图，或者从grafana 市场上拷贝ID模版`https://grafana.com/grafana/dashboards/11835/`，grafana导入模版生效



## 3.prometheus+grafana 基础服务实战

### 3.1 ngnix监控

1. `nginx -V 2>&1 | grep -o with-http_stub_status_module`   检查是否安装该模块

2. nginx.conf文件server模块添加配置

   ```
   location /stub_status {
   	stub_status on;
   	access_log off;
   	allow 0.0.0.0/0;
   	deny all;
   }
   ```

3. nginx重新加载配置

   ```
   nginx -t
   nginx -s reload
   curl http://xx.xx.xx.xx/stub_status 
   ```

4. nginx 安装对应exporter模块

   `image:nginx/nginx-prometheus-exporter`

5. Docker启动nginx_exporter容器

   `docker run -d --name nginx-exporter -p 9113:9113 nginx/nginx-prometheus-exporter -nginx.scrape-uri=http://xx.xx.xx.xx/stub_status`    注意参数

6. `http://xx.xx.xx.xx:9113/metrics`     获取nginx相关采集数据

7. prometheus.yml  先添加 job_name 地址

   ```
    - job_name: "nginx_exporter"
       static_configs:
         - targets: ["xx.xx.xx.xx:9113"]
   ```

8. alert.yml  添加触发器配置信息

   ```
   - name: nginx
     rules:
     - alert: NginxDown
       expr: nginx_up == 0
       for: 30s
       labels: 
         severity: critical
       annotations:
         summary: "nginx异常，实例：\{\{$labels.instance\}\}"
         description: "\{\{$labels.job\}\} nginx已关闭"
   ```

9. `./promtool check config prometheus.yml`   检查配置

10. `curl -X POST http://localhost:9090/-/reload`      prometheus重新加载配置文件

11. grafana 市场上拷贝ID 模版`https://grafana.com/grafana/dashboards/12708-nginx/`，grafana导入模版。



### 3.2 redis监控

1. redis 安装对应exporter模块

   `image: oliver006/redis_exporter`

2. 启动exporter 容器

   `docker run -d --name redis-exporter -p 9121:9121 oliver006/redis_exporter --redis.addr redis://xx.xx.xx.xx:6379 --redis.password '123456'`    // 注意参数

3. `http://xx.xx.xx.xx:9121/metrics`      获取redis相关参数数据

4. prometheus.yml  先添加 job_name 地址

   ```
   - job_name: "redis_exporter"
       static_configs:
         - targets: ["xx.xx.xx.xx:9121"]
   ```

5. alert.yml  添加触发器配置信息

   ```
   groups:
   - name: redis
     rules:
     - alert: RedisDown
       expr: redis_up == 0
       for: 0m
       labels:
         severity: critical
       annotations:
         summary: "redis异常，实例：\{\{$labels.instance\}\}"
         description: "\{\{$labels.job\}\} redis已关闭"
   ```

6. `./promtool check config prometheus.yml` 检查配置

7. `curl -X POST http://localhost:9090/-/reload`      prometheus重新加载配置文件

8. grafana 市场上拷贝ID 模版`https://grafana.com/grafana/dashboards/11835/`，grafana导入模版。

   

### 3.3 rabbitmq监控

1. rabbitmq 安装对应exporter模块

   `image: kbudde/rabbitmq-exporter`

2. 启动exporter 容器

   `docker run -d --name rabbitmq-exporter -p 9419:9419 -e RABBIT_URL=http://xx.xx.xx.xx:15672 -e RABBIT_USER=guest -e RABBIT_PASSWORD=guest kbudde/rabbitmq-exporter`    // 注意 env 参数

3. `http://xx.xx.xx.xx:9419/metrics`      获取rabbitmq相关参数数据

4. prometheus.yml  先添加 job_name 地址

   ```
   - job_name: "rabbitmq_exporter"
       static_configs:
         - targets: ["xx.xx.xx.xx:9419"]
   ```

5. alert.yml  添加触发器配置信息

   ```
   groups:
   - name: rabbitmq
     rules:
     - alert: RabbitmqDown
       expr: rabbitmq_up != 1
       for: 0m
       labels:
         severity: critical
       annotations:
         summary: "rabbitmq异常，实例：\{\{$labels.instance\}\}"
         description: "\{\{$labels.job\}\} rabbitmq已关闭"
   ```

6. `./promtool check config prometheus.yml` 检查配置

7. `curl -X POST http://localhost:9090/-/reload`      prometheus重新加载配置文件

8. grafana 市场上拷贝ID 模版`https://grafana.com/grafana/dashboards/4279/`，grafana导入模版。



### 3.4 docker监控

1. docker 安装对应exporter模块

   `image: google/cadvisor`

2. 启动exporter 容器

   `docker run -d --volume=/:/rootfs:ro --volume=/var/run:/var/run:rw --volume=/sys:/sys:ro --volume=/var/lib/docker/:/var/lib/docker:ro --publish=8400:8080 --name=cadvisor google/cadvisor `    // 注意参数

3. `http://xx.xx.xx.xx:8400/metrics`      获取docker相关参数数据

4. prometheus.yml  先添加 job_name 地址

   ```
   - job_name: "docker_cadvisor"
       static_configs:
         - targets: ["xx.xx.xx.xx:8400"]
   ```

5. `./promtool check config prometheus.yml` 检查配置

6. `curl -X POST http://localhost:9090/-/reload`      prometheus重新加载配置文件

7. grafana 市场上拷贝ID 模版`https://grafana.com/grafana/dashboards/11600/`，grafana导入模版。



### 3.5 mysql监控

1. mysql 安装对应exporter模块

   `image: prom/mysqld-exporter`

2. 启动exporter 容器

   `docker run -d --name mysql-exporter -e DATA_SOURCE_NAME="exporter:123456@(xx.xx.xx.xx:3306)/" -p 9104:9104 prom/mysqld-exporter `    // 注意参数

3. `http://xx.xx.xx.xx:9104/metrics`      获取mysql相关参数数据

4. prometheus.yml  先添加 job_name 地址

   ```
   - job_name: "mysql_exporter"
       static_configs:
         - targets: ["xx.xx.xx.xx:9104"]
   ```

5. `./promtool check config prometheus.yml` 检查配置

6. `curl -X POST http://localhost:9090/-/reload`      prometheus重新加载配置文件

7. grafana 市场上拷贝ID 模版`https://grafana.com/grafana/dashboards/7362/`，grafana导入模版。



## 4.PushGateWay方式实战

1. pushgateway  安装对应exporter模块

   `image: prom/pushgateway`

2. 启动exporter 容器

   `docker run -d --name pushgateway -p 9091:9091 prom/pushgateway `    // 注意参数

3. `http://xx.xx.xx.xx:9091/metrics`      获取mysql相关参数数据

4. prometheus.yml  先添加 job_name 地址

   ```
   - job_name: "pushgateway"
       honor_labels: true
       static_configs:
         - targets: ["xx.xx.xx.xx:9091"]
   ```

5. `./promtool check config prometheus.yml` 检查配置

6. `curl -X POST http://localhost:9090/-/reload`      prometheus重新加载配置文件

7. 向pushgateway测试推送数据：

   `echo ‘指标名 指标值’ ｜ curl --data-binary @- http://xx.xx.xx.xx:9091/metrics/job/job_name`  

8. `http://xx.xx.xx.xx:9091/metrics` 查看推送指标

9. shell 脚本推送到pushgatway , crontab 定期发送 +  sleep方式

   shell脚本：

   ```
   #!/bin/sh
   use_memory=`free -m | awk 'NR==2{print $3}'`
   echo "use_memory_mb ${use_memory}" | curl --data-binary @- http://xx.xx.xx.xx:9091/metrics/job/test_job/instance/test
   ```

   crontab:

   ```
   * * * * * /bin/sh /usr/local/prometheus/shell/use_memory.sh  >/dev/null 2>&1
   * * * * * sleep 15; /bin/sh /usr/local/prometheus/shell/use_memory.sh  >/dev/null 2>&1
   * * * * * sleep 30; /bin/sh /usr/local/prometheus/shell/use_memory.sh  >/dev/null 2>&1
   * * * * * sleep 45; /bin/sh /usr/local/prometheus/shell/use_memory.sh  >/dev/null 2>&1
   ```

10. grafana上制作dashboard模版

11. pushgateway 数据不更新的话，是旧的数据

    删除pushgateway指标

    `curl -X DELETE http://xx.xx.xx.xx:9091/metrics/job/test_job/instance/test`

    