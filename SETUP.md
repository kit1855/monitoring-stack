# Установка и настройка мониторинга

Пошаговая инструкция по развёртыванию Prometheus, Grafana и Alertmanager.

## 1. Node Exporter

### 1.1 Установка

```bash
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
tar xvf node_exporter-1.7.0.linux-amd64.tar.gz
sudo mv node_exporter-1.7.0.linux-amd64/node_exporter /usr/local/bin/

```

### 1.2 Создание сервиса

```bash
sudo vim /etc/systemd/system/node_exporter.service
```

Содержимое файла:

```ini
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=nobody
Group=nogroup
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

### 1.3 Запуск

```bash
sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter
sudo systemctl status node_exporter
```

## 2. Prometheus

### 2.1 Установка

```bash
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v2.53.0/prometheus-2.53.0.linux-amd64.tar.gz
tar xvf prometheus-2.53.0.linux-amd64.tar.gz
sudo mv prometheus-2.53.0.linux-amd64 /etc/prometheus
```

### 2.2 Создание пользователя

```bash
sudo useradd --no-create-home --shell /bin/false prometheus
sudo chown -R prometheus:prometheus /etc/prometheus
```

### 2.3 Создание сервиса

```bash
sudo vim /etc/systemd/system/prometheus.service
```

Содержимое файла:

```ini
[Unit]
Description=Prometheus
After=network.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/etc/prometheus/prometheus --config.file=/etc/prometheus/prometheus.yml --storage.tsdb.path=/etc/prometheus/data

[Install]
WantedBy=multi-user.target
```

### 2.4 Конфигурация

```bash
sudo mkdir -p /etc/prometheus/data
sudo chown -R prometheus:prometheus /etc/prometheus
sudo vim /etc/prometheus/prometheus.yml
```

Содержимое файла:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
```

### 2.5 Запуск

```bash
sudo systemctl daemon-reload
sudo systemctl start prometheus
sudo systemctl enable prometheus
sudo systemctl status prometheus
```

### 2.6 Проверка

Откройте в браузере: `http://serv123.ru:9090`

## 3. Alertmanager

### 3.1 Установка

```bash
cd /tmp
wget https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.27.0.linux-amd64.tar.gz
tar xvf alertmanager-0.27.0.linux-amd64.tar.gz
sudo mv alertmanager-0.27.0.linux-amd64 /etc/alertmanager
```

### 3.2 Создание пользователя

```bash
sudo useradd --no-create-home --shell /bin/false alertmanager
sudo chown -R alertmanager:alertmanager /etc/alertmanager
```

### 3.3 Создание сервиса

```bash
sudo vim /etc/systemd/system/alertmanager.service
```

Содержимое файла:

```ini
[Unit]
Description=Alertmanager
After=network.target

[Service]
User=alertmanager
Group=alertmanager
Type=simple
ExecStart=/etc/alertmanager/alertmanager --config.file=/etc/alertmanager/alertmanager.yml --storage.path=/etc/alertmanager/data

[Install]
WantedBy=multi-user.target
```

### 3.4 Конфигурация

```bash
sudo mkdir -p /etc/alertmanager/data
sudo chown -R alertmanager:alertmanager /etc/alertmanager
sudo vim /etc/alertmanager/alertmanager.yml
```

Содержимое файла (замените пароль приложения):

```yaml
global:
  smtp_smarthost: 'smtp.mail.ru:587'
  smtp_from: 'ваша_почта@mail.ru'
  smtp_auth_username: 'ваша_почта@mail.ru'
  smtp_auth_password: 'пароль_приложения'
  smtp_require_tls: true

route:
  receiver: 'email'
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h

receivers:
- name: 'email'
  email_configs:
  - to: 'ваша_почта@mail.ru'
```

### 3.5 Запуск

```bash
sudo systemctl daemon-reload
sudo systemctl start alertmanager
sudo systemctl enable alertmanager
sudo systemctl status alertmanager
```

### 3.6 Проверка

Откройте в браузере: `http://serv123.ru:9093`

## 4. Настройка Prometheus для Alertmanager

### 4.1 Добавление секции alerting

```bash
sudo vim /etc/prometheus/prometheus.yml
```

Добавьте в конец файла (перед строкой `scrape_configs`):

```yaml
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['127.0.0.1:9093']

rule_files:
  - "/etc/prometheus/alerts.yml"
```

### 4.2 Создание правил алертов

```bash
sudo vim /etc/prometheus/alerts.yml
```

Содержимое файла:

```yaml
groups:
  - name: example_alerts
    interval: 30s
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is above 80% for more than 5 minutes."

      - alert: DiskUsageHigh
        expr: (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 < 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Low disk space on {{ $labels.instance }}"
          description: "Only {{ $value }}% disk space left on root partition."

      - alert: ServiceDown
        expr: up{job="node_exporter"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.job }} is down on {{ $labels.instance }}"
          description: "The service has been unreachable for more than 1 minute."
```

### 4.3 Перезапуск Prometheus

```bash
sudo systemctl restart prometheus
```

### 4.4 Проверка

Откройте в браузере:
- `http://serv123.ru:9090/targets` (должен быть alertmanager UP)
- `http://serv123.ru:9090/alerts` (должны быть правила алертов)
- `http://serv123.ru:9093/#/alerts` (должны отображаться алерты)

## 5. Grafana

### 5.1 Установка через Docker

```bash
sudo docker run -d \
  --name=grafana \
  -p 3001:3000 \
  --restart=unless-stopped \
  grafana/grafana-enterprise:latest
```

### 5.2 Проверка

```bash
sudo docker ps | grep grafana
```

### 5.3 Открыть порт в UFW

```bash
sudo ufw allow 3001/tcp
sudo ufw reload
```

### 5.4 Вход в Grafana

Откройте в браузере: `http://serv123.ru:3001`

**Логин:** `admin`  
**Пароль:** `admin`

При первом входе система попросит придумать новый пароль.

### 5.5 Подключение Prometheus к Grafana

1. В левом меню нажмите **Connections** → **Data sources**
2. Нажмите **Add data source**
3. Выберите **Prometheus**
4. В поле **URL** введите: `http://localhost:9090`
5. Нажмите **Save & Test**

### 5.6 Импорт дашборда Node Exporter

1. В левом меню нажмите **Dashboards** → **Import**
2. В поле **Import via grafana.com** введите ID: `1860`
3. Нажмите **Load**
4. Выберите источник данных (Prometheus)
5. Нажмите **Import**

## 6. Тестовый алерт

### 6.1 Добавление тестового правила

```bash
sudo vim /etc/prometheus/alerts.yml
```

Добавьте в конец файла (перед последней строкой `groups`):

```yaml
      - alert: TestAlert
        expr: vector(1) == 1
        for: 0s
        labels:
          severity: test
        annotations:
          summary: "Test alert"
          description: "This is a test message to check email delivery."
```

### 6.2 Перезапуск Prometheus

```bash
sudo systemctl restart prometheus
```

### 6.3 Проверка

Подождите 30 секунд и откройте Alertmanager:

`http://serv123.ru:9093/#/alerts`

Должен появиться `TestAlert` в статусе `firing`.

### 6.4 Проверка почты

На указанный email должно прийти письмо с уведомлением.

### 6.5 Удаление тестового алерта (опционально)

```bash
sudo vim /etc/prometheus/alerts.yml
```

Удалите блок с `TestAlert`, затем:

```bash
sudo systemctl restart prometheus
```


