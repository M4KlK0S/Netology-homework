# Домашнее задание к занятию "Практическое задание с самопроверкой «Система мониторинга Zabbix. Часть 2»" - Костенко Максим

> [!IMPORTANT]
> Инструкция составлена с учётом всех проблем, возникших в процессе реальной установки: несовместимость репозиториев, пустая база данных, ошибки Apache, настройка PHP-FPM, включение русской локали, добавление хостов и подключение агентов.

Что понадобится:
- Виртуальная машина или сервер с Ubuntu 26.04
- Права sudo
- IP-адрес машины (узнать: ip a или hostname -I)
- В примерах используется IP 192.168.0.138. Замените его на свой.

Почему PHP-FPM, а не mod_php:
- PHP-FPM работает как отдельный процесс, не нагружая Apache
- Обходит ограничение systemd MemoryDenyWriteExecute, которое в Ubuntu 26.04 ломает JIT в mod_php
- Это стандартная схема для production-окружений
- Конфиг Zabbix «из коробки» уже настроен под PHP-FPM — меньше правок

## 1. Установка Zabbix Server с веб-интерфейсом

**1. Обновление системы и установка PostgreSQL**
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install postgresql -y
```
Проверьте, что служба запущена:
```bash
sudo systemctl status postgresql --no-pager
```
## 2. Подключение правильного репозитория Zabbix

```bash
wget https://repo.zabbix.com/zabbix/7.4/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.4+ubuntu26.04_all.deb
sudo dpkg -i zabbix-release_latest_7.4+ubuntu26.04_all.deb
sudo apt update
```
> [!WARNING]
> Если dpkg сообщит о конфликте файлов — удалите старые списки репозиториев Zabbix и повторите:
```bash
sudo rm -f /etc/apt/sources.list.d/zabbix.list /etc/apt/sources.list.d/zabbix-tools.list
sudo apt update
```
**3. Установка компонентов Zabbix**

Устанавливаем Zabbix Server, Web Frontend, Agent и PHP-FPM. Обратите внимание: мы не ставим libapache2-mod-php — вместо него будем использовать PHP-FPM.
```bash
sudo apt install zabbix-server-pgsql zabbix-frontend-php php8.5-pgsql php8.5-fpm libapache2-mod-fcgid zabbix-apache-conf zabbix-sql-scripts zabbix-agent -y
```
> [!IMPORTANT]
> Пакет libapache2-mod-fcgid нужен для того, чтобы Apache умел работать с FastCGI-процессами, а php8.5-fpm — это сам менеджер FastCGI-процессов PHP.

**4. Создание базы данных и пользователя**

Система запросит пароль для пользователя zabbix — придумайте и запишите его, он понадобится на нескольких следующих шагах.
```bash
sudo -u postgres createuser --pwprompt zabbix
sudo -u postgres createdb -O zabbix zabbix
```
**5. Импорт схемы базы данных**

Важно: путь к файлу схемы может отличаться в зависимости от версии пакета. Сначала найдите его:
```bash
find /usr/share -name "server.sql.gz" 2>/dev/null
```
На практике встречаются два варианта:
```bash
/usr/share/zabbix/sql-scripts/postgresql/server.sql.gz
/usr/share/zabbix-sql-scripts/postgresql/server.sql.gz
```
Импортируйте схему, используя найденный путь:
```bash
zcat /usr/share/zabbix/sql-scripts/postgresql/server.sql.gz | sudo -u zabbix psql zabbix
```
Проверка: убедитесь, что таблицы созданы:
```bash
sudo -u zabbix psql zabbix -c "\dt" | head -20
```
> [!IMPORTANT]
> В выводе должны быть таблицы users, hosts, items. Если их нет — импорт не выполнился, повторите шаг с правильным путём.

**6. Настройка пароля в конфигурации Zabbix Server**
```bash
sudo nano /etc/zabbix/zabbix_server.conf
```
Найдите закомментированную строку # DBPassword= и замените её на:
```text
DBPassword=ваш_пароль
```
Сохраните файл (Ctrl+O, Enter, Ctrl+X).

**7. Настройка PHP-FPM для Zabbix**

При установке пакета zabbix-frontend-php в /etc/php/8.5/fpm/pool.d/ создаётся символическая ссылка zabbix-php-fpm.conf на /etc/zabbix/php-fpm.conf. Проверим, что там указано:
```bash
cat /etc/zabbix/php-fpm.conf | grep -E "^(listen|user|group)"
```
Убедитесь, что строка listen указывает на сокет:
```text
listen = /var/run/php/zabbix.sock
```
Если там что-то другое (например, TCP-порт), исправьте:
```bash
sudo nano /etc/zabbix/php-fpm.conf
```
**8. Создание директории для сокета**

PHP-FPM не сможет создать сокет, если директории /var/run/php/ не существует:
```bash
sudo mkdir -p /var/run/php
sudo chown www-data:www-data /var/run/php
sudo chmod 755 /var/run/php
```
**9. Включение модулей Apache для работы с PHP-FPM**
```bash
sudo a2enmod proxy proxy_fcgi setenvif
```
> [!IMPORTANT]
> Отключите mod_php, если он был установлен ранее:
```bash
sudo a2dismod php8.5 2>/dev/null
sudo a2dismod mpm_prefork 2>/dev/null
sudo a2enmod mpm_event
```
Пояснение: PHP-FPM работает через mpm_event (или mpm_worker), а не через mpm_prefork, который нужен для mod_php. Смена MPM — обязательный шаг для корректной работы PHP-FPM.

**10. Активация конфигурации PHP-FPM для Apache**
```bash
sudo a2enconf php8.5-fpm
```
Эта команда подключает стандартный конфиг Apache, который перенаправляет PHP-запросы в PHP-FPM.

**11. Устранение предупреждения Apache о ServerName**
```bash
echo "ServerName localhost" | sudo tee -a /etc/apache2/apache2.conf
```
**12. Проверка конфига Zabbix для Apache**
Убедитесь, что в конфиге Zabbix прописан правильный обработчик для PHP-FPM:
```bash
cat /etc/apache2/conf-enabled/zabbix.conf | grep -A 3 "FilesMatch"
```
Там должно быть:
```apache
<FilesMatch \.(php|phar)$>
    SetHandler "proxy:unix:/var/run/php/zabbix.sock|fcgi://localhost"
