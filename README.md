# Grafana-Loki Logs monitoring for multpile AWS EC2 instacne
## Observability Stack

Here, I'm trying to showcase how logs are being monitored in Grafana using Loki in the simplest way possible, or in other words, how the observability stack is utilized in the DevOps environment. I'm attempting to make this information useful for those who are interested in understanding the basics of Grafana Loki management.

The basic working behind this is as follows: **LOKI** is deployed to collect, index, and store log data, exposing its services on port 3100. Another component, **Promtail**, collects logs from machines, formats them, and sends them to Loki. Following that, Grafana and Prometheus come into play. **Prometheus** scrapes metrics data and is configured to collect metrics from services running in containers, exposing its services on port 9090. For visualization, **Grafana** connects to Loki and Prometheus to create dashboards and visualize both logs and metrics data

## Features
- Getting familiar with the observability components: Loki for logging, Promtail for log shipping, Prometheus for monitoring, and Grafana for visualization.
- This setup forms a foundation for logging, monitoring, and visualization in a local environment only. For production, additional configurations and security measures may be required.

## Pre-Requests
- Created an EC2 instance with t2.micro for the Observability stack.
- Configured Docker and Docker Compose in the environment ( [Docker Configuration Steps](https://docs.docker.com/engine/install/ubuntu/) ) 
- Additionally, created another EC2 instance with t2.micro and configured Apache2 for shipping and monitoring its logs to Loki


## Docker-Compose file 
- Here Loki Exposes port 3100 and Specifies the Loki configuration file with` -config.file=/etc/loki/local-config.yaml`.
- Promtail ensures starts after Loki using the `depends_on` and volume Maps `/var/log` on the host to `/var/log` in the container and `./promtail` on the host to `/etc/promtail` in the container as of now.
- Prometheus Exposes port 9090 and Grafana Exposes port 3000. 
- Here as it is for testing only Grafana environment variables sets including the admin username and password.
- Bridge network named `grafana-loki` is used that all services are connected.

```
version: "3"
services:
    loki:
        image: grafana/loki
        ports:
          - "3100:3100"
        command: -config.file=/etc/loki/local-config.yaml
        networks:
          - grafana-loki
    promtail:
        depends_on:
            - loki
        image: grafana/promtail
        volumes:
            - /var/log:/var/log
            - ./promtail:/etc/promtail
        command: -config.file=/etc/promtail/config.yml
        networks:
          - grafana-loki
    prometheus:
        image: prom/prometheus:latest
        ports:
            - "9090:9090"
        command: >-
            --config.file=/etc/prometheus/prometheus.yml
            --storage.tsdb.path=/prometheus
            --web.console.libraries=/usr/share/prometheus/console_libraries
            --web.console.templates=/usr/share/prometheus/consoles
        networks:
          - grafana-loki
    grafana:
        depends_on:
            - loki
        image: grafana/grafana:latest
        ports:
            - "3000:3000"
        environment:
            GF_SECURITY_ADMIN_USER: admin
            GF_SECURITY_ADMIN_PASSWORD: test
            GF_PATHS_PROVISIONING: '/app.cfg/provisioning'
        volumes:
            - ./grafana:/app.cfg
        networks:
          - grafana-loki
networks:
  grafana-loki:
```
### Promtail Config file
- Here promtail is used to scrape log entries from `/var/log/` on the local machine and push them to Loki at `http://loki:3100/loki/api/v1/push`
- `scrape_configs` is configured as such to specifying from which path the logs needs to be scraped by providing with the `__path__`
```
---
server:
  http_listen_port: 9080
  grpc_listen_port: 0
positions:
  filename: /tmp/positions.yaml
clients:
  - url: http://loki:3100/loki/api/v1/push
scrape_configs:
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          __path__: /var/log/*log
```
- Configured Promtail on another EC2 instance to push the logs to Loki. Apache is already configured on this instance
> In this configuration, log entries are scraped from `/var/log/apache2/` on the machine and pushed to Loki at `http://ec2-instance-ip:3100/loki/api/v1/push`
```
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://13.235.104.34:3100/loki/api/v1/push

scrape_configs:
  - job_name: ec2-instance
    static_configs:
      - targets:
          - localhost
        labels:
          job: apache2-logs
          instance: ec2-instance
          __path__: /var/log/apache2/*log
```
## Start the stack and verifying the Setup

- `docker-compose up -d` to start the entire stack.
> Once Loki is deployed,will be able to see Loki metrics via [http://ec2-instance-ip:3100](http://ec2-instance-ip:3100)

![loki](https://github.com/ajish-antony/Grafana-Loki-Observability-Stack/assets/48723128/0fa96177-38a4-44eb-bb94-4f8073d9af5d)

>Grafana can be accessd by  [http://ec2-instance-ip:3000](http://ec2-instance-ip:3000) 

![grafana](https://github.com/ajish-antony/Grafana-Loki-Observability-Stack/assets/48723128/f5c2601e-d7eb-4230-b622-28b325b8717e)

> Configure the data source for Loki and verify the connectivity. In the case of production, provide the necessary authentication

![loki-3](https://github.com/ajish-antony/Grafana-Loki-Observability-Stack/assets/48723128/3700d244-1368-410c-bc2e-5603fa591583)

![loki-2](https://github.com/ajish-antony/Grafana-Loki-Observability-Stack/assets/48723128/97090a54-9c8c-498e-82ca-306348fde50b)


>Here it's visible that both the configured jobs are available in Loki

![explore-1](https://github.com/ajish-antony/Grafana-Loki-Observability-Stack/assets/48723128/d44c6856-7bf3-4743-b4e1-4b0a41d8a52a)

>The logs that have been pushed from the specified paths

![explore-2](https://github.com/ajish-antony/Grafana-Loki-Observability-Stack/assets/48723128/5715ebeb-fc39-4209-b7b0-3535981f7350)

## Reference
- [LOKI](https://grafana.com/oss/loki/)
- [Grafana](https://grafana.com/oss/grafana/)
- [Grafana & Prometheus](https://grafana.com/docs/grafana/latest/getting-started/get-started-grafana-prometheus/)
- [Promtail](https://grafana.com/docs/loki/latest/send-data/promtail/)


## Conclusion

Here, what has been configured can be used for a basic understanding setup of the observability stack using these various components. When it comes to production, it needs to be configured with much more detail and security. I hope this helps beginners to get started.


### ⚙️ Connect with Me

<p align="center">
<a href="mailto:ajishantony95@gmail.com"><img src="https://img.shields.io/badge/Gmail-D14836?style=for-the-badge&logo=gmail&logoColor=white"/></a>
<a href="https://www.linkedin.com/in/ajish-antony/"><img src="https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white"/></a>
