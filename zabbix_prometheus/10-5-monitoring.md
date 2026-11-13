# Домашнее задание к занятию "Практическое задание с самопроверкой "Система мониторинга Prometheus. Часть 2"" - Костенко Максим

## Задание 1. Создание правила оповещения и статус Pending

1. Создание файла с правилом

Согласно официальной документации Prometheus, правило оповещения описывается в YAML-файле с использованием выражения PromQL . Создайте файл /etc/prometheus/alert.rules.yml:
```yaml
groups:
- name: node_alerts
  rules:
  - alert: NodeExporterDown
    expr: up{job="node"} == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Node Exporter недоступен"
      description: "Node Exporter на {{ $labels.instance }} не отвечает более 1 минуты."
```
Ключевой момент: параметр for: 1m означает, что после того, как условие up{job="node"} == 0 станет истинным, оповещение сначала перейдёт в состояние Pending и будет оставаться в нём в течение 1 минуты, прежде чем стать Firing .

## 2. Подключение правила к Prometheus

Отредактируйте /etc/prometheus/prometheus.yml, добавив секцию rule_files:
```yaml
global:
  scrape_interval: 15s

rule_files:
  - /etc/prometheus/alert.rules.yml

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']
```
Перезапустите Prometheus:
```bash
sudo systemctl restart prometheus
```
3. Остановка Node Exporter и получение статуса Pending
```bash
sudo systemctl stop node-exporter
```
Сразу после остановки откройте в браузере http://<IP>:9090/alerts. Правило NodeExporterDown должно отображаться в состоянии Pending (жёлтый цвет).

<img width="1886" height="758" alt="image" src="https://github.com/user-attachments/assets/ac1651bf-d18e-4124-8600-77bf2360ebff" />

## Задание 2. Установка Alertmanager и интеграция с Prometheus

1. Скачивание и установка Alertmanager

С официальной страницы загрузок Prometheus скачайте Alertmanager (linux-amd64):
```bash
wget https://github.com/prometheus/alertmanager/releases/download/v0.28.1/alertmanager-0.28.1.linux-amd64.tar.gz
tar xvfz alertmanager-0.28.1.linux-amd64.tar.gz
sudo cp alertmanager-0.28.1.linux-amd64/alertmanager /usr/local/bin/
sudo cp alertmanager-0.28.1.linux-amd64/amtool /usr/local/bin/
sudo mkdir -p /etc/alertmanager /var/lib/alertmanager
sudo cp alertmanager-0.28.1.linux-amd64/alertmanager.yml /etc/alertmanager/
sudo useradd --no-create-home --shell /bin/false alertmanager
sudo chown -R alertmanager:alertmanager /etc/alertmanager /var/lib/alertmanager
sudo chown alertmanager:alertmanager /usr/local/bin/alertmanager /usr/local/bin/amtool
```
2. Создание сервиса systemd

Создайте /etc/systemd/system/alertmanager.service:
```ini
[Unit]
Description=Alertmanager Service
After=network.target

[Service]
User=alertmanager
Group=alertmanager
Type=simple
ExecStart=/usr/local/bin/alertmanager \
  --config.file=/etc/alertmanager/alertmanager.yml \
  --storage.path=/var/lib/alertmanager

[Install]
WantedBy=multi-user.target
```
Запуск:
```bash
sudo systemctl daemon-reload
sudo systemctl start alertmanager
sudo systemctl status alertmanager
```
3. Настройка Prometheus для отправки оповещений в Alertmanager

Отредактируйте /etc/prometheus/prometheus.yml, добавив секцию alerting:
```yaml
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['localhost:9093']
```
Перезапустите Prometheus:
```bash
sudo systemctl restart prometheus
```
4. Получение статуса Firing

Убедитесь, что Node Exporter всё ещё остановлен. Подождите более 1 минуты.

В Prometheus (http://<IP>:9090/alerts) правило NodeExporterDown перейдёт в статус Firing (красный цвет).

<img width="2523" height="521" alt="image" src="https://github.com/user-attachments/assets/50d0f3af-2f1f-4841-8151-5ccb0c6a4381" />

Затем откройте Alertmanager (http://<IP>:9093) — на главной странице должен отображаться активный алерт с именем NodeExporterDown и метками экземпляра.

<img width="1164" height="582" alt="image" src="https://github.com/user-attachments/assets/a7cafc37-5911-4892-a09a-dd6a20c0e359" />

## Задание 3. Экспортёр метрик Docker и подключение к Prometheus

1. Запуск экспортёра Docker (cAdvisor)

Официального «docker_exporter» от проекта Prometheus не существует. Стандартный способ сбора метрик Docker-контейнеров — cAdvisor от Google, который рекомендован в документации Prometheus для мониторинга контейнеров . Запустите cAdvisor в Docker:
```bash
docker run -d \
  --name=cadvisor \
  --restart=unless-stopped \
  -p 8080:8080 \
  -v /:/rootfs:ro \
  -v /var/run:/var/run:ro \
  -v /sys:/sys:ro \
  -v /var/lib/docker/:/var/lib/docker:ro \
  -v /dev/disk/:/dev/disk:ro \
  --privileged \
  --device=/dev/kmsg \
  gcr.io/cadvisor/cadvisor:latest
```
2. Проверка эндпоинта

Откройте в браузере http://<IP>:8080/metrics. Должна отображаться страница с метриками в формате Prometheus (множество строк container_*, machine_* и т.д.).

3. Подключение к Prometheus

Добавьте задание в /etc/prometheus/prometheus.yml:
```yaml
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['localhost:8080']
```
Перезапустите Prometheus:
```bash
sudo systemctl restart prometheus
```
Откройте http://<IP>:9090/targets — в списке должны присутствовать три эндпоинта: prometheus, node, cadvisor, все в состоянии UP.

## Задание 4* (со звёздочкой). Дашборд Grafana с метриками Docker и сервера

1. Подключение Prometheus как источника данных

В Grafana перейдите в Connections > Data sources > Add data source > Prometheus. Укажите URL http://localhost:9090 (если Grafana и Prometheus на одном хосте) и сохраните.

2. Импорт готового дашборда или создание своего

Самый быстрый способ получить дашборд с метриками Docker и сервера — импортировать готовый шаблон из официальной библиотеки Grafana. Например, дашборд Docker Container & Host Metrics (ID 24342) использует данные cAdvisor и Prometheus и содержит панели для метрик контейнеров и хоста.

В Grafana перейдите в Dashboards > New > Import, введите ID 24342, выберите источник данных Prometheus и нажмите Import.

Если требуется создать собственный дашборд, можно добавить панели со следующими PromQL-запросами:
- CPU хоста: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
- Память хоста: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
- CPU контейнера: sum(rate(container_cpu_usage_seconds_total{name!=""}[5m])) by (name) * 100
- Память контейнера: container_memory_usage_bytes{name!=""}
