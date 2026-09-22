# Домашнее задание к занятию "Практическое задание «Конфигурация почтового сервиса»" - Костенко Максим

## 1. Установка компонентов почтового сервера

**1.1. Установка модулей**

Где: ispmanager → Конфигурация ПО → Почтовый сервер (SMTP/POP3/IMAP).

Устанавливаем:

| Компонент | Назначение |
|---|---|
| Exim | Отправка почты (SMTP) |
| Dovecot (POP3/IMAP) | Хранение и получение почты |
| OpenDKIM (exim) | Цифровая подпись писем |
| SpamAssassin | Фильтр спама |
| ClamAV | Антивирус для писем |
| Sieve | Сортировщик писем и фильтры |
| Roundcube | Веб-клиент для работы с почтой |

Нажимаем Установить / Сохранить.

<img width="208" height="338" alt="Image" src="https://github.com/user-attachments/assets/2fb5a539-771b-41f0-a624-1859fa41fcc2" />

**1.2. Создание почтового домена**

Где: ispmanager → Почта → Почтовые домены → Создать почтовый домен.

Заполняем:

| Поле | Значение |
|---|---|
| Имя | mk-industrias.online |
| Владелец | пользователь-владелец |
| IP-адрес | 192.168.10.32 (внутренний IP web-a) |
| Действие по умолчанию | «Сообщение об ошибке» |
| Включить DKIM | ✅ |
| Включить DMARC | ✅ |
| Защищённое соединение SSL | ✅ |
| SSL-сертификат | «Новый SSL Let's Encrypt» |
| Псевдоним для сертификата | mail.mk-industrias.online |
| Email | ваш контактный email |

Нажимаем Создать.

<img width="547" height="1036" alt="Image" src="https://github.com/user-attachments/assets/ecf9094b-ff57-4cc7-9c05-23c6f6c306c9" />

**1.3. Создание почтового ящика**

Где: ispmanager → Почта → Создать почтовый ящик.

| Поле | Значение |
|---|---|
| Имя | info |
| Домен | mk-industrias.online |
| Пароль | задайте надёжный пароль |
| Дисковая квота | 500 МБ |

Нажимаем Создать.

<img width="468" height="642" alt="Image" src="https://github.com/user-attachments/assets/23f4b81b-9a64-4ab6-b745-6acba44a9641" />

## 2. Настройка DNS-записей у регистратора

**2.1. Где настраивать**

NS домена mk-industrias.online = ns1.reg.ru, все внешние DNS-записи добавляются в панели Reg.ru.

> [!WARNING]
> - ispmanager создаёт локальную зону, которая не видна из интернета.
> - Поэтому все записи, которые должны быть видны снаружи (A, MX, TXT, DKIM, DMARC, SPF), нужно дублировать у регистратора.

**2.2. Зайди в панель Reg.ru**

Открой reg.ru → личный кабинет → найди mk-industrias.online → Управление зоной.

**2.3. Добавление DNS записей**

| Тип | Имя | Значение | Для чего |
|---|---|---|---|
| A | @ | \<PUBLIC_IP\> | Основной IP: сайт и приём почты |
| A | mail | \<PUBLIC_IP\> | IP почтового сервера (на него указывает MX) |
| A | pop | \<PUBLIC_IP\> | Получение почты по POP3 |
| A | smtp | \<PUBLIC_IP\> | Отправка почты по SMTP |
| A | www | \<PUBLIC_IP\> | Доступ к сайту по www.mk-industrias.online |
| MX | @ | mail.mk-industrias.online (приоритет 10) | Куда доставлять входящую почту |
| TXT | @ | v=spf1 a mx ip4:\<PUBLIC_IP\> ~all | SPF: кто имеет право отправлять от имени домена |
| TXT | dkim._domainkey | (публичный ключ из ispmanager) | DKIM: проверка подписи писем |
| TXT | _dmarc | v=DMARC1; p=quarantine; rua=mailto:info@mk-industrias.online | Политика для писем без SPF/DKIM |

> [!IMPORTANT]
> - PTR-запись настраивается у хостера, который выдал вам <PUBLIC_IP> (не в Reg.ru!).
> - Значение PTR = mail.mk-industrias.online. Без PTR Gmail/Яндекс будут отклонять письма.

