---
title: Создание хранилища Ceph для Kubernetes (ceph-ansible)
description: 
published: true
date: 2025-07-18T04:04:02.697Z
tags: devops, kubernetes, ceph
editor: markdown
dateCreated: 2025-07-12T21:47:32.612Z
---

# Структура кластера Ceph

>Важно! 
Способ развертывания через ceph-ansible устарел и на данный момент имеет множетсво недостатков.
Для развертывания Ceph на выделенных нодах лучше использовать новый официальный инструмент - Cephadm.
> {.is-warning}


В кластере Ceph есть основные 3 роли которые может иметь узел:

-   **MON**  
    Монитор — это демон, выполняющий роль координатора, с которого начинается кластер. Как только у нас появляется хотя бы один рабочий монитор, у нас появляется Ceph-кластер. Монитор хранит информацию о здоровье и состоянии кластера, обмениваясь различными картами с другими мониторами. Клиенты обращаются к мониторам, чтобы узнать, на какие OSD писать/читать данные. При разворачивании нового хранилища, первым делом создается монитор (или несколько). Кластер может прожить на одном мониторе, но рекомендуется делать 3 или 5 мониторов, во избежание падения всей системы по причине падения единственного монитора. Главное, чтобы количество оных было нечетным, дабы избежать ситуаций раздвоения сознания (split-brain). Мониторы работают в кворуме, поэтому если упадет больше половины мониторов, кластер заблокируется для предотвращения рассогласованности данных.
-   **MGR**  
    Менеджер Ceph демон работает вместе с демоном монитора, чтобы обеспечить дополнительный контроль.  
    С версии 12.x демон ceph-mgr стал необходим для нормальной работы.  
    Если демон mgr не запущен, вы увидите предупреждение об этом.
-   **OSD (Object Storage Device)**  
    OSD — это юнит хранилища, который хранит сами данные и обрабатывает запросы клиентов, обмениваясь данными с другими OSD. Обычно это диск. И обычно за каждый OSD отвечает отдельный OSD-демон, который может запускаться на любой машине, на которой установлен этот диск.

Так же есть 2 дополнительные роли:

-   **RGW (RADOS Gateway)**   
    RGW — это шлюз, предоставляющий доступ к хранилищу Ceph через RESTful API, совместимый с Amazon S3 и OpenStack Swift. Он позволяет использовать Ceph в качестве объектного хранилища для облачных приложений. RGW не является обязательным компонентом для базовой работы Ceph, но необходим, если требуется доступ к данным через S3/Swift-совместимые интерфейсы. В Kubernetes (K8s) RGW может быть полезен, если кластеру нужен объектный storage-класс, например, для работы с приложениями, использующими S3-совместимое хранилище.
-   **MDS (Metadata Server)**   
    MDS — это сервер метаданных, используемый в файловой системе CephFS. Он отвечает за хранение иерархии файлов и директорий, а также за управление правами доступа. MDS требуется только при использовании CephFS. Если в Kubernetes планируется развертывание CephFS в качестве PersistentVolume, то MDS необходим. В противном случае (например, при использовании только RBD — блочного устройства) MDS не нужен.

**Нужны ли RGW и MDS для Kubernetes?**

-   RGW нужен в Kubernetes, если требуется объектное хранилище (например, для приложений, работающих с S3 API).
-   MDS нужен в Kubernetes только при использовании CephFS в качестве файловой системы для PersistentVolumes. Если Kubernetes использует Ceph только через RBD (блочные устройства), то ни RGW, ни MDS не требуются.

## Варианты интеграции K8s и Ceph

На текущий момент существует 3 подхода (+ RGW для организации S3 хранилища) к их связи:

