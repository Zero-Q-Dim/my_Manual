### 1 安装前准备

### 1.1 主机环境

准备一台虚拟机

| IP地址     | 操作系统 | 配置            |
| ---------- | -------- | --------------- |
| 20.0.0.214 | AnolisOS | 4核4G，100G磁盘 |

### 1.2 规划安装目录

将prometheus相关服务都安装在/opt/prometheus目录下面，最好/opt是一块单独的磁盘，易于扩容

```text
mkdir -p /opt/{prometheus,grafana,alertmanager,node_exporter}
```

### 1.3 下载安装包

**下载地址：**

- [https://prometheus.io/download/](https://link.zhihu.com/?target=https%3A//prometheus.io/download/)
- [https://grafana.com/grafana/download/](https://link.zhihu.com/?target=https%3A//grafana.com/grafana/download/)

**版本信息：**

- **Prometheus版本：2.53.4**
- **grafana版本：11.5.3**
- **alertmanager版本：0.28.1**
- **exporter版本：1.9.0**

```text
# 进入/opt目录
cd /opt

# 下载prometheus
wget https://github.com/prometheus/prometheus/releases/download/v2.53.4/prometheus-2.53.4.linux-amd64.tar.gz

# 下载grafana
wget https://dl.grafana.com/enterprise/release/grafana-enterprise-11.5.3.linux-amd64.tar.gz

# 下载altermanager
wget https://github.com/prometheus/alertmanager/releases/download/v0.28.1/alertmanager-0.28.1.linux-amd64.tar.gz

# 下载node_exporter
wget https://github.com/prometheus/node_exporter/releases/download/v1.9.0/node_exporter-1.9.0.linux-amd64.tar.gz
```

### 2 安装prometheus相关服务

### 2.1 安装Prometheus

**解压安装包**

```text
[root@localhost ~]# cd /opt
[root@localhost data]# tar -xvf prometheus-2.53.4.linux-amd64.tar.gz  
[root@localhost data]# mv prometheus-2.53.4.linux-amd64/* /opt/prometheus
```

**创建prometheus用户**

```text
useradd -M -s /sbin/nologin  prometheus
```

**授予prometheus目录权限**

```text
chown -R prometheus.prometheus  /opt/prometheus
```

**给prometheus创建[systemd服务](https://zhida.zhihu.com/search?content_id=255975730&content_type=Article&match_order=1&q=systemd服务&zhida_source=entity)**

```text
cat >> /etc/systemd/system/prometheus.service << EOF
[Unit]
Description=Prometheus Server
Documentation=https://prometheus.io/docs/introduction/overview
After=network.target

[Service]
Type=simple
User=prometheus
Group=prometheus
Restart=on-failure
ExecStart=/opt/prometheus/prometheus \
  --config.file=/opt/prometheus/prometheus.yml \
  --storage.tsdb.path=/opt/prometheus/opt \
  --storage.tsdb.retention.time=15d \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
EOF
```

**重载服务**

```text
systemctl daemon-reload
```

**启动prometheus并设置开机自启动**

```text
systemctl enable --now prometheus.service
```

**检查状态**

```text
systemctl status prometheus.service
```

**访问prometheus**

| prometheus | [http://20.0.0.214:9090](https://link.zhihu.com/?target=http%3A//20.0.0.214%3A9090) | 无用户无密码 |
| ---------- | ------------------------------------------------------------ | ------------ |
| 监控指标   | [http://20.0.0.214:9090](https://link.zhihu.com/?target=http%3A//20.0.0.214%3A9090)/metrics |              |

![image-20260324145401572](C:\Users\KeroYoshI\AppData\Roaming\Typora\typora-user-images\image-20260324145401572.png)

![image-20260324145727003](C:\Users\KeroYoshI\AppData\Roaming\Typora\typora-user-images\image-20260324145727003.png)

### 2.2 安装alertmanager

**解压安装包**

```text
tar -xvf alertmanager-0.28.1.linux-amd64.tar.gz
```

**解压的内容复制到/opt/alertmanager目录**

```text
mv /opt/alertmanager-0.28.1.linux-amd64/* /opt/alertmanager
```

**更改alertmanager权限**

```text
chown -R prometheus.prometheus /opt/alertmanager
```

**给alertmanager创建systemd服务**

```text
cat >> /etc/systemd/system/alertmanager.service << EOF
[Unit]
Desciption=Alert Manager
wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=prometheus
Group=prometheus
ExecStart=/opt/alertmanager/alertmanager \
--config.file=/opt/alertmanager/alertmanager.yml \
--storage.path=/opt/alertmanager/opt
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```

**启动alertmanager**

```text
# 重载服务
systemctl daemon-reload

# 启动并设置开机自启动
systemctl enable --now  alertmanager.service
```

**查看alertmanager状态**

```text
systemctl status alertmanager
```

**将alertmanager加入prometheus。**

```text
vi  /opt/prometheus/prometheus.yml

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
        # 根据实际填写alertmanager的IP地址
            - 20.0.0.214:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # 根据实际名修改文件名，可以有多个规则文件
  - "/opt/alertmanager/rule/alert.yml"
```



**增加触发器配置文件**

```text
# 新建存放告警文件目录
mkdir /opt/alertmanager/rule
chown -R prometheus.prometheus /opt/alertmanager

# 编辑配置文件
vim /opt/alertmanager/rule/alert.yml 

groups:
  - name: 主机状态监控
    rules:
      - alert: 主机宕机
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.instance }} 主机宕机，请尽快处理"
          description: "{{ $labels.instance }} 已经宕机超过 1 分钟。请检查服务状态。"
```

检查配置

```text
cd /opt/prometheus/
[root@localhost prometheus]# ./promtool  check config prometheus.yml
Checking prometheus.yml
  SUCCESS: 1 rule files found
 SUCCESS: prometheus.yml is valid prometheus config file syntax

Checking /opt/alertmanager/rule/alert.yml
  SUCCESS: 1 rules found
```



一定要检测通过再进行重启prometheus

**重启prometheus**

```text
systemctl restart prometheus
```

**访问alertmanager**：

[http://20.0.0.214:9093](https://link.zhihu.com/?target=http%3A//20.0.0.214%3A9093)

![image-20260324145541099](C:\Users\KeroYoshI\AppData\Roaming\Typora\typora-user-images\image-20260324145541099.png)

### 2.3 安装node_exporter

**解压安装包**

```text
tar -xvf node_exporter-1.9.0.linux-amd64.tar.gz
```

**解压的内容复制到/opt/node_exporter目录**

```text
mv node_exporter-1.9.0.linux-amd64/* /opt/node_exporter
```

**修改权限**

```text
chown prometheus.prometheus  -R /opt/node_exporter
```

**给node_exporter创建systemd服务**

```text
cat >> /etc/systemd/system/node_exporter.service << EOF
[Unit]
Description=node_exporter
Documentation=https://prometheus.io/
After=network.target

[Service]
User=prometheus
Group=prometheus
ExecStart=/opt/node_exporter/node_exporter
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

启动node_exporter

```text
systemctl daemon-reload
systemctl enable --now node_exporter.service
```

查看状态

```text
systemctl status node_exporter
```

**访问地址：**

[http://20.0.0.214:9100/metrics](https://link.zhihu.com/?target=http%3A//20.0.0.214%3A9100/metrics)

![image-20260324145618282](C:\Users\KeroYoshI\AppData\Roaming\Typora\typora-user-images\image-20260324145618282.png)

### 2.4 配置exporter到prometheus

```text
vi /opt/prometheus/prometheus.yml
# 在尾部添加一个job_name,可以添加多个targets
- job_name: "node_exporter"
    static_configs:
      - targets: ["20.0.0.214:9100"]
        labels:
          instance: 20.0.0.214服务器
```

```bash
# my global config
global:
  scrape_interval: 15s
  evaluation_interval: 15s

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - 20.0.0.214:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"
  - "/opt/alertmanager/rule/alert.yml"

# A scrape configuration containing exactly one endpoint to scrape:
scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
        labels:
          app: "prometheus"

  - job_name: "node_exporter"
    static_configs:
      - targets: ["20.0.0.214:9100"]
        labels:
          instance: "20.0.0.214服务器"
```

重载prometheus

```text
# 重启前检查配置是否正确
./promtool  check config prometheus.yml

# 平滑加载
curl -X POST http://20.0.0.214:9090/-/reload

# 或者直接重启
systemctl restart prometheus
```

登录prometheus查看-node_exporter是否起来了

### 2.5 安装Grafana

**解压安装包**

```text
tar -xvf grafana-enterprise-11.5.3.linux-amd64.tar.gz
```

**将解压内容移动到/opt/grafana**

```text
mv grafana-v11.5.3/* /opt/grafana
```

**更改grafana目录权限**

```text
chown -R prometheus.prometheus  /opt/grafana
```

**给grafana创建systemd服务**

```text
cat >> /etc/systemd/system/grafana-server.service << EOF
[Unit]
Description=Grafana server
Documentation=http://dosc.grafana.org

[Service]
Type=simple
User=prometheus
Group=prometheus
Restart=on-failure
ExecStart=/opt/grafana/bin/grafana-server --config=/opt/grafana/conf/defaults.ini --homepath=/opt/grafana

[Install]
WantedBy=multi-user.target
EOF
```

**启动grafana**

```text
# 重载系统服务
systemctl daemon-reload

# 启动并设置开机自启动
systemctl enable --now grafana-server.service
```

**查看状态**

```text
systemctl status grafana-server.service
```

**访问grafana**

[http://20.0.0.214:3000](https://link.zhihu.com/?target=http%3A//20.0.0.214%3A3000)

默认用户名/密码：admin/admin

![image-20260324142953228](C:\Users\KeroYoshI\AppData\Roaming\Typora\typora-user-images\image-20260324142953228.png)

### 2.6 grafana对接prometheus

![image-20260324143140643](C:\Users\KeroYoshI\AppData\Roaming\Typora\typora-user-images\image-20260324143140643.png)

![image-20260324143222025](C:\Users\KeroYoshI\AppData\Roaming\Typora\typora-user-images\image-20260324143222025.png)

从Grafana官网导入符合要求的仪表盘

[https://grafana.com/grafana/dashboards](https://link.zhihu.com/?target=https%3A//grafana.com/grafana/dashboards)

在grafana右上角处点击Import dashboard，

![image-20260324144123255](C:\Users\KeroYoshI\AppData\Roaming\Typora\typora-user-images\image-20260324144123255.png)

导入id号或json文件，在grafana官网可以直接获取。

![image-20260324151909684](C:\Users\KeroYoshI\AppData\Roaming\Typora\typora-user-images\image-20260324151909684.png)

![image-20260324153436739](C:\Users\KeroYoshI\AppData\Roaming\Typora\typora-user-images\image-20260324153436739.png)



***

- ### 1. **Node Exporter**

  - **功能**：用于收集主机（节点）的硬件和操作系统指标，如 CPU 使用率、内存使用、磁盘 I/O、网络流量等。
  - **用途**：监控服务器的整体性能，帮助运维人员及时发现瓶颈。

  ### 2. **cAdvisor**

  - **功能**：专门用于监控运行在 Docker 容器中的应用程序，收集容器的 CPU、内存、网络和文件系统使用情况。
  - **用途**：提供容器级别的资源使用情况，适用于容器化环境的监控。

  ### 3. **Blackbox Exporter**

  - **功能**：用于外部服务的可用性监控，通过 HTTP、TCP、ICMP 等协议检查服务是否可用。
  - **用途**：监控网站、API 和其他网络服务的响应时间和可用性。

  ### 4. **MySQL Exporter**

  - **功能**：用于从 MySQL 数据库中收集性能指标，如查询速度、连接数、慢查询等。
  - **用途**：帮助数据库管理员监控 MySQL 数据库的性能和健康状况。

  ### 5. **PostgreSQL Exporter**

  - **功能**：类似于 MySQL Exporter，用于从 PostgreSQL 数据库中收集指标。
  - **用途**：监控 PostgreSQL 数据库的性能，识别潜在问题。

  ### 6. **Redis Exporter**

  - **功能**：从 Redis 数据库中收集性能指标，如内存使用、命中率、连接数等。
  - **用途**：帮助开发和运维人员监控 Redis 的性能。

  ### 7. **Kafka Exporter**

  - **功能**：用于监控 Apache Kafka 集群，收集主题的消息速率、消费者组状态等指标。
  - **用途**：确保 Kafka 集群的健康和性能。

  ### 8. **JMX Exporter**

  - **功能**：用于监控 Java 应用程序，特别是使用 JMX（Java Management Extensions）的应用程序。
  - **用途**：收集 JVM 相关的性能指标，如内存使用、线程状态、垃圾回收等。

  ### 9. **SNMP Exporter**

  - **功能**：用于通过 SNMP 协议从网络设备（如路由器、交换机、防火墙等）收集性能指标。
  - **用途**：监控网络基础设施的健康状况。

  ### 10. **Windows Exporter (formerly WMI Exporter)**

  - **功能**：用于从 Windows 系统收集性能指标，如 CPU、内存、磁盘 I/O 等。
  - **用途**：监控 Windows 服务器的性能。

  ### 11. **Kube-state-metrics**

  - **功能**：用于监控 Kubernetes 集群的状态指标，如 Pod 状态、ReplicaSet、Deployment 等。
  - **用途**：提供 Kubernetes 对象的状态信息，帮助运维人员管理集群。

  ### 12. **Alertmanager**

  - **功能**：尽管不直接用于数据收集，Alertmanager 用于处理 Prometheus 的告警，支持告警分组和通知。
  - **用途**：帮助团队管理告警，确保及时响应。

****

