---
title: LEMP
description: 
published: true
date: 2026-04-12T17:41:18.972Z
tags: 
editor: markdown
dateCreated: 2026-04-01T17:10:29.683Z
---

# Развертывание распределенного веб-сервиса с маршрутизацией в среде PVE

### Цель работы
Изучить принципы построения изолированных виртуальных сетей в PVE, освоить базовую настройку маршрутизации и NAT с помощью `iptables`, а также получить практические навыки интеграции веб-сервера (Nginx + PHP) с сервером баз данных (MariaDB/MySQL) по сети.

## Топология сети
В рамках работы необходимо создать две сети:
1.  **WAN (Внешняя сеть):** `vmbr0` (существующий мост с доступом в интернет).
2.  **LAN (Внутренняя изолированная сеть):** `vmbr1` (сеть `10.0.0.0/24`).

**Виртуальные машины:**
*   **VM1 (Router):**
    *   Внешний интерфейс (WAN) — получает IP по DHCP (например, `192.168.1.100`).
    *   Внутренний интерфейс (LAN) — статический IP `10.0.0.1`.
*   **VM2 (Nginx Web Server):**
    *   Интерфейс (LAN) — статический IP `10.0.0.2`, шлюз `10.0.0.1`.
*   **VM3 (MariaDB Database):**
    *   Интерфейс (LAN) — статический IP `10.0.0.3`, шлюз `10.0.0.1`.

## Ход работы

### Шаг 1. Подготовка виртуальной среды
1. Зайдите в веб-интерфейс Proxmox. Перейдите в раздел вашего узла -> **Network**.
2. Создайте новый Linux Bridge. Назовите его `vmbr1`. Поле *Bridge ports* оставьте пустым (это создаст изолированную сеть). Нажмите *Apply Configuration*.
3. Создайте 3 виртуальные машины (Router, Web, DB) и установите на них ALT Server.
4. Настройте сетевые адаптеры ВМ:
   * Для **Router**: добавьте два сетевых устройства. Первое подключите к `vmbr0`, второе к `vmbr1`.
   * Для **Web** и **DB**: добавьте по одному сетевому устройству, подключенному **только** к `vmbr1`.

### Шаг 2. Настройка маршрутизатора (VM1 - Router)
1. Узнайте имена ваших сетевых интерфейсов командой `ip a`. Допустим, внешний (WAN) — это `eth0`, а внутренний (LAN) — `eth1`.
2. Настройте внутренний интерфейс через Etcnet:
```bash
vim /etc/net/ifaces/eth1/ipv4address
```
```yaml
10.0.0.1/24
```
```bash
vim /etc/net/ifaces/eth1/resolv.conf
```
```yaml
nameserver 8.8.8.8
```
Примените настройки: `systemctl restart network`.

3. Включите маршрутизацию пакетов (IP Forwarding) в ядре Linux:
Раскомментируйте строку `net.ipv4.ip_forward=1` в файле `/etc/sysctl.conf` и примените:
```bash
sysctl -p
```

