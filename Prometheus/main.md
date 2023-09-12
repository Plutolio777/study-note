# Prometheus

## 安装配置

### 下载并解压

~~~shell

// 下载
wget -c https://github.com/prometheus/prometheus/releases/download/v2.28.1/prometheus-2.28.1.linux-amd64.tar.gz


// 移动
prometheus-2.28.1.linux-amd64 /usr/local/prometheus

~~~

### 配置系统服务

~~~shell
// 编辑文件
vim /usr/lib/systemd/system/prometheus.service


// 添加以下内容
[Unit]
Description=https://prometheus.io
  
[Service]
Restart=on-failure
ExecStart=/root/prometheus/prometheus --config.file=/root/prometheus/prometheus.yml
 
[Install]                      
WantedBy=multi-user.target

// 重置启动服务

systemctl daemon-reload
systemctl start prometheus.service
~~~

验证，浏览器输入： `http://<ip>:9090/`，出现如下界面说明安装成功了。

### node_exporter

prometheus只是监控数据，那么数据的来源呢，由XXX_exporter进行收集，如果是监控linux系统，那么就要安装node_exporter。

下载node_exporter，下载地址： `https://github.com/prometheus/node_exporter/releases`

~~~shell

wget https://github.com/prometheus/node_exporter/releases/download/v1.1.2/node_exporter-1.1.2.linux-amd64.tar.gz

# cat /etc/systemd/system/node_exporter.service
[Unit]
Description=node-exporter

[Service]
Type=simple
Restart=on-failure
RestartSec=5
ExecStart=/usr/local/node_exporter-1.1.2.linux-amd64/node_exporter

[Install]
WantedBy=multi-user.target


systemctl daemon-reload
systemctl start node_exporter
systemctl status node_exporter
systemctl enable node_exporter

~~~

验证，浏览器输入： `http://<ip>:9100/`

### 添加数据源

将node_exporter与prometheus建立联系，在prometheus.yml最后添加如下内容：

~~~yaml
- job_name: 'linux'
    static_configs:
    - targets: ['192.168.0.207:9100']
~~~

可以使用./promtool check config prometheus.yml对yml文件格式进行校验。`http://192.168.0.207:9090/targets` 查看监听的目标。

### grafana的安装

prometheus只是监控，而grafana对这些数据进行图形化的展示，下面采用yum进行安装。官方参考教程: `https://grafana.com/docs/grafana/latest/installation/rpm/`

#### 配置yum源

~~~shell
# cat /etc/yum.repos.d/grafana.repo 
[grafana]
name=grafana
baseurl=https://packages.grafana.com/oss/rpm
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://packages.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
~~~

#### 安装

~~~shell
yum update
yum makecache
yum install -y grafana
~~~

#### 设置系统服务

~~~shell
# cat /etc/systemd/system/grafana-server.service
systemctl daemon-reload
systemctl start grafana-server
systemctl status grafana-server
systemctl enable grafana-server
~~~

验证，浏览器输入 `http://<ip>:3000/`，默认密码admin/admin

### grafana中添加数据源

在 `http://<ip>:3000/datasources/new`添加数据源，选择prometheus。输入prometheus的地址 `http://<ip>:9090/` 保存即可，其余都默认。导入dashboard，打开 `http://<ip>:3000/dashboard/import`，可以在 `https://grafana.com/grafana/dashboards`寻找合适的模板，复制模板ID直接导入即可。

### redis_exporter监控多实例方案

#### 下载配置redis_exporter

官网下载 `https://github.com/oliver006/redis_exporter`

~~~shell
wget https://github.com/oliver006/redis_exporter/releases/download/v1.3.5/redis_exporter-v1.3.5.linux-amd64.tar.gz   
~~~

#### 单实例监

~~~shell
nohup ./redis_exporter -redis.addr 172.18.11.138:6379 -redis.password xxxxx &
~~~

prometheus配置单实例

~~~yaml
Prometheus 添加单实例
  - job_name: redis_since
    static_configs:
    - targets: ['172.18.11.138:9121']
~~~

#### 多实例监控

官方提供模板 多实例监控实际上需要将多个实例在prometheus中进行配置，以下是官方的配置

~~~yaml
scrape_configs:
  ## config for the multiple Redis targets that the exporter will scrape
  - job_name: 'redis_exporter_targets'
    static_configs:
      - targets: // 需要监控的多实例列表
        - redis://first-redis-host:6379
        - redis://second-redis-host:6379
        - redis://second-redis-host:6380
        - redis://second-redis-host:6381
    metrics_path: /scrape
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: <<REDIS-EXPORTER-HOSTNAME>>:9121 //exporter 地址
  
  ## config for scraping the exporter itself
  - job_name: 'redis_exporter'
    static_configs:
      - targets:
        - <<REDIS-EXPORTER-HOSTNAME>>:9121  //exporter 地址
~~~

此时只需要启动exporter即可无需填写 `redis.addr` 如果有密码则需要在命令行添加`redis.password` 注意如果需要这样配置应保证所有实例都使用相同的密码。

此时只需要通过命令行启动redis_exporter即可

~~~bash
nohup ./redis_exporter -redis.password xxxxx  &
~~~
