# HAProxy Service Analytics (Loki Logs)

A comprehensive Grafana dashboard for analyzing HAProxy access logs stored in Loki.

This dashboard provides a service-centric view of all applications behind HAProxy, helping operations, DevOps, SRE, and development teams monitor traffic patterns, service health, latency, error rates, and user behavior from a single place.

**Features**

- Total request volume and real-time visitor statistics
- HTTP status code distribution and trends over time
- 5xx error rate monitoring
- Request distribution by domain/host
- Traffic volume and bytes transferred per application
- 95th percentile response time and maximum request latency
- Detection of slow and heavy requests
- Top requested pages and endpoints
- Top visitor IP addresses
- Top user agents and HTTP referrers
- Recent request stream for quick troubleshooting
- Use Cases
- Monitor application availability and service quality
- Detect traffic spikes and unusual request patterns
- Identify slow endpoints and performance bottlenecks
- Investigate HTTP errors and failed requests
- Analyze user behavior and traffic sources
- Track bandwidth consumption across hosted applications

The dashboard was inspired by the [Loki NGINX Service Mesh](https://grafana.com/grafana/dashboards/12559-loki-nginx-service-mesh-json-version/) dashboard and adapted specifically for HAProxy log formats and fields, with additional queries and improvements focused on real-world production environments.

## Installation

### 1) Enable HAProxy Access Logging

First, ensure that HAProxy is configured to write access logs through **rsyslog**.

Edit your `haproxy.cfg` and make sure the global section contains:

```
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
```
---
Configure **rsyslog** to write HAProxy logs to a dedicated log file.

Create or edit:
```
/etc/rsyslog.d/49-haproxy.conf
```
with the following content:
```
$AddUnixListenSocket /var/lib/haproxy/dev/log

$template OnlyMsg,"%msg:2:$%\n"

local0.* /var/log/haproxy.log;OnlyMsg

& stop
```
---
Create the log file:
```Shell
sudo touch /var/log/haproxy.log
sudo chown syslog:adm /var/log/haproxy.log
sudo chmod 640 /var/log/haproxy.log
```
Restart the services:
```Shell
sudo systemctl restart rsyslog
sudo systemctl restart haproxy
```

Verify that the file is receiving logs:
```Shell
tail -f /var/log/haproxy.log
```

### 2) Enable JSON Logging in HAProxy

This dashboard expects HAProxy logs in **JSON** format.

Add the following directives to your **frontend** section.

```
http-request capture req.hdr(Host) len 64
http-request capture req.hdr(Referer) len 128
http-request capture req.hdr(User-Agent) len 128
http-request capture req.hdr(X-Forwarded-For) len 64
http-request capture req.hdr(X-Real-IP) len 64

unique-id-format %[uuid()]
unique-id-header X-Unique-ID

http-request set-header X-Forwarded-For %[src]

log-format '{"time_local":"%t","request_id":"%ID","remote_addr":"%ci","remote_port":%cp,"frontend_name":"%f","backend_name":"%b","server_name":"%s","status":%ST,"bytes_sent":%B,"bytes_received":%U,"request_method":"%HM","request_uri":"%[capture.req.uri,json(utf8s)]","http_version":"%HV","host":"%[capture.req.hdr(0),json(utf8s)]","referer":"%[capture.req.hdr(1),json(utf8s)]","user_agent":"%[capture.req.hdr(2),json(utf8s)]","x_forwarded_for":"%[capture.req.hdr(3),json(utf8s)]","x-real-ip":"%[capture.req.hdr(4),json(utf8s)]","duration_total_ms":%Ta,"duration_wait_ms":%Tc,"duration_response_ms":%Tr,"termination_state":"%ts","actconn":%ac,"feconn":%fc,"beconn":%bc,"srvconn":%sc,"retries":%rc}'
```

Restart HAProxy after making the changes:

```Shell
sudo systemctl restart haproxy
```

### 3) Configure Promtail

If you are using **Promtail** to collect logs and send them to Loki, configure it as follows.

```
- job_name: haproxy
  static_configs:
    - targets:
        - localhost
      labels:
        job: haproxy
        instance: ${HOSTNAME}
        __path__: /var/log/haproxy.log

  pipeline_stages:

    - regex:
        expression: "^(?P<timestamp>\\w{3}\\s+\\d+\\s+\\d+:\\d+:\\d+)\\s+(?P<hostname>[\\w-]+)\\s+haproxy\\[(?P<pid>\\d+)\\]:\\s+(?P<json_payload>.*)$"

    - output:
        source: json_payload

    - json:
        source: json_payload
        expressions:
          status: status
          host: host
          backend: backend_name

    - labels:
        status:
        host:
        backend:
```

Restart Promtail after updating the configuration.
```Shell
sudo systemctl restart promtail
```

### 4) Configure Log Rotation (Optional)

To prevent the HAProxy log file from growing indefinitely, configure **logrotate**.

Create or edit:
```
/etc/logrotate.d/haproxy
```
```
/var/log/haproxy.log {
    hourly
    size 1G
    rotate 0
    missingok
    notifempty

    postrotate
        [ ! -x /usr/lib/rsyslog/rsyslog-rotate ] || /usr/lib/rsyslog/rsyslog-rotate
    endscript
}
```

This step is optional but highly recommended for production environments.

#### Example Log
```Json
{"time_local":"15/Jun/2026:12:11:18.838","request_id":"8d0ef2d5-7e3b-49ac-8eb2-4e4312f0ab04","remote_addr":"172.19.0.3","remote_port":44278,"frontend_name":"eways","backend_name":"testinventory_https_servers","server_name":"node3","status":200,"bytes_sent":163,"bytes_received":152,"request_method":"GET","request_uri":"/","http_version":"HTTP/2.0","host":"testinventory.example.com","referer":"-","user_agent":"Zabbix","x_forwarded_for":"-","x-real-ip":"-","duration_total_ms":11,"duration_wait_ms":7,"duration_response_ms":4,"termination_state":"--","actconn":5,"feconn":4,"beconn":0,"srvconn":0,"retries":0}
```