**2.4. Настройка PTR-записи (Yandex Cloud)**

PTR-запись настраивается не в Reg.ru, а на стороне хостера публичного IP. 

В нашем случае сервер находится в Yandex Cloud, поэтому используем yc CLI.

> [!IMPORTANT]
> A/MX/TXT-записи домена `mk-industrias.online` обслуживаются NS-серверами Reg.ru — домен делегирован на них. Публичная DNS-зона в YC используется **только** как «контейнер» для привязки PTR-записи к IP. Если делегировать домен на NS Yandex Cloud, все A/MX/TXT-записи придётся переносить в YC.

> [!WARNING]
> - PTR можно задать только для статического (reserved) IP-адреса. Для динамического (ephemeral) YC выдаёт ошибку:
> - Address ... is dynamic (ephemeral). DNS settings should be set for the Compute instance network interface, not for the Address itself.

a. Проверить текущий статус IP
```bash
yc vpc address list
```
Найти строку, где `EXTERNAL_IPV4_ADDRESS` совпадает с `<PUBLIC_IP>`, и запомнить её `ID` — это и есть `<ID_IP_адреса>`.

В нашем случае публичный IP <ID_IP_адреса> имеет RESERVED = false (динамический).

b. Сделать IP статическим
```bash
yc vpc address update --reserved=true <ID_IP_адреса>
```
Проверьте, что RESERVED стал true:
```bash
yc vpc address list
```
Теперь адрес закреплён за вами и не изменится при перезапуске ВМ.

c. Убедиться, что публичная DNS-зона создана
```bash
yc dns zone list
```
Нужна зона с VISIBILITY = PUBLIC. Если её нет — создайте:
```bash
yc dns zone create \
  --name mk-industrias-public \
  --zone mk-industrias.online. \
  --public-visibility
```
> [!IMPORTANT]
> Создание публичной DNS-зоны в YC — платная услуга (тарифицируется по правилам YC).

d. Привязать PTR к статическому IP
```bash
yc vpc address update <ID_IP_адреса> \
  --dns-record ptr=true,fqdn=mail.mk-industrias.online.,dns-zone=<ID_DNS_зоны>
```
> [!WARNING]
> Точка в конце fqdn обязательна: mail.mk-industrias.online. — иначе имя будет воспринято как поддомен относительно зоны.

Разбор параметров:

| Параметр | Значение | Пояснение |
|---|---|---|
| \<ID_IP_адреса\> | \<ID_IP_адреса\> | ID публичного IP из `yc vpc address list` |
| ptr=true | — | Включаем PTR-запись |
| fqdn | mail.mk-industrias.online. | Имя, на которое будет резолвиться IP |
| dns-zone | \<ID_DNS_зоны\> | ID публичной DNS-зоны из `yc dns zone list` |

e. Проверка
```bash
dig +short -x <PUBLIC_IP>
# ожидаем: mail.mk-industrias.online.
```

## 3. Выпуск SSL-сертификата

**3.1. Выпуск**

Где: ispmanager → SSL-сертификаты → Создать → Let's Encrypt.

| Поле | Значение |
|---|---|
| Домены | mk-industrias.online, mail.mk-industrias.online |
| Email | ваш email |

Нажимаем Выпустить.

<img width="2228" height="218" alt="Image" src="https://github.com/user-attachments/assets/4c0e0f4d-e78a-49af-a580-a15319a152ef" />

**3.2. Применение**

Где: Почта → Почтовые домены → mk-industrias.online → Изменить.

В поле SSL-сертификат выбери выпущенный. Сохрани.

<img width="448" height="157" alt="Image" src="https://github.com/user-attachments/assets/7cccad76-3015-4c05-beb5-01978d44aff5" />

## 4. Проверка почты онлайн-инструментами
```bash
# A-записи
dig +short mk-industrias.online A
dig +short mail.mk-industrias.online A
dig +short pop.mk-industrias.online A
dig +short smtp.mk-industrias.online A

# MX
dig +short mk-industrias.online MX

# TXT: SPF, DKIM, DMARC
dig +short mk-industrias.online TXT
dig +short dkim._domainkey.mk-industrias.online TXT
dig +short _dmarc.mk-industrias.online TXT

# NS и SOA
dig +short mk-industrias.online NS
dig +short mk-industrias.online SOA

# Обратная зона (PTR) — проверяем для <PUBLIC_IP>
dig +short -x <PUBLIC_IP>

# Универсальные проверки
host mk-industrias.online
host mail.mk-industrias.online

# Доступность HTTPS
curl -fsSIL --max-time 5 https://mk-industrias.online/
```
## 5. Настройка фильтров и автоответчика

