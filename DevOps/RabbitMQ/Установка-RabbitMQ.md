---
title: Установка RabbitMQ
description: 
published: true
date: 2025-07-12T19:21:08.517Z
tags: 
editor: markdown
dateCreated: 2025-07-12T19:16:49.199Z
---

Поднять RabbitMQ можно через docker\\docker-compose или apt

*Установка через docker-compose.yml:*

```plaintext
version: "2.1"
services:
 rabbitmq:
   image: rabbitmq:3.10.7-management
   hostname: rabbitmq
   restart: always
   environment:
     - RABBITMQ_DEFAULT_USER=rmuser
     - RABBITMQ_DEFAULT_PASS=rmpassword
     - RABBITMQ_SERVER_ADDITIONAL_ERL_ARGS=-rabbit log_levels [{connection,error},{default,error}] disk_free_limit 2147483648
   volumes:
     - ./rabbitmq:/var/lib/rabbitmq
   ports:
     - 15672:15672
     - 5672:5672
```

Установка через Docker:

```plaintext
docker run --rm -p 15672:15672 rabbitmq:3.10.7-management
```

Установка в систему через Apt:

```plaintext
#Установка
apt-get install rabbitmq-server -y --fix-missing
#Запуск и добавление в автозагрузку
systemctl start rabbitmq-server
#Запуск веб-интерфейса
rabbitmq-plugins enable rabbitmq_management
```

Подключение осуществляется через веб-UI, по HTTP.

`http://<IP-адрес>:15672`

Данные для входа по стандарту:

```plaintext
Login: guest
Passw: guest
```

Стоит отметить что вводить их надо не в браузерное окно ввода, а именно в графику страницы. Если поднимать через  docker-compose то все данные для входа пишутся в compose файле.

Сразу после установки нужно **Обратить внимание на правый верхний угол:**

![](https://habrastorage.org/r/w1560/getpro/habr/upload_files/3f8/e3b/460/3f8e3b46030546939e60bf49992e72fd.png)

В поле Cluster значение после @ — имя сервера, автоматически назначенное Docker для контейнера. 
Почему это плохо? RabbitMQ хранит стейт в папках, содержащих название сервера, а Docker при пересоздании контейнера даёт ему случайные названия. При каждом пересоздании контейнера RabbitMQ будет терять свой стейт и работать как новый, пустой.

Идём дальше. Посмотрите на поле Disk space, в частности low watermark:

![](https://habrastorage.org/r/w1560/getpro/habr/upload_files/297/1e2/058/2971e2058ed6ed2ba972d02bd6de9612.png)

Эта настройка говорит о том, что RabbitMQ перейдёт в защиту и перестанет писать в стейт при свободном месте менее 48 мбит, что очень мало — порог пробивается в 90% случаев. А если RabbitMQ попытается записать на диск, где нет места, это с 90% вероятностью уничтожит стейт без возможности восстановления. Поэтому я настоятельно рекомендуется для прод инсталляций переопределять это значение на хотя бы 2 гигабита (2147483648 бит). В docker-compose уже расписано все что надо. Вообще пороговое значение подбирается индивидуально в зависимости от параметров сервера и характера нагрузки, но для начала поставить 2 гигабита — уже в 100 раз лучше, чем значение по умолчанию.

---

> В docker-compose.yml уже представлена прод-инсталяция
{.is-success}