4. Настройте NAT (Masquerade), чтобы внутренние машины имели доступ в интернет. **Вместо `eth0` укажите ВАШ внешний интерфейс**:
```bash
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

5. Настройте проброс портов (DNAT), чтобы при обращении на внешний IP роутера по порту 80, трафик уходил на Web-сервер. **Вместо `eth0` укажите ВАШ внешний интерфейс**:
```bash
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j DNAT --to-destination 10.0.0.2:80
```

6. Сохраните правила iptables, чтобы они применялись после перезагрузки:
```bash
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables
```

### Шаг 3. Настройка сервера БД (VM3 - MariaDB)
1. Запустите VM3. Настройте статический IP через Etcnet (в папке вашего интерфейса, например `eth0`):
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

2. Установите и запустите MariaDB:
```bash
apt-get update && apt-get install mariadb-server
systemctl enable --now mariadb.service
```

3. **ВАЖНО:** По умолчанию БД слушает только локальные запросы. Разрешим подключения по сети. Откройте конфигурационный файл (обычно `/etc/my.cnf.d/server.cnf` или `/etc/my.cnf.d/mariadb-server.cnf`):
```bash
vim /etc/my.cnf.d/server.cnf
```
Найдите блок `[mysqld]` и добавьте/измените строку:
```ini
bind-address = 0.0.0.0
```
Перезапустите службу: `systemctl restart mariadb`.

4. Создание БД и пользователя (выполните команды в консоли MySQL):
```bash
mysql
```
```sql
CREATE DATABASE example_db;
CREATE USER 'example_user'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON example_db.* TO 'example_user'@'%';
FLUSH PRIVILEGES;
USE example_db;
CREATE TABLE test (item_id INT AUTO_INCREMENT, content VARCHAR(255), PRIMARY KEY(item_id));
INSERT INTO test (content) VALUES ("Test1");
INSERT INTO test (content) VALUES ("Test2");
EXIT;
```

### Шаг 4. Настройка веб-сервера (VM2 - Nginx)
1. Настройте сеть аналогично БД, но с IP `10.0.0.2/24`. Примените настройки (`systemctl restart network`). Проверьте интернет.
2. Установите Nginx и PHP:
```bash
apt-get update
apt-get install nginx php8.2-fpm-fcgi php8.2-mysqlnd php8.2-mysqlnd-mysqli 
systemctl enable --now php8.2-fpm
```
3. Настройте Nginx. В ALT Linux файлы конфигурации должны иметь расширение `.conf`. Создайте файл `/etc/nginx/sites-available.d/default.conf`:
```bash
vim /etc/nginx/sites-available.d/default.conf
```
Приведите его к следующему виду:
```nginx
server {
    listen 80;
    server_name _;
    root /var/www/html;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        include /etc/nginx/fastcgi_params;
        fastcgi_pass unix:/var/run/php8.2-fpm/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```
Активируйте конфиг и перезапустите Nginx: 
```bash
ln -s /etc/nginx/sites-available.d/default.conf /etc/nginx/sites-enabled.d/default.conf
systemctl enable --now nginx
systemctl restart nginx
```

4. Создайте PHP-скрипт для вывода данных из БД. Создайте директорию и файл `/var/www/html/index.php`:
```bash
mkdir -p /var/www/html
vim /var/www/html/index.php
```
Вставьте следующий код:
```php
<?php
$user = "example_user";
$password = "password";
$database = "example_db";
$table = "test";

// Подключаемся к IP-адресу сервера БД (VM3)
$conn = mysqli_connect("10.0.0.3", $user, $password, $database);

if (!$conn) {
    die("Connection failed: " . mysqli_connect_error());
}
echo "<h2>Вывод из БД MariaDB</h2><ol>";
foreach($conn->query("SELECT content FROM $table") as $row) {
    echo "<li>" . $row['content'] . "</li>";
  }
echo "</ol>";
mysqli_close($conn);
?>
```
Удалите дефолтный файл Nginx (если есть): `rm -f /var/www/html/index.html`.

### Шаг 5. Проверка работоспособности
1. Узнайте внешний IP-адрес маршрутизатора (VM1) командой `ip a` (на внешнем интерфейсе).
2. Откройте браузер на хостовой машине и введите этот IP-адрес: `http://<IP_РОУТЕРА>`.
3. Если всё настроено верно, iptables перенаправит запрос на VM2 (Nginx), Nginx передаст его в PHP, PHP подключится к VM3 (MariaDB), заберет данные и выведет их в виде HTML-списка в вашем браузере.

# Контрольные вопросы 
1. Для чего используется параметр `net.ipv4.ip_forward=1` на маршрутизаторе?
2. В чем разница между цепочками `PREROUTING` и `POSTROUTING` в таблице `nat` iptables?
3. Зачем мы изменяли параметр `bind-address = 0.0.0.0` в конфигурации MariaDB?
4. Почему мы не можем установить прямое соединение между Nginx и MariaDB без использования обработчика (например, PHP-FPM)?
5. Что такое изолированный мост (Bridge без портов) в Proxmox и как он обеспечивает безопасность внутренней сети?