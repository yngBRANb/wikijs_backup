---
title: Установка ELK filebeat
description: 
published: true
date: 2025-07-14T22:35:14.440Z
tags: 
editor: markdown
dateCreated: 2025-07-14T22:35:05.188Z
---

# Плейбук для установки filebeat
Сам плейбук ниже, использует jinja2 и пока что работает только на debian. Каждая строчка прокомментирована для понимания что для чего делается.

filebeat-install.yml
```yml
---
# Начало YAML-документа (стандартный формат Ansible)

# Блок определения play (набора задач для выполнения)
- name: Install and configure Filebeat  # Описание цели playbook
  hosts: all                          # Применять на всех хостах из инвентаря
  become: yes                         # Выполнять с повышенными привилегиями (sudo)

  # Определение переменных
  vars:
    filebeat_url: "https://mirror.cyberkal.ru/files/filebeat/filebeat-8.18.2-amd64.deb"  # URL для скачивания
    filebeat_deb: "/tmp/filebeat-8.18.2-amd64.deb"  # Локальный путь для сохранения .deb пакета
    logstash_host: "172.16.0.12:5044"       # Адрес Logstash сервера (заменить)
    elasticsearch_host: "http://172.16.0.12:9200"  # Адрес Elasticsearch (заменить)

  # Блок задач (tasks) - последовательность выполняемых действий
  tasks:
    # Задача 1: Установка wget (если не установлен)
    - name: Ensure wget is installed # Описание задачи
      apt:                          # Использование модуля apt (для Debian/Ubuntu)
        name: wget                  # Имя пакета для установки
        state: present              # Гарантировать, что пакет установлен
        update_cache: yes           # Обновить кеш пакетов перед установкой

    # Задача 2: Скачивание Filebeat
    - name: Download Filebeat
      get_url:                      # Модуль для загрузки файлов по URL
        url: "{{ filebeat_url }}"   # URL источника (использует переменную)
        dest: "{{ filebeat_deb }}"  # Путь назначения (использует переменную)
        mode: '0644'                # Права на файл (rw-r--r--)
      register: download_result     # Сохранить результат выполнения в переменную
      until: download_result is succeeded  # Повторять пока не получится
      retries: 3                   # Максимум 3 попытки
      delay: 10                    # Пауза 10 сек между попытками

    # Задача 3: Установка Filebeat из .deb пакета
    - name: Install Filebeat
      apt:                          # Модуль для работы с .deb пакетами
        deb: "{{ filebeat_deb }}"   # Путь к .deb файлу
        state: present              # Гарантировать установку

    # Задача 4: Настройка конфигурации Filebeat
    - name: Configure Filebeat
      template:                     # Модуль для работы с шаблонами
        src: filebeat.yml.j2        # Исходный шаблон (Jinja2)
        dest: /etc/filebeat/filebeat.yml  # Конечный путь конфига
        owner: root                 # Владелец файла
        group: root                 # Группа файла
        mode: '0640'               # Права (rw-r-----)
      notify:                      # Уведомление handler'а
        - restart filebeat         # Имя handler'а для вызова

    # Задача 5: Запуск и включение автозапуска Filebeat
    - name: Ensure Filebeat is enabled and running
      service:                     # Модуль управления службами
        name: filebeat             # Имя службы
        state: started             # Гарантировать, что служба запущена
        enabled: yes               # Включить автозапуск при загрузке

  # Блок handlers (обработчики) - выполняются по уведомлению
  handlers:
    # Handler 1: Перезапуск Filebeat при изменении конфига
    - name: restart filebeat
      service:
        name: filebeat
        state: restarted           # Перезапустить службу
```

Jinjia шаблон (должен находиться по пути /templates):

filebeat.yml.j2
```yml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/*  # Сбор логов со всего /var/log/

output.logstash:
  hosts: ["{{ logstash_host }}"]  # Вставка переменной из playbook

xpack.monitoring:
  enabled: true
  elasticsearch:
    hosts: ["{{ elasticsearch_host }}"]  # Вставка переменной
```

Запускается:
```bash
ansible-playbook -i inventory.ini filebeat-install.yml
```

Здесь: 
- `ansible-playbook` - просим выполнить плейбук
-  `-i` - указываем файл инвентаря
-  `filebeat-install.yml` - файл плейбука
