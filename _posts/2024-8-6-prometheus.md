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

访问地址：http://xx.xx.xx.xx:9090/graph

### 1.3 node_exporter安装

node_exporter 安装到被监视节点，prometheus通过接口http://xx.xx.xx.xx:9100/metrics获取采集数据

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
      summary: "redis异常，实例：{{$labels.instance}}"
      description: "{{$labels.job}} redis已关闭"
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

访问地址：http://xx.xx.xx.xx:3000

- 设置prometheus数据源 Home > Connections > Data sources > Add data source
- 建立dashboard视图，或者从grafana 市场上拷贝ID模版https://grafana.com/grafana/dashboards/11835/，grafana导入模版生效

 




