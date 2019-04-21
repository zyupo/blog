title:  Prometheus+Grafana快速部署监控系统
date: 2018/6/2  11:36
categories: 技术分享
---
### prometheus监控介绍：
{% blockquote %}
Prometheus是一套开源的监控&报警&时间序列数据库的组合,起始是由SoundCloud公司开发的。成立于2012年，之后许多公司和组织接受和采用prometheus,他们便将它独立成开源项目，并且有公司来运作.该项目有非常活跃的社区和开发人员，目前是独立的开源项目，任何公司都可以使用它，2016年，Prometheus加入了云计算基金会，成为kubernetes之后的第二个托管项目.google SRE的书内也曾提到跟他们BorgMon监控系统相似的实现是Prometheus。现在最常见的Kubernetes容器管理系统中，通常会搭配Prometheus进行监控。
{% endblockquote %}
### 主要功能：

* 多维 数据模型（时序由 metric 名字和 k/v 的 labels 构成）。
* 灵活的查询语句（PromQL）。
* 无依赖存储，支持 local 和 remote 不同模型。
* 采用 http 协议，使用 pull 模式，拉取数据，简单易懂。
* 监控目标，可以采用服务发现或静态配置的方式。
* 支持多种统计数据模型，图形化友好。

### 核心组件：
* Prometheus Server， 主要用于抓取数据和存储时序数据，另外还提供查询和 Alert Rule 配置管理。
* client libraries，用于对接 Prometheus Server, 可以查询和上报数据。
* push gateway ，用于批量，短期的监控数据的汇总节点，主要用于业务数据汇报等。
* 各种汇报数据的 exporters ，例如汇报机器数据的 node_exporter, 汇报 MongoDB 信息的 MongoDB exporter 等等。
* 用于告警通知管理的 alertmanager 。

## 基础架构
{% img [图片] /images/201806/prometheus.jpg %}

大致逻辑：
1. Prometheus server 定期从静态配置的 targets 或者服务发现的 targets 拉取数据。
2. 当新拉取的数据大于配置内存缓存区的时候，Prometheus 会将数据持久化到磁盘（如果使用 remote storage 将持久化到云端）。
3. Prometheus 可以配置 rules，然后定时查询数据，当条件触发的时候，会将 alert 推送到配置的 Alertmanager。
4. Alertmanager 收到警告的时候，可以根据配置，聚合，去重，降噪，最后发送警告。
5. 可以使用 API， Prometheus Console 或者 Grafana 查询和聚合数据

### 配置环境：
* 服务端：172.31.15.120
* 客户端： 172.31.17.93
* 服务端和客户端需要提前安装好Docker

--------------------------------------------

## Prometheus服务器部署

1、拉取prometheus服务端docker镜像：
```
docker pull prom/prometheus
```

2、配置prometheus,建议将prometheus监控数据存放到宿主机
```
mkdir -p /data/prometheus_data
```
3、授权数据目录权限，这里的nfsnobody.nfsnobody，是ID为65534的用户，prometheus用户在容器内部用65534用户运行。
```

chown -R nfsnobody.nfsnobody /data/prometheus_data
```
4、启动prometheus
```
docker run -d --name=prometheus-tmp -v /data/prometheus_data/:/data/prometheus_data/ prom/prometheus 
```
-----
## 部署 Grafana

1、下载Grafana镜像
```
docker pull grafana/grafana
```

2、创建存放Grafana的数据目录
```
mkdir -p /data/grafana/
```
3、启动Grafana：
```
docker run -d --user root -v /data/grafana/:/var/lib/grafana --name=grafana -p 3001:3000 grafana/grafana
```

-----

## 被监控的客户端安装
1、下载收集信息的镜像：
```
docker pull prom/node-exporter
```

2、启动收集客户端
```
docker run -d -p 9100:9100 prom/node-exporte
```
----
## 配置服务端图形展示
1、进入服务端prometheus容器,将配置文件prometheus拷贝到系统主机目录，方便后面修改配置文件做准备
```
docker exec -it prometheus-tmp cp /etc/prometheus/prometheus.yml /data/prometheus_data/prometheus.yml
docker stop prometheus-tmp
docker rm prometheus-tmp
```
2、编辑prometheus配置文件
```
vim /data/prometheus_data/prometheus.yml
 
  - job_name: 'client01'
    static_configs:
      - targets: ['172.31.17.93:9100']
        labels:
          host: client01
```
3、重新启动新的prometheus容器
```
docker run -d --name=prometheus -p 9090:9090 -v /data/prometheus_data/:/data/prometheus_data/ prom/prometheus --config.file=/data/prometheus_data/prometheus.yml --storage.tsdb.path=/data/prometheus_data/
```

4、浏览器访问：http://服务端ip/9090,点击Status--Targets，可以看到
{% img [图片] /images/201806/prometheus9090.jpg %}

5、配置grafan，浏览器访问：http://服务端ip:3001，默认账号密码：admin/admin
{% img [图片] /images/201806/grafana3001.jpg %}

6、添加数据源：设置—>Data Source,
* name是添加数据源的名字
* Type选择Prometheus
* Url输入服务端IP（因为grafana在容器运行，写默认localhost可能找不到
* 最后添加Save & Test
{% img [图片] /images/201806/addsource.jpg %}


7、导入模板：https://grafana.com/dashboards/1860  
{% img [图片] /images/201806/import.jpg %}

8、导入完成后，就可以看到客户端监控的界面了。
{% img [图片] /images/201806/grafana01.jpg %}

9、可以根据自己需要适当减少不需要的模板
{% img [图片] /images/201806/grafana02.jpg %}

-----
结束

后续完善：prometheus自动注册，目前客户端主机需要自己手动修改peometheus配置文件添加。


