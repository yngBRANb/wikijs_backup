---
title: LEMP
description: 
published: true
date: 2026-04-01T17:10:29.683Z
tags: 
editor: markdown
dateCreated: 2026-04-01T17:10:29.683Z
---

# Развертывание распределенного веб-сервиса с маршрутизацией в среде PVE

### Цель работы
Изучить принципы построения изолированных виртуальных сетей в PVE, освоить базовую настройку маршрутизации и NAT с помощью `iptables`, а также получить практические навыки интеграции веб-сервера (Nginx + PHP) с сервером баз данных (MySQL) по сети.

## Топология сети
В рамках работы необходимо создать две сети:
1.  **WAN (Внешняя сеть):** `vmbr0` (существующий мост с доступом в интернет).
2.  **LAN (Внутренняя изолированная сеть):** `vmbr1` (сеть `10.0.0.0/24`).

**Виртуальные машины:**
*   **VM1 (Router):**
    *   `eth0` (WAN) — получает IP по DHCP (например, `192.168.1.100`).
    *   `eth1` (LAN) — статический IP `10.0.0.1`.
*   **VM2 (Nginx Web Server):**
    *   `eth0` (LAN) — статический IP `10.0.0.2`, шлюз `10.0.0.1`.
*   **VM3 (PostgreSQL Database):**
    *   `eth0` (LAN) — статический IP `10.0.0.3`, шлюз `10.0.0.1`.

## Ход работы

### Шаг 1. Подготовка виртуальной среды
1. Зайдите в веб-интерфейс ALT. Перейдите в раздел вашего узла -> **Network**.
2. Создайте новый Linux Bridge. Назовите его `vmbr1`. Поле *Bridge ports* оставьте пустым (это создаст изолированную сеть). Нажмите *Apply Configuration*.
3. Создайте 3 виртуальные машины (Router, Web, DB) и установите на них ALT Server.
4. Настройте сетевые адаптеры ВМ:
   * Для **Router**: добавьте два сетевых устройства. Первое подключите к `vmbr0`, второе к `vmbr1`.
   * Для **Web** и **DB**: добавьте по одному сетевому устройству, подключенному **только** к `vmbr1`.

### Шаг 2. Настройка маршрутизатора (VM1 - Router)
1. Запустите Router. Настройте сетевые интерфейсы (через Etcnet в `/etc/net/ifaces/ens*/ipv4address, resolv.conf`):
```bash
vim ipv4address
```
```yaml
10.0.0.1/24
```
```bash
vim resolv.conf
```
```yaml
nameserver 8.8.8.8
```
Примените настройки: `systemctl restart network`.

2. Включите маршрутизацию пакетов (IP Forwarding) в ядре Linux:
Раскомментируйте строку `net.ipv4.ip_forward=1` в файле `/etc/sysctl.conf` и примените:
```bash
sysctl -p
```

3. Настройте NAT (Masquerade), чтобы внутренние машины имели доступ в интернет для скачивания пакетов:
```bash
sudo iptables -t nat -A POSTROUTING -o ens  -j MASQUERADE #в ens укажите ваш интерфейс (ip -c a)
```

4. Настройте проброс портов (DNAT), чтобы при обращении на внешний IP роутера по порту 80, трафик уходил на Web-сервер:
```bash
iptables -t nat -A PREROUTING -i enp1s0 -p tcp --dport 80 -j DNAT --to-destination 10.0.0.2:80
```

5. Сохраните правила iptables, чтобы они применялись после перезагрузки:
```bash
iptables-save >> /etc/sysconfig/iptables
systemctl enable --now iptables
```

### Шаг 3. Настройка сервера БД (VM3 - PostgreSQL)
1. Запустите VM3. Настройте статический IP через Etcnet:
```bash
vim ipv4address
```
```yaml
10.0.0.3/24
```
```bash
vim ipv4route
```
```yaml
default via 10.0.0.1
```
```bash
vim resolv.conf
```
```yaml
nameserver 8.8.8.8
```
Примените настройки: `systemctl restart network`. Проверьте наличие интернета (`ping ya.ru`).