</FilesMatch>
```
Именно этот вариант нам и нужен — он соответствует PHP-FPM. Ничего менять не надо.

**13. Запуск служб**
```bash
sudo systemctl daemon-reload
sudo systemctl restart php8.5-fpm
sudo systemctl restart apache2 zabbix-server zabbix-agent
sudo systemctl enable php8.5-fpm apache2 zabbix-server zabbix-agent
```
Проверка статуса:
```bash
sudo systemctl status php8.5-fpm apache2 zabbix-server zabbix-agent --no-pager
```
Все четыре должны быть active (running).

Проверьте, что сокет создан:
```bash
ls -la /var/run/php/
```
Вы должны увидеть zabbix.sock. Если его нет — смотрите лог PHP-FPM:
```bash
sudo journalctl -xeu php8.5-fpm.service --no-pager | tail -n 30
```
> [!WARNING]
> Типичные ошибки:
> - relation "users" does not exist → схема БД не импортирована (шаг 5)
> - database is not a Zabbix database → то же самое
> - No such file or directory для сокета → проверьте права на /var/run/php/ (шаг 8)

**14. Проверка работы PHP-FPM**

Создайте тестовый файл вне папки Zabbix:
```bash
echo '<?php echo "PHP-FPM WORKS"; ?>' | sudo tee /var/www/html/test.php
```
Откройте http://localhost:8080/test.php (или http://<IP>/test.php).

Если видите «PHP-FPM WORKS» — PHP-FPM работает, переходите к шагу 15.

Если видите исходный код — смотрите лог Apache и PHP-FPM.

Удалите тестовый файл:
```bash
sudo rm /var/www/html/test.php
```
**15. Веб-установка Zabbix**

Откройте в браузере:
```text
http://localhost:8080/zabbix
```
Если видите исходный код PHP — значит, обработчик для папки Zabbix всё ещё неправильный. Вернитесь к шагу 12 и проверьте zabbix.conf.

Пройдите мастер установки:
1. Welcome → выберите язык → Next step
2. Check of pre-requisites → все пункты должны быть OK → Next ste
3. Configure DB connection:
   - Database type: PostgreSQL
   - Database host: localhost
   - Database port: 0
   - Database name: zabbix
   - User: zabbix
   - Password: ваш пароль
   - Снимите галочку Database TLS encryption (для localhost TLS не нужен)
   - Settings: имя сервера (например, Zabbix Server), часовой пояс (например, Europe/Moscow)
4. Pre-installation summary → Next step → Finish

**16. Вход в админку**

Логин: Admin

Пароль: zabbix

## 2. Включение русского языка в интерфейсе Zabbix

По умолчанию в Zabbix многие языки отображаются серыми и недоступны для выбора. Это не баг Zabbix, а отсутствие соответствующей системной локали в Ubuntu. Чтобы активировать русский язык, нужно сгенерировать локаль на уровне ОС.

**1. Установите пакет русской локали**
```bash
sudo apt install language-pack-ru-base -y
```
Этот пакет специально создан для Ubuntu 26.04 и содержит все необходимые данные.

**2. Переконфигурируйте локали**
```bash
sudo dpkg-reconfigure locales
```
Откроется текстовое меню. В нём:
- Найдите в списке ru_RU.UTF-8 UTF-8
- Переместитесь к ней стрелками
- Нажмите Пробел, чтобы поставить звёздочку * (выбор)
- Нажмите Tab до кнопки <Ok> и нажмите Enter
- На следующем экране выберите локаль по умолчанию — стрелками выберите ru_RU.UTF-8 и нажмите Enter.

**3. Проверьте, что локаль появилась**
```bash
locale -a | grep ru_RU
```
Должно вывести ru_RU.utf8.

**4. Перезапустите Apache и PHP-FPM**
```bash
sudo systemctl restart apache2 php8.5-fpm
```
**5. Обновите страницу Zabbix**

В браузере нажмите Ctrl + F5 (жёсткое обновление без кеша). Язык Russian (ru_RU) должен стать активным.

Если язык всё ещё серый

Проверьте, подключено ли расширение gettext в PHP:
```bash
php -m | grep gettext
```
Если команда ничего не выведет — установите его:
```bash
sudo apt install php8.5-gettext -y
sudo systemctl restart php8.5-fpm
```
После этого снова обновите страницу Zabbix через Ctrl + F5.

> [!TIP]
> Примечание: сменить язык интерфейса можно в любой момент после входа в систему — в настройках профиля пользователя (User settings → Profile → Language) или в разделе Administration → General → GUI.

## 3. Установка и подключение Zabbix Agent
Вариант А: агент на самом Zabbix Server
```bash
sudo nano /etc/zabbix/zabbix_agentd.conf
```
Найдите Server=127.0.0.1 и измените на:
```text
Server=127.0.0.1,192.168.0.138
```
Перезапустите агент:
```bash
sudo systemctl restart zabbix-agent
```
Вариант Б: агент на второй виртуальной машине

Выполните на втором хосте:
```bash
# Подключение репозитория
wget https://repo.zabbix.com/zabbix/7.4/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.4+ubuntu26.04_all.deb
sudo dpkg -i zabbix-release_latest_7.4+ubuntu26.04_all.deb
sudo apt update