| **Способ** | **Простота** | **Подходит для** | **Нужные компоненты Ceph** |
| --- | --- | --- | --- |
| **RBD (вручную)** | Средняя | Блочные диски (аналог EBS) | MON, OSD, MGR |
| **Ceph CSI (RBD)** | Лучший вариант | Автоматическое управление томами | MON, OSD, MGR |
| **CephFS** | Сложнее | Файловое хранилище (аналог NFS) | MON, OSD, MGR, **MDS** |
| **RGW (S3)** | Отдельная тема | Объектное хранилище (аналог MinIO) | MON, OSD, MGR, **RGW** |

Самый походящий способом связывания сегодня это использование CSI (Container Storage Interface) драйвера, который позволяет динамически управлять хранилищем. Это обеспечивает гибкость и масштабируемость, что делает его лучшей практикой для продакшн-сред.

## Особенности создания сети между K8s и Ceph

Для production среды между кластером Kubernetes и кластером Ceph должна быть выделенная сеть с пропускной способностью не менее 10 Гбит/с (Public Network), где будет идти общение между подами в K8s и их volume в Ceph.

Так же должна быть сеть Cluster Network, предназначенная только для репликации данных между узлами Ceph.

Но для тестовой среды подойдет и одна общая сеть.

## Информация о кластере

На каждой машине нашего кластера будут работать все три демона. Соответственно, демоны монитора и менеджера как служебные, и демоны OSD для дисков нашей виртуальной машины. Используемый дистрибутив - Red Hat Enterprise Linux 10.0.