2. Установите PostgreSQL:
```bash
apt-get update && apt-get install mariadb
```
3. Запуск службы:
```
systemctl enable --now mariadb.service
```
4. Задать пароль root для MySQL и настройки безопасности:
```
mysql_secure_installation
```
5. Создание БД:
```
mysql
CREATE DATABASE example_db;
CREATE USER 'example_user'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON example_db.* TO 'example_user'@'%';
FLUSH PRIVILEGES;
exit;
---
mysql -u example_user -p
SHOW DATABASES;
USE example_db;
CREATE TABLE test (item_id INT AUTO_INCREMENT, content VARCHAR(255), PRIMARY KEY(item_id));
INSERT INTO test (content) VALUES ("Test1");
INSERT INTO test (content) VALUES ("Test2");
EXIT
```
### Шаг 4. Настройка веб-сервера (VM2 - Nginx)
1. Настройте сеть аналогично БД, но с IP `10.0.0.2/24`. Примените Netplan. Проверьте интернет.
2. Установите Nginx и PHP (для связи с БД Nginx нужен обработчик):
```bash
apt-get update
apt-get install nginx php8.2-fpm-fcgi php8.2-mysqlnd php8.2-mysqlnd-mysqli 
systemctl enable --now php8.2-fpm
```
3. Настройте Nginx для работы с PHP. Откройте `/etc/nginx/sites-available.d/default` и приведите блок `server` к виду:
```nginx
server {
    listen 80;
    server_name _;
    index       index.php;
    root /var/www/;
    location / {
        try_files $uri =404;
    }

    location ~ \.php$ {
        try_files $uri =404;
        include /etc/nginx/fastcgi_params;
        fastcgi_pass unix:/var/run/php8.2-fpm/php8.2-fpm.sock; #версия может отличаться
        fastcgi_param SCRIPT_FILENAME /var/www/test.alt/$fastcgi_script_name;
    }
    access_log /var/log/nginx/test.alt-access.log;
}
```
Активировать конфиг: `ln -s /etc/nginx/sites-available.d/default /etc/nginx/sites-enabled.d/`
Перезапустите Nginx: `systemctl enable --now nginx`.

4. Создайте PHP-скрипт для вывода данных из PostgreSQL. Создайте файл `/var/www/test.alt/list.php`:
```php
<?php
$user = "example_user";
$password = "password";
$database = "example_db";
$table = "test";
$conn = mysqli_connect("localhost", $user, $password, $database);
if (!$conn) {
    die("Connection failed: " . mysqli_connect_error());
}
echo "<h2>Вывод из БД</h2><ol>";
foreach($conn->query("SELECT content FROM $table") as $row) {
    echo "<li>" . $row['content'] . "</li>";
  }
echo "</ol>";
mysqli_close($conn);
?>
```
Удалите дефолтный файл: `rm /var/www/html/index.html`.

### Шаг 5. Проверка работоспособности
1. Узнайте внешний IP-адрес маршрутизатора (VM1) командой `ip -c a` (интерфейс `enp1s0`).
2. Откройте браузер на хостовой машине и введите этот IP-адрес: `http://<IP_РОУТЕРА>`.
3. Если всё настроено верно, iptables перенаправит запрос на VM2 (Nginx), Nginx передаст его в PHP, PHP подключится к VM3 (PostgreSQL), заберет данные и выведет их в виде HTML-таблицы в вашем браузере.

# Контрольные вопросы для защиты работы
1. Для чего используется параметр `net.ipv4.ip_forward=1` на маршрутизаторе?
2. В чем разница между цепочками `PREROUTING` и `POSTROUTING` в таблице `nat` iptables?
3. Какую роль выполняет файл `pg_hba.conf` в PostgreSQL?
4. Почему мы не можем установить прямое соединение между Nginx и PostgreSQL без использования языка программирования (например, PHP или Python)?
5. Что такое изолированный мост (Bridge без портов) в Proxmox и как он обеспечивает безопасность внутренней сети?