**5.1. Автоответчик**

Где: ispmanager → Почта → info@mk-industrias.online → Автоответчик.

| Поле | Значение |
|---|---|
| Тема | «Автоответ» |
| Текст | текст автоматического ответа |

Нажимаем Включить / Сохранить.

<img width="559" height="721" alt="Image" src="https://github.com/user-attachments/assets/34d13e73-2156-4a41-9ea5-3e6c104c3855" />

**5.2. Фильтр**

**a. Новый почтовый сортировщик**

| Поле | Что указать | Пояснение |
|---|---|---|
| Имя почтового фильтра | delete_spam | Произвольное имя фильтра. Лучше — латиницей, без пробелов |
| Сопоставление условий | Все истинны | Для одного условия разницы нет. Оставь по умолчанию |
| Расположить перед | ***поместить в конец*** | Оставь по умолчанию — фильтр будет последним в списке |

<img width="753" height="448" alt="Image" src="https://github.com/user-attachments/assets/7dae053c-9788-493a-a182-1d9ed7c01181" />

Нажимаем «Далее».

**b. Новое условие сортировки**

| Поле | Что указать | Пояснение |
|---|---|---|
| Условие 1 | тема письма | Фильтр проверяет именно тему входящего письма |
| Не (галочка) | не отмечено | Без галочки — значит «содержит». С галочкой было бы «не содержит» |
| Условие | содержит | Проверяем, есть ли в теме слово «спам» |
| Значения | спам | Ключевое слово, по которому срабатывает фильтр |
| Условие 2 | не выбрано | Второе условие не нужно |

<img width="749" height="544" alt="Image" src="https://github.com/user-attachments/assets/19679cac-17c2-4746-813c-489bf2b8aee5" />

Нажимаем «Далее».

**c. Новое действие сортировки**

| Поле | Что указать | Пояснение |
|---|---|---|
| Действие 1 | сохранить в директорию | Junk |

Нажимаем «Завершить».

Логика фильтра
```text
ЕСЛИ тема письма СОДЕРЖИТ "спам"
ТО сохранить в директорию Junk (спам)
```
Письмо, у которого в теме есть слово «спам», не попадёт во «Входящие» — Exim/Sieve сохранит его сразу при приёме в Junk (спам).

**5.3. Проверка через Roundcube**

Проверка через Roundcube

- Отправь письмо с Gmail на info@mk-industrias.online — проверь автоответ.
- Отправь письмо с темой «спам» — проверь фильтр.

Где: ispmanager → Почта → info@mk-industrias.online → Почтовый клиент.

1. Письма в Roundcube, отфильтрованные.

<img width="1118" height="284" alt="Image" src="https://github.com/user-attachments/assets/42160787-772c-45a2-b331-c248bd1fd0c5" />

2. Письмо с автоответчиком.

Проверка локальной почты при блокировке порта 25

ispmanager → IP-адреса → NAT — включи, укажи:

Внутренний: 192.168.10.32

Внешний: 127.0.0.1

Сохрани.

<img width="472" height="302" alt="Image" src="https://github.com/user-attachments/assets/305ce9e9-06da-4670-a462-ef0bc3112f70" />

Создай 2 ящика: user1@mk-industrias.online и user2@mk-industrias.online.

<img width="2223" height="86" alt="Image" src="https://github.com/user-attachments/assets/bb23fea3-7d12-4f2a-b734-0b779e7dfa4c" />

Отправь письмо с user1 на user2 через Roundcube. Потом ответь обратно. Оба должны дойти.

<img width="1200" height="288" alt="Image" src="https://github.com/user-attachments/assets/7b962608-59dc-42a0-b26c-6d653776ff7c" />

Проверка автоответчика, было получено сообщение в ответ

<img width="1618" height="294" alt="Image" src="https://github.com/user-attachments/assets/f1c7fa22-f8b2-429f-88bf-180e7019e032" />

Проверь автоответчик и фильтр — оба работают через локальный обмен.