# Установка агента
sudo apt install zabbix-agent -y
```
Настройте конфигурацию:
```bash
sudo nano /etc/zabbix/zabbix_agentd.conf
```
Измените Server= на IP вашего Zabbix Server:
```text
Server=192.168.0.138
```
Перезапустите и включите автозагрузку:
```bash
sudo systemctl restart zabbix-agent
sudo systemctl enable zabbix-agent
```
Добавление хостов в веб-интерфейсе
> [!IMPORTANT]
> Важный момент: в Zabbix 7.4 шаблон Linux by Zabbix agent находится внутри группы Templates/Operating systems. Чтобы добавить его хосту:

1. Перейдите в Data collection → Hosts (или Configuration → Hosts) → Create host.
2. Заполните поля:
- Хост 1 (Zabbix Server):
- Host name: Zabbix server
- Templates: нажмите Select, в поле Name (не в поле Template group!) введите Linux by Zabbix agent и выберите его
- Host groups: Zabbix servers (или Linux servers)
- Interfaces: Agent → IP: 127.0.0.1, Port: 10050

Хост 2 (вторая ВМ):
- Host name: имя второй ВМ (например, VM-2)
- Templates: Linux by Zabbix agent
- Host groups: Virtual machines или Linux servers
- Interfaces: Agent → IP: реальный IP второй ВМ, Port: 10050

> [!WARNING]
> Стандартный порт Zabbix Agent — 10050, а не 10060. Если указать неверный порт, сервер не сможет подключиться к агенту.

Проверка данных

Перейдите в Monitoring → Latest data, выберите хост — данные должны обновляться (колонка Last check показывает недавнее время).

Проверка лога агента
```bash
cat /var/log/zabbix/zabbix_agentd.log
```
Что должно получиться в итоге

После добавления хостов страница Configuration → Hosts должна выглядеть так:

| Name | Interface | Availability | Status |
|---|---|---|---|
| VM-2 | 10.0.2.4:10050 | 🟢 ZBX | Enabled |
| Zabbix server | 127.0.0.1:10050 | 🟢 ZBX | Enabled |

> [!TIP]
> Зелёный ZBX означает, что Zabbix Server успешно связался с агентом и собирает данные. Количество Items (метрик) отображается в отдельной колонке — у Linux-хоста обычно 40–150 items в зависимости от шаблона.

## 4. Все команды для README.md
```bash
# === Задание 1: Zabbix Server ===
sudo apt update && sudo apt upgrade -y
sudo apt install postgresql -y

