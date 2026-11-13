# Установка и настройка

## Задание 1. Установка Prometheus

1. Создание пользователя
```bash
sudo useradd --no-create-home --shell /bin/false prometheus
```
2. Скачивание и размещение файлов

Перейдите на страницу официальных загрузок Prometheus и выберите linux-amd64 архив. В качестве примера возьмём стабильную версию (замените на актуальный номер версии):
```bash
wget https://github.com/prometheus/prometheus/releases/download/v3.15.1/prometheus-3.15.1.linux-amd64.tar.gz
tar xvfz prometheus-3.15.1.linux-amd64.tar.gz
sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo cp prometheus-3.15.1.linux-amd64/prometheus /usr/local/bin/
sudo cp prometheus-3.15.1.linux-amd64/promtool /usr/local/bin/
sudo cp -r prometheus-3.15.1.linux-amd64/consoles /etc/prometheus/
sudo cp -r prometheus-3.15.1.linux-amd64/console_libraries /etc/prometheus/
sudo cp prometheus-3.15.1.linux-amd64/prometheus.yml /etc/prometheus/
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
sudo chown prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool
```
3. Создание сервиса systemd

Создайте /etc/systemd/system/prometheus.service:
```ini
[Unit]
Description=Prometheus Service Netology Lesson 9.4 — [Ваши ФИО]
After=network.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
```
4. Запуск и проверка
```bash
sudo systemctl daemon-reload
sudo systemctl start prometheus
sudo systemctl status prometheus
```

<img width="2531" height="444" alt="image" src="https://github.com/user-attachments/assets/1c997c40-55cd-4d3e-91ed-54805ec8f6dd" />

## Задание 2. Установка Node Exporter

1. Скачивание и размещение

В официальном руководстве Prometheus приведён пример загрузки Node Exporter :
```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.10.2/node_exporter-1.10.2.linux-amd64.tar.gz
tar xvfz node_exporter-1.10.2.linux-amd64.tar.gz
sudo cp node_exporter-1.10.2.linux-amd64/node_exporter /usr/local/bin/
sudo useradd --no-create-home --shell /bin/false node_exporter
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
```
2. Создание сервиса systemd

Создайте /etc/systemd/system/node-exporter.service:
```ini
[Unit]
Description=Node Exporter Netology Lesson 9.4 — [Ваши ФИО]
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```
3. Запуск и проверка
```bash
sudo systemctl daemon-reload
sudo systemctl start node-exporter
sudo systemctl status node-exporter
curl http://localhost:9100/metrics
```

<img width="1532" height="423" alt="image" src="https://github.com/user-attachments/assets/8ed290d6-3ed3-4cd4-83f0-d0abce6a51e9" />

## Задание 3. Подключение Node Exporter к Prometheus

1. Редактирование конфигурации

Согласно официальной документации Prometheus, добавьте задание node в scrape_configs . Отредактируйте /etc/prometheus/prometheus.yml:
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']
```
2. Перезапуск и проверка
```bash
sudo systemctl restart prometheus
sudo systemctl status prometheus
```
3. Скриншоты в интерфейсе

Откройте http://<IP-сервера>:9090:
- Status > Configuration: убедитесь, что в конфигурации присутствует задание node с target localhost:9100.

<img width="321" height="598" alt="image" src="https://github.com/user-attachments/assets/7854ad33-ff83-4b8f-b3bc-271a44b8e3dd" />

- Status > Targets: должны отображаться как минимум два эндпоинта — prometheus (localhost:9090) и node (localhost:9100), оба в состоянии UP.

<img width="2550" height="438" alt="image" src="https://github.com/user-attachments/assets/c35f3a58-36b0-40d0-ba18-b57196cbe4b5" />

## Задание 4* . Установка Grafana

Перейдите на официальную страницу загрузки Grafana и выберите версию OSS для своей платформы. В качестве примера для Linux (Debian/Ubuntu):
```bash
sudo apt-get install -y adduser libfontconfig1 musl
wget https://dl.grafana.com/oss/release/grafana_12.4.0_amd64.deb
sudo dpkg -i grafana_12.4.0_amd64.deb
sudo systemctl start grafana-server
sudo systemctl status grafana-server
```
<img width="680" height="391" alt="image" src="https://github.com/user-attachments/assets/790d35c2-24a4-4695-8707-63de40239e6d" />

## Задание 5* . Интеграция Grafana и Prometheus

1. В Grafana перейдите в Connections > Data sources > Add data source.
2. Выберите Prometheus.
3. В поле Prometheus server URL укажите http://localhost:9090 (если Grafana и Prometheus на одном хосте) .
4. Нажмите Save & Test, убедитесь, что отображается сообщение об успешном подключении.

<img width="2545" height="287" alt="image" src="https://github.com/user-attachments/assets/94985769-ae56-4721-9126-b1b23db792a6" />