Развертывание будет осуществляться с помощью набора плейбуков [ceph-ansible](https://github.com/ceph/ceph-ansible?tab=readme-ov-file). Документация расположена [тут](https://docs.ceph.com/projects/ceph-ansible/en/latest/).

Перед началом на каждом сервере должно быть корректное время. Желательно использование одного и того же NTP сервера.

> На каждом узле рекомендуется иметь по 3 диска. 1 - Система, 2 - Хранилище, 3 - Хранилище + Журнал
> 
> К тому же, если OSD (дисков) в кластере будет меньше 4, то будет использоваться репликация, за место эффективного Erasure Coding.

Актуальную версию Ceph можно узнать на [официальном сайте](https://docs.ceph.com/en/latest/releases/).

## Требования к узлам-хранилищам

Узлы которые будут использоваться с ролью OSD, нужно подготовить к добавлению в кластер Ceph.

Диски которые будут использоваться в качестве хранилища должны быть подключены к машинам, но не иметь на них данных (ни ФС, ни таблицы разделов). Если на них что-то есть, то данные будут автоматически стерты (я не проверял, но должно быть именно так).

# Начало работы

Для установки кластера желательно иметь выделенную Admin машину:

![](/ssa/devops/ceph-admin.png)

(но не обязательно, можно установить ansible-ceph прямо на один из узлов)

В моем случае это будет дистрибутив с которого отрабатывал kubespray. С этой машины будет отрабатывать Ansible playbook, который и развернет сам Ceph кластер.

```plaintext
ssh-keygen -t ed25519 -f ~/.ssh/ceph
ssh-copy-id -i ~/.ssh/ceph.pub service@ceph-1.k8s.rhel
ssh-copy-id -i ~/.ssh/ceph.pub service@ceph-2.k8s.rhel
ssh-copy-id -i ~/.ssh/ceph.pub service@ceph-3.k8s.rhel
```

Клонирование репозитория проекта:

```plaintext
cd
git clone https://github.com/ceph/ceph-ansible.git
cd ceph-ansible
```

Как и в статье по установке K8s с помощью kubespray, будет использоваться venv для компонентов Ansible:

```plaintext
sudo dnf install python3 python3-devel python3-pip
python3 -m venv ceph-venv
source ~/ceph-ansible/ceph-venv/bin/activate
```

Установка зависимостей (включая Ansible и его модули):

```plaintext
pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yml
```

В этой же директории создайте inventory файл (под ceph удобнее писать в yaml):

```plaintext
all:
  vars:
    # Параметры авторизации
    ansible_user: service
    ansible_ssh_private_key_file: ~/.ssh/ceph
  

    # Общие параметры для всех узлов
    monitor_interface: ens18           # Единый интерфейс для всех MON
    public_network: "172.16.0.0/23"    # Публичная сеть (для клиентов)
    cluster_network: "10.10.0.0/24"    # Приватная сеть (репликация OSD, НЕОБЯЗАТЕЛЬНО, я ее не реализовывал)
    ceph_mgr_modules: "prometheus"     # Включение метрик для мониторинга
    configure_monitoring: false
    configure_grafana: false

  hosts:
    ceph-1.k8s.rhel:
      devices:                         # Диски под OSD (НЕ системные!)
        - /dev/sdb
        - /dev/sdc

    ceph-2.k8s.rhel:
      devices:
        - /dev/sdb
        - /dev/sdc

    ceph-3.k8s.rhel:
      devices:
        - /dev/sdb
        - /dev/sdc

  children:
    mons:  # Мониторы (обязательно на всех 3 нодах для кворума)
      hosts:
        ceph-1.k8s.rhel:
        ceph-2.k8s.rhel:
        ceph-3.k8s.rhel:

    mgrs:  # Менеджеры (для HA достаточно 2)
      hosts:
        ceph-1.k8s.rhel:
        ceph-2.k8s.rhel:
        ceph-3.k8s.rhel:

    osds:  # Хранилище (все ноды)
      hosts:
        ceph-1.k8s.rhel:
        ceph-2.k8s.rhel:
        ceph-3.k8s.rhel:

    # Это вы указываете если встроенный мониторинг кластера не нужен (если нужен смотрите документацию)
    monitoring:
      hosts: {}

```

Скопируйте файл с настройками:

```plaintext
cp group_vars/all.yml.sample group_vars/all.yml
```

Дополнительные настройки **(group\_vars/all.yml):**

```plaintext
ceph_stable_release: squid    # Тут можно указать желаемую версию (по умолчанию последняя доступная)
ceph_origin: repository
ceph_repository: community

# OSD
## Параметр, который ограничивает максимальный объем оперативной памяти, используемый 
## одним OSD в Ceph. Если указано значение 2 GB, а на узле 2 диска под хранилище (соответственно 2 OSD),
## то будет использовано 4 GB RAM. Самый минимум это 1 GB под 1 OSD.
osd_memory_target: 1.5GB        # Подстройте под ваш RAM

# Оптимизации
## Автоматически регулирует размер кэша BlueStore в рамках osd_memory_target
bluestore_cache_autotune: true
## Ограничивает количество потоков ввода-вывода на один OSD.
## Подходит для малых кластеров на HDD, с малым количество CPU.
osd_op_num_threads_per_shard: 2

# CSI-совместимые настройки
pool_default_size: 3          # Репликация на 3 OSD
pool_default_pg_num: 256      # PG для маленького кластера
rbd_default_features: "layering,exclusive-lock,object-map,fast-diff,deep-flatten"
```

## Запуск установки Ceph

Проверка доступности узлов

```plaintext
ansible -i inventory all -m ping --become

# Проверка, видны ли диски на узлах
ansible -i inventory all -m shell -a "lsblk"
```

Скопируйте файл playbook

```plaintext
cp site.yml.sample site.yml
```

Откройте site.yml и в самом начале измените default(false) на default(true)

```plaintext
- hosts: localhost
  connection: local
  tasks:
    - name: Warn about ceph-ansible current status
      fail:
        msg: "cephadm is the new official installer. Please, consider migrating.
              See https://docs.ceph.com/en/latest/cephadm/install for new deployments
              or https://docs.ceph.com/en/latest/cephadm/adoption for migrating existing deployments."
      when: not yes_i_know | default(true) | bool
```

Если все закончилось без ошибок, то запустите playbook развертывания:

```plaintext
ansible-playbook -i inventory.yml site.yml --become -v
```