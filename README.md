# Monitoring Stack

Стек мониторинга на базе Prometheus, Grafana и Alertmanager для сбора метрик, визуализации и отправки алертов.

## Технологии

| Компонент | Технология |
|-----------|------------|
| Сбор метрик | Prometheus |
| Визуализация | Grafana |
| Алертинг | Alertmanager |
| Экспортер метрик | Node Exporter |
| Уведомления | Email (SMTP) |

## Архитектура

Node Exporter (порт 9100) → Prometheus (порт 9090) → Grafana (порт 3001)

Prometheus (порт 9090) → Alertmanager (порт 9093) → Email

## Быстрая проверка

| Сервис | Адрес |
|--------|-------|
| Prometheus | `http://serv123.ru:9090` |
| Grafana | `http://serv123.ru:3001` |
| Alertmanager | `http://serv123.ru:9093` |

### Статус сервера

```bash
sudo systemctl status node_exporter prometheus alertmanager
```

## Скриншоты

| Скриншот | Что показывает |
|----------|----------------|
| [prometheus-targets.png](./screenshots/prometheus-targets.png) | Targets в Prometheus (оба UP) |
| [prometheus-alerts.png](./screenshots/prometheus-alerts.png) | Правила алертов в Prometheus |
| [alertmanager-alerts.png](./screenshots/alertmanager-alerts.png) | Алерты в Alertmanager |
| [grafana-dashboard.png](./screenshots/grafana-dashboard.png) | Дашборд Grafana (Node Exporter) |
| [email-notification.png](./screenshots/email-notification.png) | Email-уведомление |

## Конфигурационные файлы

| Файл | Назначение |
|------|------------|
| [prometheus.yml](./configs/prometheus.yml) | Конфигурация Prometheus |
| [alerts.yml](./configs/alerts.yml) | Правила алертов |
| [alertmanager.yml](./configs/alertmanager.yml) | Конфигурация Alertmanager |

## Установленные алерты

| Алерт | Условие | Действие |
|-------|---------|----------|
| HighCPUUsage | CPU > 80% в течение 5 минут | Email |
| DiskUsageHigh | Свободно < 10% на root-разделе | Email |
| ServiceDown | Node Exporter недоступен 1 минуту | Email |
| TestAlert | Всегда активен (для проверки) | Email |

## Мониторинг

- **Node Exporter** собирает метрики: CPU, RAM, диск, сеть
- **Prometheus** опрашивает Node Exporter каждые 15 секунд
- **Grafana** показывает дашборд (ID 1860 для Node Exporter)
- **Alertmanager** отправляет письма при срабатывании алертов

## Почтовые уведомления

Настроена отправка на `serv123ru@mail.ru` через SMTP-сервер Mail.ru (порт 587, STARTTLS). Используется пароль приложения.
