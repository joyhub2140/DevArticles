# Prometheus实战

## 安装

版本选择：

* ## prometheus

[prometheus-2.24.1.darwin-amd64.tar.gz](https://github.com/prometheus/prometheus/releases/download/v2.24.1/prometheus-2.24.1.darwin-amd64.tar.gz)

[prometheus-2.24.1.linux-amd64.tar.gz](https://github.com/prometheus/prometheus/releases/download/v2.24.1/prometheus-2.24.1.linux-amd64.tar.gz)

[prometheus-2.24.1.windows-amd64.zip](https://github.com/prometheus/prometheus/releases/download/v2.24.1/prometheus-2.24.1.windows-amd64.zip)

* ## alertmanager

[alertmanager-0.21.0.darwin-amd64.tar.gz](https://github.com/prometheus/alertmanager/releases/download/v0.21.0/alertmanager-0.21.0.darwin-amd64.tar.gz)

[alertmanager-0.21.0.linux-amd64.tar.gz](https://github.com/prometheus/alertmanager/releases/download/v0.21.0/alertmanager-0.21.0.linux-amd64.tar.gz)

[alertmanager-0.21.0.windows-amd64.tar.gz](https://github.com/prometheus/alertmanager/releases/download/v0.21.0/alertmanager-0.21.0.windows-amd64.tar.gz)

* ## blackbox_exporter

[blackbox_exporter-0.18.0.darwin-amd64.tar.gz](https://github.com/prometheus/blackbox_exporter/releases/download/v0.18.0/blackbox_exporter-0.18.0.darwin-amd64.tar.gz)

[blackbox_exporter-0.18.0.linux-amd64.tar.gz](https://github.com/prometheus/blackbox_exporter/releases/download/v0.18.0/blackbox_exporter-0.18.0.linux-amd64.tar.gz)

[blackbox_exporter-0.18.0.windows-amd64.tar.gz](https://github.com/prometheus/blackbox_exporter/releases/download/v0.18.0/blackbox_exporter-0.18.0.windows-amd64.tar.gz)

* ## node_exporter

[node_exporter-1.0.1.darwin-amd64.tar.gz](https://github.com/prometheus/node_exporter/releases/download/v1.0.1/node_exporter-1.0.1.darwin-amd64.tar.gz)

[node_exporter-1.0.1.linux-amd64.tar.gz](https://github.com/prometheus/node_exporter/releases/download/v1.0.1/node_exporter-1.0.1.linux-amd64.tar.gz)

[windows_exporter-0.15.0-amd64.exe](https://github.com/prometheus-community/windows_exporter/releases/download/v0.15.0/windows_exporter-0.15.0-amd64.exe)

### 安装prometheus

```bash
./prometheus --config.file=prometheus.yml --web.enable-lifecycle &
```

浏览器访问`http://localhost:9090/`

```
--config.file

--web.listen-address 

--web.external-url

--web.enable-lifecycle

--web.enable-admin-api

-alertmanager.url
```

prometheus.yml

```yml
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - localhost:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  
  # 监控prometheus自身
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']

	# 监控node1实例
  - job_name: 'node1'
    static_configs:
    - targets: ['localhost:9100']
      labels:
        instance: 'node1'
	# 监控node2实例
  - job_name: 'node2'
    params:
    	# 过滤收集器
      collect[]:
        - cpu
        - meminfo
        - diskstats
        - netstat
        - filefd
        - filesystem
        - xfs
        - systemd
    static_configs:
    - targets: ['192.168.223.2:9100']
      labels:
        instance: 'node2'
```

### 校验配置文件

```bash
./promtool check config prometheus.yml
Checking prometheus.yml
  SUCCESS: 2 rule files found

Checking alerts/memory_over.yml
  SUCCESS: 1 rules found

Checking alerts/server_down.yml
  SUCCESS: 1 rules found
```

### 告警规则文件

alerts/memory_over.yml

```yaml
groups:
  - name: memory_over
    rules:
      - alert: NodeMemoryUsage
        expr: (node_memory_MemTotal_bytes - (node_memory_MemFree_bytes+node_memory_Buffers_bytes+node_memory_Cached_bytes )) / node_memory_MemTotal_bytes * 100 > 80
        for: 20s
        labels:
          user: swfeng
        annotations:
          summary: "{{$labels.instance}}: High Memory usage detected"
          description: "{{$labels.instance}}: Memory usage is above 80% (current value is:{{ $value }})"
```

alerts/server_down.yml

```yaml
groups:
  - name: server_down
    rules:
      - alert: InstanceDown
        expr: up == 0
        for: 20s
        labels:
          user: swfeng
        annotations:
          summary: "Instance {{ $labels.instance }} down"
          description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 20 s."
```

### 配置文件热加载

当`prometheus`启动时开启`--web.enable-lifecycle`配置项后，当`prometheus`主配置文件发生修改后可发送如下的`POST`请求实现`prometheus`的热加载。

```bash
curl -X POST 'http://localhost:9090/-/reload'
```

配置生效后会在`prometheus`的控制台输出`Loading configuration file`信息。若配置文件有误则不会生效

```
level=info ts=2021-02-02T03:31:11.811Z caller=main.go:887 msg="Loading configuration file" filename=prometheus.yml
level=info ts=2021-02-02T03:31:16.320Z caller=main.go:918 msg="Completed loading of configuration file" filename=prometheus.yml totalDuration=4.508586935s remote_storage=2.961µs web_handler=656ns query_engine=2.136µs scrape=4.505752137s scrape_sd=90.392µs notify=21.123µs notify_sd=36.438µs rules=2.408µs
```

### 过滤收集器

```yaml
scrape_configs:
...
- job_name: 'node'
  static_configs:
    - targets: ['192.168.27.136:9100', '192.168.27.138:9100', '192.168.27.139:9100']
  params:
    collect[]:
      - cpu
      - meminfo
      - diskstats
      - netdev
      - netstat
      - filefd
      - filesystem
      - xfs
      - systemd
```

使用`params`块中`collect[]`列表指定，然后将它们作为URL参数传递给抓取请求。你可以使用Node Exporter实例上的curl命令来对此进行测试（只收集`cpu`指标，其它指标忽略）

```bash
curl -g -X GET 'http://192.168.223.2:9100/metrics?collect[]=cpu'
```

### 安装node_exporter

下载对应操作系统的`node_exporter`，解压直接运行即可

```bash
./node_exporter &
```

默认监听端口为`9100`，打开浏览器访问`http://IP:9100/metrics`即可看到监控指标

根据物理主机的不同，具体的监控指标也有差异：

- node_boot_time_seconds：系统启动时间
- node_cpu_*：系统CPU使用量
- node_disk_*：磁盘IO
- node_filesystem_*：文件系统用量
- node_load1：系统负载
- node_memeory_*：内存使用量
- node_network_*：网络带宽
- node_time：当前系统时间
- go_*：node exporter中go相关指标
- process_*：node exporter自身进程相关运行指标



### Management API

* 健康检查

```
GET http://localhost:9090/-/healthy
```

* 准备就绪检查

```
GET http://localhost:9090/-/ready
```

* 热加载

```
PUT  http://localhost:9090/-/reload
POST http://localhost:9090/-/reload
```

会向`Prometheus`进程发送`SIGTERM`信号，需要开启`--web.enable-lifecycle` 选项。

* 退出

```
PUT  http://localhost:9090/-/quit
POST http://localhost:9090/-/quit
```

优雅停机，会向`Prometheus`进程发送`SIGTERM`信号。需要开启`--web.enable-lifecycle` 选项。

# Alertmanager

alertmanager.yml

```yaml
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.exmail.qq.com:465'
  smtp_from: 'fengj@anchnet.com'
  smtp_auth_username: 'fengj@anchnet.com'
  smtp_auth_password: '******'
  smtp_require_tls: false

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'mail-receiver'

receivers:
- name: 'mail-receiver'
  email_configs:
    - to: 'fengj@anchnet.com'
      send_resolved: true
```

### 校验配置文件

```bash
./amtool check-config alertmanager.yml
Checking 'alertmanager.yml'  SUCCESS
Found:
 - global config
 - route
 - 0 inhibit rules
 - 1 receivers
 - 0 templates
```

### 安装Alertmanager

```bash
./alertmanager --config.file=alertmanager.yml
```

浏览器访问`http://localhost:9093`

### Management API

* 健康检查

```
GET http://localhost:9093/-/healthy
```

* 准备就绪检查

```
GET http://localhost:9093/-/ready
```

* 热加载

```
POST http://localhost:9093/-/reload
alertmanager           | level=info ts=2021-02-04T02:26:59.032Z caller=coordinator.go:119 component=configuration msg="Loading configuration file" file=/etc/alertmanager/alertmanager.yml
alertmanager           | level=info ts=2021-02-04T02:26:59.036Z caller=coordinator.go:131 component=configuration msg="Completed loading of configuration file" file=/etc/alertmanager/alertmanager.yml
```

```bash
curl -X POST 'http://localhost:9093/-/reload'
```

会向`Alertmanager`进程发送`SIGTERM`信号。



# Prometheus远程存储

https://prometheus.io/docs/operating/integrations/#remote-endpoints-and-storage

https://github.com/timescale/promscale/blob/master/docs/docker.md

https://github.com/timescale/promscale/blob/master/docker-compose/high-availability/docker-compose.yml

### [PostgreSQL/TimescaleDB](https://github.com/timescale/promscale)

* Docker方式:

```bash
# 创建网络
docker network create --driver bridge promscale-timescaledb
# 运行 TimescaleDB
docker run --name timescaledb -e POSTGRES_PASSWORD=<password> -d -p 5432:5432 --network promscale-timescaledb timescaledev/promscale-extension:latest-pg12 postgres -csynchronous_commit=off
# 运行 Promscale
docker run --name promscale -d -p 9201:9201 --network promscale-timescaledb timescale/promscale:<version-tag> -db-password=<password> -db-port=5432 -db-name=postgres -db-host=timescaledb -db-ssl-mode=allow
```

完整的`docker-compose.yml`文件如下:

```yaml
version: '3'

services:

  db:
    image: timescaledev/promscale-extension:latest-pg12
    ports:
      - 5432:5432/tcp
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_USER: postgres
      POSTGRES_DB: timescale

  prometheus:
    image: prom/prometheus:v2.24.1
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./alerts/memory_over.yml:/etc/prometheus/alerts/memory_over.yml:ro
      - ./alerts/server_down.yml:/etc/prometheus/alerts/server_down.yml:ro
    ports:
      - 9090:9090
    restart: always

  promscale-connector:
    image: timescale/promscale:latest
    ports:
      - 9201:9201/tcp
    restart: on-failure
    depends_on:
      - db
      - prometheus
    environment:
      PROMSCALE_LOG_LEVEL: debug
      PROMSCALE_DB_CONNECT_RETRIES: 10
      PROMSCALE_DB_HOST: db
      PROMSCALE_DB_PASSWORD: postgres
      PROMSCALE_WEB_TELEMETRY_PATH: /metrics-text
      PROMSCALE_DB_SSL_MODE: allow

  alertmanager:
    image: prom/alertmanager:v0.21.0
    container_name: alertmanager
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
    ports:
      - 9093:9093
    restart: always

  grafana:
    image: grafana/grafana
    container_name: grafana
    ports:
      - 3000:3000
    restart: always

networks:
  default:
    driver: bridge
```

* 二进制方式:

[promscale_0.1.4_Darwin_x86_64](https://github.com/timescale/promscale/releases/download/0.1.4/promscale_0.1.4_Darwin_x86_64)

[promscale_0.1.4_Linux_x86_64](https://github.com/timescale/promscale/releases/download/0.1.4/promscale_0.1.4_Linux_x86_64)

下载对应平台的二进制文件。

```bash
chmod +x promscale
./promscale --db-name <DBNAME> --db-password <DB-Password> --db-ssl-mode allow
```

在`prometheus.yml`配置文件中添加如下配置

```yam
remote_write:
  - url: "http://<connector-address>:9201/write"
remote_read:
  - url: "http://<connector-address>:9201/read"
```

### [InfluxDB](https://docs.influxdata.com/influxdb/v1.8/supported_protocols/prometheus)

Docker方式:

```bash
docker run --name influxdb -p 8086:8086 influxdb:1.8.4
```

docker-compose.yml

```yaml
version: '3'

services:

  influxdb:
    image: influxdb:1.8.4
    container_name: influxdb
    ports:
      - "8086:8086"
    environment:
      - INFLUXDB_DB=prometheus
      - INFLUXDB_ADMIN_ENABLED=true
      - INFLUXDB_ADMIN_USER=admin
      - INFLUXDB_ADMIN_PASSWORD=admin
      - INFLUXDB_USER=influxdb
      - INFLUXDB_USER_PASSWORD=influxdb

  chronograf:
    image: chronograf:1.8.8
    container_name: chronograf
    ports:
      - "8888:8888"
    environment:
      - INFLUXDB-URL=http://influxdb:8086

  prometheus:
    image: prom/prometheus:v2.24.1
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./alerts/memory_over.yml:/etc/prometheus/alerts/memory_over.yml:ro
      - ./alerts/server_down.yml:/etc/prometheus/alerts/server_down.yml:ro
    ports:
      - 9090:9090
    restart: always

  alertmanager:
    image: prom/alertmanager:v0.21.0
    container_name: alertmanager
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
    ports:
      - 9093:9093
    restart: always

  grafana:
    image: grafana/grafana
    container_name: grafana
    ports:
      - 3000:3000
    restart: always

networks:
  default:
    driver: bridge
```

在`prometheus.yml`配置文件中添加如下配置

```yam
remote_write:
  - url: "http://influxdb:8086/api/v1/prom/write?db=prometheus&u=influxdb&p=influxdb"

remote_read:
  - url: "http://influxdb:8086/api/v1/prom/read?db=prometheus&u=influxdb&p=influxdb"
```

查看influxdb中的数据

```bash
❯ docker exec -it influxdb /bin/bash
root@bd004204de23:/# influx
Connected to http://localhost:8086 version 1.8.4
InfluxDB shell version: 1.8.4
> help
Usage:
        connect <host:port>   connects to another node specified by host:port
        auth                  prompts for username and password
        pretty                toggles pretty print for the json format
        chunked               turns on chunked responses from server
        chunk size <size>     sets the size of the chunked responses.  Set to 0 to reset to the default chunked size
        use <db_name>         sets current database
        format <format>       specifies the format of the server responses: json, csv, or column
        precision <format>    specifies the format of the timestamp: rfc3339, h, m, s, ms, u or ns
        consistency <level>   sets write consistency level: any, one, quorum, or all
        history               displays command history
        settings              outputs the current settings for the shell
        clear                 clears settings such as database or retention policy.  run 'clear' for help
        exit/quit/ctrl+d      quits the influx shell

        show databases        show database names
        show series           show series information
        show measurements     show measurement information
        show tag keys         show tag key information
        show field keys       show field key information

        A full list of influxql commands can be found at:
        https://docs.influxdata.com/influxdb/latest/query_language/spec/
> show databases
name: databases
name
----
prometheus
_internal
> use prometheus
Using database prometheus
> show measurements
name: measurements
name
----
ALERTS
ALERTS_FOR_STATE
go_gc_duration_seconds
go_gc_duration_seconds_count
go_gc_duration_seconds_sum
go_goroutines
go_info
go_memstats_alloc_bytes
go_memstats_alloc_bytes_total
go_memstats_buck_hash_sys_bytes
go_memstats_frees_total
go_memstats_gc_cpu_fraction
go_memstats_gc_sys_bytes
go_memstats_heap_alloc_bytes
go_memstats_heap_idle_bytes
go_memstats_heap_inuse_bytes
go_memstats_heap_objects
go_memstats_heap_released_bytes
go_memstats_heap_sys_bytes
go_memstats_last_gc_time_seconds
go_memstats_lookups_total
go_memstats_mallocs_total
go_memstats_mcache_inuse_bytes
go_memstats_mcache_sys_bytes
go_memstats_mspan_inuse_bytes
go_memstats_mspan_sys_bytes
go_memstats_next_gc_bytes
go_memstats_other_sys_bytes
go_memstats_stack_inuse_bytes
go_memstats_stack_sys_bytes
go_memstats_sys_bytes
go_threads
net_conntrack_dialer_conn_attempted_total
net_conntrack_dialer_conn_closed_total
net_conntrack_dialer_conn_established_total
net_conntrack_dialer_conn_failed_total
net_conntrack_listener_conn_accepted_total
net_conntrack_listener_conn_closed_total
node_arp_entries
node_boot_time_seconds
node_context_switches_total
node_cpu_guest_seconds_total
node_cpu_seconds_total
node_disk_io_now
node_disk_io_time_seconds_total
node_disk_io_time_weighted_seconds_total
node_disk_read_bytes_total
node_disk_read_errors_total
node_disk_read_retries_total
node_disk_read_sectors_total
node_disk_read_time_seconds_total
node_disk_reads_completed_total
node_disk_reads_merged_total
node_disk_write_errors_total
node_disk_write_retries_total
node_disk_write_time_seconds_total
node_disk_writes_completed_total
node_disk_writes_merged_total
node_disk_written_bytes_total
node_disk_written_sectors_total
node_entropy_available_bits
node_exporter_build_info
node_filefd_allocated
node_filefd_maximum
node_filesystem_avail_bytes
node_filesystem_device_error
node_filesystem_files
node_filesystem_files_free
node_filesystem_free_bytes
node_filesystem_readonly
node_filesystem_size_bytes
node_forks_total
node_intr_total
node_load1
node_load15
node_load5
node_memory_Active_anon_bytes
node_memory_Active_bytes
node_memory_Active_file_bytes
node_memory_AnonHugePages_bytes
node_memory_AnonPages_bytes
node_memory_Bounce_bytes
node_memory_Buffers_bytes
node_memory_Cached_bytes
node_memory_CmaFree_bytes
node_memory_CmaTotal_bytes
node_memory_CommitLimit_bytes
node_memory_Committed_AS_bytes
node_memory_DirectMap2M_bytes
node_memory_DirectMap4k_bytes
node_memory_Dirty_bytes
node_memory_HardwareCorrupted_bytes
node_memory_HugePages_Free
node_memory_HugePages_Rsvd
node_memory_HugePages_Surp
node_memory_HugePages_Total
node_memory_Hugepagesize_bytes
node_memory_Inactive_anon_bytes
node_memory_Inactive_bytes
node_memory_Inactive_file_bytes
node_memory_KernelStack_bytes
node_memory_Mapped_bytes
node_memory_MemAvailable_bytes
node_memory_MemFree_bytes
node_memory_MemTotal_bytes
node_memory_Mlocked_bytes
node_memory_NFS_Unstable_bytes
node_memory_PageTables_bytes
node_memory_SReclaimable_bytes
node_memory_SUnreclaim_bytes
node_memory_Shmem_bytes
node_memory_Slab_bytes
node_memory_SwapCached_bytes
node_memory_SwapFree_bytes
node_memory_SwapTotal_bytes
node_memory_Unevictable_bytes
node_memory_VmallocChunk_bytes
node_memory_VmallocTotal_bytes
node_memory_VmallocUsed_bytes
node_memory_WritebackTmp_bytes
node_memory_Writeback_bytes
node_memory_active_bytes
node_memory_compressed_bytes
node_memory_free_bytes
node_memory_inactive_bytes
node_memory_swap_total_bytes
node_memory_swap_used_bytes
node_memory_swapped_in_bytes_total
node_memory_swapped_out_bytes_total
node_memory_total_bytes
node_memory_wired_bytes
node_netstat_Icmp6_InErrors
node_netstat_Icmp6_InMsgs
node_netstat_Icmp6_OutMsgs
node_netstat_Icmp_InErrors
node_netstat_Icmp_InMsgs
node_netstat_Icmp_OutMsgs
node_netstat_Ip6_InOctets
node_netstat_Ip6_OutOctets
node_netstat_IpExt_InOctets
node_netstat_IpExt_OutOctets
node_netstat_Ip_Forwarding
node_netstat_TcpExt_ListenDrops
node_netstat_TcpExt_ListenOverflows
node_netstat_TcpExt_SyncookiesFailed
node_netstat_TcpExt_SyncookiesRecv
node_netstat_TcpExt_SyncookiesSent
node_netstat_Tcp_ActiveOpens
node_netstat_Tcp_CurrEstab
node_netstat_Tcp_InErrs
node_netstat_Tcp_PassiveOpens
node_netstat_Tcp_RetransSegs
node_netstat_Udp6_InDatagrams
node_netstat_Udp6_InErrors
node_netstat_Udp6_NoPorts
node_netstat_Udp6_OutDatagrams
node_netstat_UdpLite6_InErrors
node_netstat_UdpLite_InErrors
node_netstat_Udp_InDatagrams
node_netstat_Udp_InErrors
node_netstat_Udp_NoPorts
node_netstat_Udp_OutDatagrams
node_network_address_assign_type
node_network_carrier
node_network_carrier_changes_total
node_network_device_id
node_network_dormant
node_network_flags
node_network_iface_id
node_network_iface_link
node_network_iface_link_mode
node_network_mtu_bytes
node_network_name_assign_type
node_network_net_dev_group
node_network_protocol_type
node_network_receive_bytes_total
node_network_receive_compressed_total
node_network_receive_drop_total
node_network_receive_errs_total
node_network_receive_fifo_total
node_network_receive_frame_total
node_network_receive_multicast_total
node_network_receive_packets_total
node_network_speed_bytes
node_network_transmit_bytes_total
node_network_transmit_carrier_total
node_network_transmit_colls_total
node_network_transmit_compressed_total
node_network_transmit_drop_total
node_network_transmit_errs_total
node_network_transmit_fifo_total
node_network_transmit_multicast_total
node_network_transmit_packets_total
node_network_transmit_queue_length
node_network_up
node_nf_conntrack_entries
node_nf_conntrack_entries_limit
node_procs_blocked
node_procs_running
node_scrape_collector_duration_seconds
node_scrape_collector_success
node_sockstat_FRAG_inuse
node_sockstat_FRAG_memory
node_sockstat_RAW_inuse
node_sockstat_TCP_alloc
node_sockstat_TCP_inuse
node_sockstat_TCP_mem
node_sockstat_TCP_mem_bytes
node_sockstat_TCP_orphan
node_sockstat_TCP_tw
node_sockstat_UDPLITE_inuse
node_sockstat_UDP_inuse
node_sockstat_UDP_mem
node_sockstat_UDP_mem_bytes
node_sockstat_sockets_used
node_textfile_scrape_error
node_time_seconds
node_timex_estimated_error_seconds
node_timex_frequency_adjustment_ratio
node_timex_loop_time_constant
node_timex_maxerror_seconds
node_timex_offset_seconds
node_timex_pps_calibration_total
node_timex_pps_error_total
node_timex_pps_frequency_hertz
node_timex_pps_jitter_seconds
node_timex_pps_jitter_total
node_timex_pps_shift_seconds
node_timex_pps_stability_exceeded_total
node_timex_pps_stability_hertz
node_timex_status
node_timex_sync_status
node_timex_tai_offset_seconds
node_timex_tick_seconds
node_uname_info
node_vmstat_pgfault
node_vmstat_pgmajfault
node_vmstat_pgpgin
node_vmstat_pgpgout
node_vmstat_pswpin
node_vmstat_pswpout
process_cpu_seconds_total
process_max_fds
process_open_fds
process_resident_memory_bytes
process_start_time_seconds
process_virtual_memory_bytes
process_virtual_memory_max_bytes
prometheus_api_remote_read_queries
prometheus_build_info
prometheus_config_last_reload_success_timestamp_seconds
prometheus_config_last_reload_successful
prometheus_engine_queries
prometheus_engine_queries_concurrent_max
prometheus_engine_query_duration_seconds
prometheus_engine_query_duration_seconds_count
prometheus_engine_query_duration_seconds_sum
prometheus_engine_query_log_enabled
prometheus_engine_query_log_failures_total
prometheus_http_request_duration_seconds_bucket
prometheus_http_request_duration_seconds_count
prometheus_http_request_duration_seconds_sum
prometheus_http_requests_total
prometheus_http_response_size_bytes_bucket
prometheus_http_response_size_bytes_count
prometheus_http_response_size_bytes_sum
prometheus_notifications_alertmanagers_discovered
prometheus_notifications_dropped_total
prometheus_notifications_errors_total
prometheus_notifications_latency_seconds
prometheus_notifications_latency_seconds_count
prometheus_notifications_latency_seconds_sum
prometheus_notifications_queue_capacity
prometheus_notifications_queue_length
prometheus_notifications_sent_total
prometheus_remote_storage_enqueue_retries_total
prometheus_remote_storage_highest_timestamp_in_seconds
prometheus_remote_storage_max_samples_per_send
prometheus_remote_storage_metadata_bytes_total
prometheus_remote_storage_metadata_failed_total
prometheus_remote_storage_metadata_retried_total
prometheus_remote_storage_metadata_total
prometheus_remote_storage_queue_highest_sent_timestamp_seconds
prometheus_remote_storage_read_queries_total
prometheus_remote_storage_read_request_duration_seconds_bucket
prometheus_remote_storage_read_request_duration_seconds_count
prometheus_remote_storage_read_request_duration_seconds_sum
prometheus_remote_storage_remote_read_queries
prometheus_remote_storage_samples_bytes_total
prometheus_remote_storage_samples_dropped_total
prometheus_remote_storage_samples_failed_total
prometheus_remote_storage_samples_in_total
prometheus_remote_storage_samples_pending
prometheus_remote_storage_samples_retried_total
prometheus_remote_storage_samples_total
prometheus_remote_storage_sent_batch_duration_seconds_bucket
prometheus_remote_storage_sent_batch_duration_seconds_count
prometheus_remote_storage_sent_batch_duration_seconds_sum
prometheus_remote_storage_shard_capacity
prometheus_remote_storage_shards
prometheus_remote_storage_shards_desired
prometheus_remote_storage_shards_max
prometheus_remote_storage_shards_min
prometheus_remote_storage_string_interner_zero_reference_releases_total
prometheus_rule_evaluation_duration_seconds
prometheus_rule_evaluation_duration_seconds_count
prometheus_rule_evaluation_duration_seconds_sum
prometheus_rule_evaluation_failures_total
prometheus_rule_evaluations_total
prometheus_rule_group_duration_seconds
prometheus_rule_group_duration_seconds_count
prometheus_rule_group_duration_seconds_sum
prometheus_rule_group_interval_seconds
prometheus_rule_group_iterations_missed_total
prometheus_rule_group_iterations_total
prometheus_rule_group_last_duration_seconds
prometheus_rule_group_last_evaluation_samples
prometheus_rule_group_last_evaluation_timestamp_seconds
prometheus_rule_group_rules
prometheus_sd_consul_rpc_duration_seconds_count
prometheus_sd_consul_rpc_duration_seconds_sum
prometheus_sd_consul_rpc_failures_total
prometheus_sd_discovered_targets
prometheus_sd_dns_lookup_failures_total
prometheus_sd_dns_lookups_total
prometheus_sd_failed_configs
prometheus_sd_file_read_errors_total
prometheus_sd_file_scan_duration_seconds_count
prometheus_sd_file_scan_duration_seconds_sum
prometheus_sd_kubernetes_events_total
prometheus_sd_received_updates_total
prometheus_sd_updates_total
prometheus_target_interval_length_seconds
prometheus_target_interval_length_seconds_count
prometheus_target_interval_length_seconds_sum
prometheus_target_metadata_cache_bytes
prometheus_target_metadata_cache_entries
prometheus_target_scrape_pool_exceeded_target_limit_total
prometheus_target_scrape_pool_reloads_failed_total
prometheus_target_scrape_pool_reloads_total
prometheus_target_scrape_pool_sync_total
prometheus_target_scrape_pool_targets
prometheus_target_scrape_pools_failed_total
prometheus_target_scrape_pools_total
prometheus_target_scrapes_cache_flush_forced_total
prometheus_target_scrapes_exceeded_sample_limit_total
prometheus_target_scrapes_sample_duplicate_timestamp_total
prometheus_target_scrapes_sample_out_of_bounds_total
prometheus_target_scrapes_sample_out_of_order_total
prometheus_target_sync_length_seconds
prometheus_target_sync_length_seconds_count
prometheus_target_sync_length_seconds_sum
prometheus_template_text_expansion_failures_total
prometheus_template_text_expansions_total
prometheus_treecache_watcher_goroutines
prometheus_treecache_zookeeper_failures_total
prometheus_tsdb_blocks_loaded
prometheus_tsdb_checkpoint_creations_failed_total
prometheus_tsdb_checkpoint_creations_total
prometheus_tsdb_checkpoint_deletions_failed_total
prometheus_tsdb_checkpoint_deletions_total
prometheus_tsdb_compaction_chunk_range_seconds_bucket
prometheus_tsdb_compaction_chunk_range_seconds_count
prometheus_tsdb_compaction_chunk_range_seconds_sum
prometheus_tsdb_compaction_chunk_samples_bucket
prometheus_tsdb_compaction_chunk_samples_count
prometheus_tsdb_compaction_chunk_samples_sum
prometheus_tsdb_compaction_chunk_size_bytes_bucket
prometheus_tsdb_compaction_chunk_size_bytes_count
prometheus_tsdb_compaction_chunk_size_bytes_sum
prometheus_tsdb_compaction_duration_seconds_bucket
prometheus_tsdb_compaction_duration_seconds_count
prometheus_tsdb_compaction_duration_seconds_sum
prometheus_tsdb_compaction_populating_block
prometheus_tsdb_compactions_failed_total
prometheus_tsdb_compactions_skipped_total
prometheus_tsdb_compactions_total
prometheus_tsdb_compactions_triggered_total
prometheus_tsdb_data_replay_duration_seconds
prometheus_tsdb_head_active_appenders
prometheus_tsdb_head_chunks
prometheus_tsdb_head_chunks_created_total
prometheus_tsdb_head_chunks_removed_total
prometheus_tsdb_head_gc_duration_seconds_count
prometheus_tsdb_head_gc_duration_seconds_sum
prometheus_tsdb_head_max_time
prometheus_tsdb_head_max_time_seconds
prometheus_tsdb_head_min_time
prometheus_tsdb_head_min_time_seconds
prometheus_tsdb_head_samples_appended_total
prometheus_tsdb_head_series
prometheus_tsdb_head_series_created_total
prometheus_tsdb_head_series_not_found_total
prometheus_tsdb_head_series_removed_total
prometheus_tsdb_head_truncations_failed_total
prometheus_tsdb_head_truncations_total
prometheus_tsdb_isolation_high_watermark
prometheus_tsdb_isolation_low_watermark
prometheus_tsdb_lowest_timestamp
prometheus_tsdb_lowest_timestamp_seconds
prometheus_tsdb_mmap_chunk_corruptions_total
prometheus_tsdb_out_of_bound_samples_total
prometheus_tsdb_out_of_order_samples_total
prometheus_tsdb_reloads_failures_total
prometheus_tsdb_reloads_total
prometheus_tsdb_retention_limit_bytes
prometheus_tsdb_size_retentions_total
prometheus_tsdb_storage_blocks_bytes
prometheus_tsdb_symbol_table_size_bytes
prometheus_tsdb_time_retentions_total
prometheus_tsdb_tombstone_cleanup_seconds_bucket
prometheus_tsdb_tombstone_cleanup_seconds_count
prometheus_tsdb_tombstone_cleanup_seconds_sum
prometheus_tsdb_vertical_compactions_total
prometheus_tsdb_wal_completed_pages_total
prometheus_tsdb_wal_corruptions_total
prometheus_tsdb_wal_fsync_duration_seconds_count
prometheus_tsdb_wal_fsync_duration_seconds_sum
prometheus_tsdb_wal_page_flushes_total
prometheus_tsdb_wal_segment_current
prometheus_tsdb_wal_truncate_duration_seconds_count
prometheus_tsdb_wal_truncate_duration_seconds_sum
prometheus_tsdb_wal_truncations_failed_total
prometheus_tsdb_wal_truncations_total
prometheus_tsdb_wal_writes_failed_total
prometheus_wal_watcher_current_segment
prometheus_wal_watcher_record_decode_failures_total
prometheus_wal_watcher_records_read_total
prometheus_wal_watcher_samples_sent_pre_tailing_total
prometheus_web_federation_errors_total
prometheus_web_federation_warnings_total
promhttp_metric_handler_errors_total
promhttp_metric_handler_requests_in_flight
promhttp_metric_handler_requests_total
scrape_duration_seconds
scrape_samples_post_metric_relabeling
scrape_samples_scraped
scrape_series_added
up
> select * from node_load1
name: node_load1
time                __name__   instance job   value
----                --------   -------- ---   -----
1612341680647000000 node_load1 node1    node1 2.44482421875
1612341683852000000 node_load1 node2    node2 0.77
1612341695647000000 node_load1 node1    node1 2.70947265625
1612341698852000000 node_load1 node2    node2 0.6
1612341710647000000 node_load1 node1    node1 2.47802734375
1612341713852000000 node_load1 node2    node2 0.47
1612341725647000000 node_load1 node1    node1 2.291015625
1612341728852000000 node_load1 node2    node2 0.43
1612341740647000000 node_load1 node1    node1 2.1337890625
1612341743852000000 node_load1 node2    node2 0.41
1612341755647000000 node_load1 node1    node1 2.63330078125
1612341758852000000 node_load1 node2    node2 0.4
1612341770647000000 node_load1 node1    node1 2.77587890625
1612341773852000000 node_load1 node2    node2 0.31
1612341785647000000 node_load1 node1    node1 2.7021484375
1612341788852000000 node_load1 node2    node2 0.24
1612341800647000000 node_load1 node1    node1 2.6875
1612341803857000000 node_load1 node2    node2 0.19
1612341815647000000 node_load1 node1    node1 3.0458984375
1612341818852000000 node_load1 node2    node2 0.14
1612341830654000000 node_load1 node1    node1 3.2626953125
1612341833852000000 node_load1 node2    node2 0.11
1612341845647000000 node_load1 node1    node1 3.19775390625
1612341848852000000 node_load1 node2    node2 0.17
1612341860647000000 node_load1 node1    node1 2.86962890625
1612341863852000000 node_load1 node2    node2 0.36
1612341875647000000 node_load1 node1    node1 2.744140625
1612341878852000000 node_load1 node2    node2 0.35
1612341890647000000 node_load1 node1    node1 2.78759765625
1612341893959000000 node_load1 node2    node2 0.27
1612341905647000000 node_load1 node1    node1 3.70849609375
1612341908852000000 node_load1 node2    node2 0.21
1612341920647000000 node_load1 node1    node1 3.40283203125
1612341923852000000 node_load1 node2    node2 0.16
1612341935647000000 node_load1 node1    node1 3.01708984375
1612341938852000000 node_load1 node2    node2 0.27
1612341950647000000 node_load1 node1    node1 6.3642578125
1612341953852000000 node_load1 node2    node2 0.21
1612341965647000000 node_load1 node1    node1 5.46484375
1612341968852000000 node_load1 node2    node2 0.16
1612341980657000000 node_load1 node1    node1 4.5673828125
1612341983852000000 node_load1 node2    node2 0.28
1612341995647000000 node_load1 node1    node1 4.0712890625
1612341998852000000 node_load1 node2    node2 0.35
1612342010647000000 node_load1 node1    node1 4.2333984375
1612342013852000000 node_load1 node2    node2 0.27
1612342025647000000 node_load1 node1    node1 4.14453125
1612342028852000000 node_load1 node2    node2 0.21
1612342040647000000 node_load1 node1    node1 5.02490234375
1612342043852000000 node_load1 node2    node2 0.16
1612342055647000000 node_load1 node1    node1 4.2802734375
1612342058852000000 node_load1 node2    node2 0.13
1612342070647000000 node_load1 node1    node1 3.6142578125
1612342073852000000 node_load1 node2    node2 0.74
1612342085647000000 node_load1 node1    node1 3.046875
1612342088852000000 node_load1 node2    node2 0.58
1612342100647000000 node_load1 node1    node1 2.69189453125
1612342103852000000 node_load1 node2    node2 0.69
1612342115647000000 node_load1 node1    node1 2.6845703125
1612342119043000000 node_load1 node2    node2 0.54
1612342130647000000 node_load1 node1    node1 2.60009765625
1612342133852000000 node_load1 node2    node2 0.49
1612342145647000000 node_load1 node1    node1 2.4599609375
1612342148852000000 node_load1 node2    node2 0.45
1612342160647000000 node_load1 node1    node1 2.19140625
1612342163852000000 node_load1 node2    node2 0.43
1612342175647000000 node_load1 node1    node1 1.97607421875
1612342178852000000 node_load1 node2    node2 0.33
1612342190647000000 node_load1 node1    node1 2.28173828125
1612342193852000000 node_load1 node2    node2 0.48
1612342205647000000 node_load1 node1    node1 2.0712890625
1612342208852000000 node_load1 node2    node2 0.53
1612342220647000000 node_load1 node1    node1 4.08837890625
1612342223852000000 node_load1 node2    node2 0.49
1612342235647000000 node_load1 node1    node1 3.6064453125
1612342238852000000 node_load1 node2    node2 0.53
1612342250647000000 node_load1 node1    node1 3.25
1612342253852000000 node_load1 node2    node2 0.41
1612342265647000000 node_load1 node1    node1 3.126953125
1612342268852000000 node_load1 node2    node2 0.59
>

```

# Prometheus服务发现

### 基于文件的服务发现

prometheus.yml

```yaml
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

remote_write:
  - url: "http://promscale-connector:9201/write"

remote_read:
  - url: "http://promscale-connector:9201/read"

# remote_write:
#   - url: "http://influxdb:8086/api/v1/prom/write?db=prometheus&u=influxdb&p=influxdb"

# remote_read:
#   - url: "http://influxdb:8086/api/v1/prom/read?db=prometheus&u=influxdb&p=influxdb"


# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"
  - "alerts/*.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['prometheus:9090']

  # - job_name: 'node1'
  #   static_configs:
  #   - targets: ['172.24.107.47:9100']
  #     labels:
  #       instance: 'node1'

  # - job_name: 'node2'
  #   static_configs:
  #   - targets: ['192.168.223.2:9100']
  #     labels:
  #       instance: 'node2'

  - job_name: 'file_ds'
    file_sd_configs:
    - files:
      - 'targets.json'
      refresh_interval: 1m
```

targets.json

```json
[
  {
    "targets": [
      "172.24.107.47:9100"
    ],
    "labels": {
      "instance": "node1",
      "job": "node1"
    }
  },
  {
    "targets": [
      "192.168.223.2:9100"
    ],
    "labels": {
      "instance": "node2",
      "job": "node2"
    }
  },
  {
    "targets": [
      "192.168.223.6:9100"
    ],
    "labels": {
      "instance": "node3",
      "job": "node3"
    }
  }
]
```

```bash
curl -X POST 'http://localhost:9090/-/reload'
```



# PromQL 常用查询语句

收集到 node_exporter 的数据后，我们可以使用 PromQL 进行一些业务查询和监控，下面是一些比较常见的查询。

注意：以下查询均以单个节点作为例子，如果大家想查看所有节点，将 `instance="xxx"` 去掉即可。

## 系统正常运行的时间

```
node_time_seconds{instance=~"node1",job=~"node1"} - node_boot_time_seconds{instance=~"node1",job=~"node1"}
```

node_time_seconds 当前系统时间

node_boot_time_seconds 系统启动时间

## CPU物理核数

```
count(count(node_cpu_seconds_total{}) by (cpu))
```

## CPU 使用率

```
100 - (avg by(instance) (irate(node_cpu_seconds_total{instance="node1", mode="idle"}[5m])) * 100)
```

## CPU 各 mode 占比率

```
avg by (instance, mode) (irate(node_cpu_seconds_total{instance="node1"}[5m])) * 100
```

## 机器平均负载

```
node_load1{instance="node1"} // 1分钟负载
node_load5{instance="node1"} // 5分钟负载
node_load15{instance="node1"} // 15分钟负载
```

## 内存使用率

node_memory_MemTotal_bytes：主机上的总内存
node_memory_MemFree_bytes：主机上的可用内存
node_memory_Buffers_bytes：缓冲缓存中的内存
node_memory_Cached_bytes：页面缓存中的内存

(总内存-(可用内存+缓冲缓存中的内存+页面缓存中的内存))÷总内存×100

```
(node_memory_MemTotal_bytes{instance="node2"} - (node_memory_MemFree_bytes{instance="node2"} + node_memory_Cached_bytes{instance="node2"} + node_memory_Buffers_bytes{instance="node2"})) / node_memory_MemTotal_bytes{instance="node2"} * 100
```

## 内存大小

```
node_memory_total_bytes{instance=~"node1",job=~"node1"}
```

## 交换分区大小

```
node_memory_swap_total_bytes{instance=~"node1",job=~"node1"}
```

## 磁盘总大小

```
node_filesystem_size_bytes{instance=~"node1",job=~"node1",mountpoint="/",fstype!="rootfs"}
```

## 磁盘使用率

```
100 - node_filesystem_free_bytes{instance="node1",fstype!~"rootfs|selinuxfs|autofs|rpc_pipefs|tmpfs|udev|none|devpts|sysfs|debugfs|fuse.*"} / node_filesystem_size_bytes{instance="node1",fstype!~"rootfs|selinuxfs|autofs|rpc_pipefs|tmpfs|udev|none|devpts|sysfs|debugfs|fuse.*"} * 100
```

或者你也可以直接使用 {fstype="xxx"} 来指定想查看的磁盘信息

## 网络 IO

```
// 上行带宽
sum by (instance) (irate(node_network_receive_bytes_total{instance="node1",device!~"bond.*?|lo"}[5m])/128)

// 下行带宽
sum by (instance) (irate(node_network_transmit_bytes_total{instance="node1",device!~"bond.*?|lo"}[5m])/128)
```

## 网卡出/入包

```
// 入包量
sum by (instance) (rate(node_network_receive_bytes_total{instance="node1",device!="lo"}[5m]))

// 出包量
sum by (instance) (rate(node_network_transmit_bytes_total{instance="node1",device!="lo"}[5m]))
```



# HTTP API中使用PromQL

## API响应格式

Prometheus API使用了JSON格式的响应内容。 当API调用成功后将会返回2xx的HTTP状态码。

反之，当API调用失败时可能返回以下几种不同的HTTP状态码：

- 404 Bad Request：当参数错误或者缺失时。
- 422 Unprocessable Entity 当表达式无法执行时。
- 503 Service Unavailiable 当请求超时或者被中断时。

所有的API请求均使用以下的JSON格式：

```json
{
  "status": "success" | "error",
  "data": <data>,

  // Only set if status is "error". The data field may still hold
  // additional data.
  "errorType": "<string>",
  "error": "<string>"
}
```

## 在HTTP API中使用PromQL

通过HTTP API我们可以分别通过`/api/v1/query`和`/api/v1/query_range`查询`PromQL`表达式当前或者一定时间范围内的计算结果。

### 瞬时数据查询

通过使用QUERY API我们可以查询PromQL在特定时间点下的计算结果。

```
GET /api/v1/query
```

URL请求参数：

- query=：PromQL表达式。
- time=<rfc3339 | unix_timestamp>：用于指定用于计算PromQL的时间戳。可选参数，默认情况下使用当前系统时间。
- timeout=：超时设置。可选参数，默认情况下使用-query,timeout的全局设置。

例如使用以下表达式查询表达式up在时间点2015-07-01T20:10:51.781Z的计算结果：

```
$ curl 'http://localhost:9090/api/v1/query?query=up&time=2015-07-01T20:10:51.781Z'
{
   "status" : "success",
   "data" : {
      "resultType" : "vector",
      "result" : [
         {
            "metric" : {
               "__name__" : "up",
               "job" : "prometheus",
               "instance" : "localhost:9090"
            },
            "value": [ 1435781451.781, "1" ]
         },
         {
            "metric" : {
               "__name__" : "up",
               "job" : "node",
               "instance" : "localhost:9100"
            },
            "value" : [ 1435781451.781, "0" ]
         }
      ]
   }
}
```

### 响应数据类型

当API调用成功后，Prometheus会返回JSON格式的响应内容，格式如上小节所示。并且在data节点中返回查询结果。data节点格式如下：

```
{
  "resultType": "matrix" | "vector" | "scalar" | "string",
  "result": <value>
}
```

PromQL表达式可能返回多种数据类型，在响应内容中使用resultType表示当前返回的数据类型，包括：

- 瞬时向量：vector

当返回数据类型resultType为vector时，result响应格式如下：

```
[
  {
    "metric": { "<label_name>": "<label_value>", ... },
    "value": [ <unix_time>, "<sample_value>" ]
  },
  ...
]
```

其中metrics表示当前时间序列的特征维度，value只包含一个唯一的样本。

- 区间向量：matrix

当返回数据类型resultType为matrix时，result响应格式如下：

```
[
  {
    "metric": { "<label_name>": "<label_value>", ... },
    "values": [ [ <unix_time>, "<sample_value>" ], ... ]
  },
  ...
]
```

其中metrics表示当前时间序列的特征维度，values包含当前事件序列的一组样本。

- 标量：scalar

当返回数据类型resultType为scalar时，result响应格式如下：

```
[ <unix_time>, "<scalar_value>" ]
```

由于标量不存在时间序列一说，因此result表示为当前系统时间一个标量的值。

- 字符串：string

当返回数据类型resultType为string时，result响应格式如下：

```
[ <unix_time>, "<string_value>" ]
```

字符串类型的响应内容格式和标量相同。

### 区间数据查询

使用QUERY_RANGE API我们则可以直接查询PromQL表达式在一段时间返回内的计算结果。

```
GET /api/v1/query_range
```

URL请求参数：

- query=: PromQL表达式。
- start=<rfc3339 | unix_timestamp>: 起始时间。
- end=<rfc3339 | unix_timestamp>: 结束时间。
- step=: 查询步长。
- timeout=: 超时设置。可选参数，默认情况下使用-query,timeout的全局设置。

当使用QUERY_RANGE API查询PromQL表达式时，返回结果一定是一个区间向量：

```
{
  "resultType": "matrix",
  "result": <value>
}
```

> 需要注意的是，在QUERY_RANGE API中PromQL只能使用瞬时向量选择器类型的表达式。

例如使用以下表达式查询表达式up在30秒范围内以15秒为间隔计算PromQL表达式的结果。

```json
$ curl 'http://localhost:9090/api/v1/query_range?query=up&start=2015-07-01T20:10:30.781Z&end=2015-07-01T20:11:00.781Z&step=15s'
{
   "status" : "success",
   "data" : {
      "resultType" : "matrix",
      "result" : [
         {
            "metric" : {
               "__name__" : "up",
               "job" : "prometheus",
               "instance" : "localhost:9090"
            },
            "values" : [
               [ 1435781430.781, "1" ],
               [ 1435781445.781, "1" ],
               [ 1435781460.781, "1" ]
            ]
         },
         {
            "metric" : {
               "__name__" : "up",
               "job" : "node",
               "instance" : "localhost:9091"
            },
            "values" : [
               [ 1435781430.781, "0" ],
               [ 1435781445.781, "0" ],
               [ 1435781460.781, "1" ]
            ]
         }
      ]
   }
}
```



# 附件

### prometheus.yml

```yaml
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - localhost:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"
  - "alerts/*.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']

  - job_name: 'node1'
    static_configs:
    - targets: ['localhost:9100']
      labels:
        instance: 'node1'

  - job_name: 'node2'
    static_configs:
    - targets: ['192.168.223.2:9100']
      labels:
        instance: 'node2'
```

校验:

```bash
./promtool check config prometheus.yml
Checking prometheus.yml
  SUCCESS: 2 rule files found

Checking alerts/memory_over.yml
  SUCCESS: 1 rules found

Checking alerts/server_down.yml
  SUCCESS: 1 rules found
```

### alerts/memory_over.yml

```yaml
groups:
  - name: memory_over
    rules:
      - alert: NodeMemoryUsage
        expr: (node_memory_MemTotal_bytes - (node_memory_MemFree_bytes+node_memory_Buffers_bytes+node_memory_Cached_bytes )) / node_memory_MemTotal_bytes * 100 > 80
        for: 20s
        labels:
          user: swfeng
        annotations:
          summary: "{{$labels.instance}}: High Memory usage detected"
          description: "{{$labels.instance}}: Memory usage is above 80% (current value is:{{ $value }})"
```

校验

```bash
./promtool check rules alerts/memory_over.yml
Checking alerts/memory_over.yml
  SUCCESS: 1 rules found
```

### alerts/server_down.yml

```yaml
groups:
  - name: server_down
    rules:
      - alert: InstanceDown
        expr: up == 0
        for: 20s
        labels:
          user: swfeng
        annotations:
          summary: "Instance {{ $labels.instance }} down"
          description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 20 s."
```

校验

```bash
./promtool check rules alerts/server_down.yml
Checking alerts/server_down.yml
  SUCCESS: 1 rules found
```

### alertmanager.yml

```yaml
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.exmail.qq.com:465'
  smtp_from: 'fengj@anchnet.com'
  smtp_auth_username: 'fengj@anchnet.com'
  smtp_auth_password: '******'
  smtp_require_tls: false

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'mail-receiver'

receivers:
- name: 'mail-receiver'
  email_configs:
    - to: 'fengj@anchnet.com'
      send_resolved: true
```

校验

```bash
./amtool check-config alertmanager.yml
Checking 'alertmanager.yml'  SUCCESS
Found:
 - global config
 - route
 - 0 inhibit rules
 - 1 receivers
 - 0 templates
```



# 参考文档

https://yunlzheng.gitbook.io/prometheus-book/

https://prometheus.io/docs/instrumenting/exporters/

https://prometheus.io/docs/operating/integrations/

https://prometheus.io/docs/prometheus/latest/querying/basics/

https://prometheus.io/docs/prometheus/latest/querying/operators/

https://prometheus.io/docs/prometheus/latest/querying/functions/

https://prometheus.io/docs/prometheus/latest/querying/examples/

https://prometheus.io/docs/prometheus/latest/querying/api/

https://github.com/timescale/promscale

https://grafana.com/grafana/dashboards/8919

https://grafana.com/grafana/dashboards/1860