# Репозиторий Zabbix для Ubuntu 26.04
wget https://repo.zabbix.com/zabbix/7.4/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.4+ubuntu26.04_all.deb
sudo dpkg -i zabbix-release_latest_7.4+ubuntu26.04_all.deb
sudo apt update

# Установка компонентов (с PHP-FPM вместо mod_php)
sudo apt install zabbix-server-pgsql zabbix-frontend-php php8.5-pgsql php8.5-fpm libapache2-mod-fcgid zabbix-apache-conf zabbix-sql-scripts zabbix-agent -y

# Создание БД и пользователя
sudo -u postgres createuser --pwprompt zabbix
sudo -u postgres createdb -O zabbix zabbix

# Импорт схемы (путь проверить через find)
find /usr/share -name "server.sql.gz" 2>/dev/null
zcat /usr/share/zabbix/sql-scripts/postgresql/server.sql.gz | sudo -u zabbix psql zabbix

# Настройка DBPassword в /etc/zabbix/zabbix_server.conf

# Настройка PHP-FPM
cat /etc/zabbix/php-fpm.conf | grep -E "^(listen|user|group)"
sudo mkdir -p /var/run/php
sudo chown www-data:www-data /var/run/php
sudo chmod 755 /var/run/php

# Включение модулей Apache
sudo a2enmod proxy proxy_fcgi setenvif
sudo a2dismod php8.5 2>/dev/null
sudo a2dismod mpm_prefork 2>/dev/null
sudo a2enmod mpm_event
sudo a2enconf php8.5-fpm

# ServerName для Apache
echo "ServerName localhost" | sudo tee -a /etc/apache2/apache2.conf

# Проверка обработчика PHP в конфиге Zabbix
grep -A 3 "FilesMatch" /etc/apache2/conf-enabled/zabbix.conf

# Запуск служб
sudo systemctl daemon-reload
sudo systemctl restart php8.5-fpm apache2 zabbix-server zabbix-agent
sudo systemctl enable php8.5-fpm apache2 zabbix-server zabbix-agent

# Проверка сокета
ls -la /var/run/php/

# === Русская локаль ===
sudo apt install language-pack-ru-base -y
sudo dpkg-reconfigure locales  # выбрать ru_RU.UTF-8
locale -a | grep ru_RU
sudo systemctl restart apache2 php8.5-fpm

# При необходимости — gettext
php -m | grep gettext
sudo apt install php8.5-gettext -y
sudo systemctl restart php8.5-fpm

# === Задание 2: Агенты ===
# Настройка агента на сервере
sudo nano /etc/zabbix/zabbix_agentd.conf  # Server=127.0.0.1,<IP>
sudo systemctl restart zabbix-agent

# Установка агента на втором хосте
wget https://repo.zabbix.com/zabbix/7.4/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.4+ubuntu26.04_all.deb
sudo dpkg -i zabbix-release_latest_7.4+ubuntu26.04_all.deb
sudo apt update
sudo apt install zabbix-agent -y
sudo nano /etc/zabbix/zabbix_agentd.conf  # Server=<IP>
sudo systemctl restart zabbix-agent
sudo systemctl enable zabbix-agent

# Проверка лога
cat /var/log/zabbix/zabbix_agentd.log
```
## 5. Задание 3 (со звёздочкой): Zabbix Agent на Windows

1. Скачайте MSI-пакет Zabbix Agent для Windows с официального сайта.
2. Запустите установщик, укажите IP вашего Zabbix Server.
3. В веб-интерфейсе создайте хост с шаблоном Windows by Zabbix agent.
4. Перейдите в Monitoring → Latest data, выберите хост и проверьте данные по диску C:.
