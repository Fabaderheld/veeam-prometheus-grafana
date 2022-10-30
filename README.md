# Veeam B&R dashboard for prometheus

###### guide-by-example

![logo](https://i.imgur.com/xScE6fL.png)

-----------------

**WORK IN PROGRESS**<br>
**WORK IN PROGRESS**<br>
**WORK IN PROGRESS**

---------------

# Purpose

Centralized monitoring dashboard for many backup jobs.

* [Veeam Backup & Replication Community Edition](
https://www.veeam.com/virtual-machine-backup-solution-free.html)
* [Prometheus](https://prometheus.io/)
* [Grafana](https://grafana.com/)

A powershell script would be periodicly running on the machine running Veeam,
that would gather the information about the backup using Get-VBRJob cmdlet.<br>
This info would be pushed to a prometheus pushgateway.<br>
Grafana dashboard would then visualize the information.

# Overview

Components

* machines running veeam B&R
* scheduled tasks running powershell script on these machines<br>
  this is probably the weakest link, least reliable component in this setup
* a dockerhost running container
  * promethus
  * pushgateway
  * grafana
* alertmanager ... to-do 

<details>
<summary><h1>Prometheus and Grafana Setup</h1></summary>

# Files and directory structure

```
/home/
└── ~/
    └── docker/
        └── prometheus/
            │
            ├── grafana/
            │   └── provisioning/
            │       ├── dashboards/
            │       │   ├── dashboard.yml
            │       │   └── veeam-backups.json
            │       │
            │       └── datasources/
            │           └── datasource.yml
            │
            ├── grafana-data/
            ├── prometheus-data/
            │
            ├── .env
            ├── docker-compose.yml
            └── prometheus.yml
```

* `grafana/` - a directory containing grafanas configs and dashboards
* `grafana-data/` - a directory where grafana stores its data
* `prometheus-data/` - a directory where prometheus stores its database and data
* `.env` - a file containing environment variables for docker compose
* `docker-compose.yml` - a docker compose file, telling docker how to run the containers
* `prometheus.yml` - a configuration file for prometheus

All files must be provided.</br>
As well as `grafana` directory and its subdirectories and files.

the directories `grafana-data` and `prometheus-data` are created
by docker compose on the first run.

# docker-compose

Five containers to spin up.</br>
While [stefanprodan/dockprom](https://github.com/stefanprodan/dockprom)
also got alertmanager and pushgateway, this is a simpler setup for now.</br>
Just want pretty graphs.

* **Prometheus** - prometheus server, pulling, storing, evaluating metrics
* **Grafana** - web UI visualization of the collected metrics
  in nice dashboards
* **Pushgateway** - service ready to receive pushed information at an open port

`docker-compose.yml`
```yml
services:

  # MONITORING SYSTEM AND THE METRICS DATABASE
  prometheus:
    image: prom/prometheus:v2.38.0
    container_name: prometheus
    hostname: prometheus
    restart: unless-stopped
    user: root
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--storage.tsdb.retention.time=200h'
      - '--web.enable-lifecycle'
      - '--web.enable-admin-api'
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus_data:/prometheus
    ports:
      - 9090:9090

  # WEB BASED UI VISUALISATION OF THE METRICS
  grafana:
    image: grafana/grafana:9.1.1
    container_name: grafana
    hostname: grafana
    restart: unless-stopped
    user: root
    environment:
      - GF_SECURITY_ADMIN_USER
      - GF_SECURITY_ADMIN_PASSWORD
      - GF_USERS_ALLOW_SIGN_UP
    volumes:
      - ./grafana_data:/var/lib/grafana
      - ./grafana/provisioning/dashboards:/etc/grafana/provisioning/dashboards
      - ./grafana/provisioning/datasources:/etc/grafana/provisioning/datasources
    ports:
      - 3000:3000

  pushgateway:
    image: prom/pushgateway:v1.4.3
    container_name: pushgateway
    hostname: pushgateway
    restart: unless-stopped
    ports:
      - 9091:9091

networks:
  default:
    name: $DOCKER_MY_NETWORK
    external: true
```

`.env`

```bash
# GENERAL
MY_DOMAIN=example.com
DOCKER_MY_NETWORK=caddy_net
TZ=Europe/Bratislava

# GRAFANA
GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=admin
GF_USERS_ALLOW_SIGN_UP=false
```

**All containers must be on the same network**.</br>
Which is named in the `.env` file.</br>
If one does not exist yet: `docker network create caddy_net`

# Reverse proxy

Caddy v2 is used, details
[here](https://github.com/DoTheEvo/selfhosted-apps-docker/tree/master/caddy_v2).</br>

`Caddyfile`
```
grafana.{$MY_DOMAIN} {
    reverse_proxy grafana:3000
}
```

# Prometheus configuration

#### prometheus.yml

* /prometheus/**prometheus.yml**

[Official documentation.](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)

A config file for prometheus, bind mounted in to prometheus container.</br>

`prometheus.yml`
```yml
global:
  scrape_interval:     15s
  evaluation_interval: 15s

# A scrape configuration containing exactly one endpoint to scrape.
scrape_configs:
  - job_name: 'pushgateway'
    scrape_interval: 60s
    honor_labels: true
    static_configs:
      - targets: ['pushgateway:9091']
```

# Grafana configuration

* first run login with admin/admin
* in Preferences > Datasources set `http://prometheus:9090` for url<br>
  save and test should be green
* once some values are pushed to prometheus, create a new dashboard...

</details>

<details>
<summary><h1>Learning in small steps</h1></summary>

what should work at this moment

* \<docker-host-ip>:3000 - grafana
* \<docker-host-ip>:9090 - prometheus 
* \<docker-host-ip>:9091 - pushgateway 

### testing how push data to pushgateway

* metrics must be floats
* for strings labels passed in url can be used 


Prometheus requires linux [line endings.](
https://github.com/prometheus/pushgateway/issues/144)<br>
The "\`n" in the `$body` is to simulate it in windows powershell.

Also in powershell the grave(backtick) character - \` 
is for [escaping stuff](https://ss64.com/ps/syntax-esc.html)<br>
Here it is used to escape new line, which allows breaking the command
in to multiple lines for readability.
It is not related to the previous issue of line endings.

`test.ps1`
```ps1
$body = "free_disk_space 32`n"

Invoke-RestMethod `
    -Method PUT `
    -Uri "http://10.0.19.4:9091/metrics/job/veeam_report/instance/PC1" `
    -Body $body
```

* in the $body we have name of the metrics - `free_disk_space`
* in the url we have two labels, job - `veeam_report` and instance - `PC1`

Heres how the data look in prometheus when executing `free_disk_space` query

![first_put](https://i.imgur.com/9G0QcuT.png)

The metrics and labels help us target the data in grafana.

* create **new dashboard**, panel
* switch type to **Status history**
* select metric - `free_disk_space`
* [query options](https://grafana.com/docs/grafana/next/panels-visualizations/query-transform-data/#query-options)
  * min interval - 1h
  * relative time - now-10h/h
* to not deal with long ugly names add transformation - Rename by regex<br>
  Match - `.+instance="([^"]*).*` - [explained](https://stackoverflow.com/questions/2013124/regex-matching-up-to-the-first-occurrence-of-a-character)<br>
  Replace - `$1`
* can also play with transparency, legend, treshold for pretty colors

should look in the end somewhat like this

![first_graph](https://i.imgur.com/DLnCWdB.png)

*extra info*<br>
[Examples.](https://prometheus.io/docs/prometheus/latest/querying/examples/)
this command deletes all metrics on prometheus, assuming api is enabled<br>
`curl -X POST -g 'http://10.0.19.4:9090/api/v1/admin/tsdb/delete_series?match[]={__name__=~".*"}'`

so now whats tested is sending data to pushgateway and visualize them in grafana

</details>

# Powershell script

<%@include file="veeam_prometheus_info_push.ps1"%>

# Grafana dasboads

First panel is for seeing last X days and result of backups, at quick glance

* new dashboard > new panel
* status history
* select labels job = veeam_report; select metric - veeam_job_result
* query options - min interval 1m; relative time - `now-7m/m`, later switch to `now-7d/d`
* transform - regex by name - `.+instance="([^"]*).*`
* panel title - Backup jobs last X days
* status history > show values - never
* treshold
  * -1 - blue
  * 0 - green
  * 1 - yellow
  * 2 - red

second panel is with more info, most important is age of data

* new panel
* table
* select labels job = veeam_report; select metric - push_time_seconds<br>
  switch from builder to code and `round(time()-push_time_seconds{job="veeam_report"})`<br>
* options - format - table; type - instant;  
* rename query from $A to push_time
* new query
* select labels job = veeam_report; select metric - veeam_job_duration
* options - format - table; type - instant;
* rename query from $A to duration
* transform - regex by name - `.+instance="([^"]*).*`
* transform - outer join - field name = instance
* transform - organize fields - hide time and any other useless columns, rename headers
* table - standard options - units - seconds
* override - fields with names - size - standard option - units - bytes(SI)


-----

https://github.com/jorinvo/prometheus-pushgateway-cleaner

https://github.com/prometheus/pushgateway/issues/19
