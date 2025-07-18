---
title: Filebeat
description: 
published: true
date: 2025-07-14T22:55:20.281Z
tags: 
editor: markdown
dateCreated: 2025-07-14T22:55:09.010Z
---

# Работа с filebeat

Для сборки логов и слива их в logstash или напрямую в elasticsearch используется filebeat, это такой аналог нод экспортеров в prometheus.

Установка:

```bash
wget https://mirror.cyberkal.ru/files/filebeat/filebeat-8.18.2-amd64.deb
dpkg -i filebeat-8.18.2-amd64.deb
```

Вся конфигурация вносится в `/etc/filebeat/filebeat.yml`

Конфигурация filebeat.yml:

```yml
filebeat.inputs:
- type: log
  enabled: true
  paths:
      - /var/log/*
  
output.logstash:
  hosts: ["адрес сервера:5044"]

xpack.monitoring:
  enabled: true
  elasticsearch:
    hosts: ["http://адрес сервеа:9200"]
```

После написания конфига нужно проверить его и применить, есть два способа как запустить  filebeat:
- Как юнит systemd
- Как исполняемую команду

## 1 Вариант: Как unit

```bash
filebeat test config
systemctl enable filebeat.service
systemctl start filebeat.service
```

## 2 Вариант: Как исполняемую команду

```bash
filebeat test config
filebeat run
```

При использовании в качестве исполняемой команды очевидно что CLI заблокируется, да и очевидно что сервисом это удобнее юзать.