## 1. Prometheus не видит Node Exporter

**Симптом:** В разделе `http://serv123.ru:9090/targets` статус Node Exporter `DOWN`.

**Вероятная причина:** Node Exporter не запущен или остановлен.

**Проверка:**

```bash
sudo systemctl status node_exporter
curl http://localhost:9100/metrics | head
```

**Решение:**

```bash
sudo systemctl restart node_exporter
sudo systemctl enable node_exporter
```

## 2. Prometheus не видит Alertmanager

**Симптом:** В разделе `http://serv123.ru:9090/targets` нет строки `alertmanager`.

**Вероятная причина:** В конфиге Prometheus отсутствует секция `alerting`.

**Проверка:**

```bash
sudo cat /etc/prometheus/prometheus.yml | grep -A 5 alerting
```

**Решение:**

Откройте конфиг:

```bash
sudo vim /etc/prometheus/prometheus.yml
```

Добавьте в конец файла:

```yaml
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['127.0.0.1:9093']
```

Перезапустите Prometheus:

```bash
sudo systemctl restart prometheus
```

## 3. Alertmanager не отправляет email

**Симптом:** Алерты есть в Alertmanager, но письма не приходят на почту.

**Вероятная причина:** Неправильные настройки SMTP в конфиге Alertmanager.

**Проверка:**

```bash
sudo journalctl -u alertmanager -n 30 --no-pager | grep -i mail
```

**Решение:**

Откройте конфиг Alertmanager:

```bash
sudo vim /etc/alertmanager/alertmanager.yml
```

Проверьте и исправьте настройки:

```yaml
global:
  smtp_smarthost: 'smtp.mail.ru:587'
  smtp_from: 'ваша_почта@mail.ru'
  smtp_auth_username: 'ваша_почта@mail.ru'
  smtp_auth_password: 'пароль_приложения'
  smtp_require_tls: true

route:
  receiver: 'email'

receivers:
- name: 'email'
  email_configs:
  - to: 'ваша_почта@mail.ru'
```

Перезапустите Alertmanager:

```bash
sudo systemctl restart alertmanager
```

## 4. Grafana не подключается к Prometheus

**Симптом:** В Grafana при добавлении Data Source ошибка `connection refused` или `Network is unreachable`.

**Вероятная причина:** Неправильный URL Prometheus или Prometheus не запущен.

**Проверка:**

```bash
curl http://localhost:9090
sudo systemctl status prometheus
```

**Решение:**

В Grafana при добавлении Data Source укажите URL:

```text
http://localhost:9090
```

(Не используйте https или 127.0.0.1)

Если Prometheus не запущен:

```bash
sudo systemctl start prometheus
sudo systemctl enable prometheus
```

## 5. Ошибка при импорте дашборда 1860

**Симптом:** Дашборд импортировался, но на графиках нет данных или они пустые.

**Вероятная причина:** Не выбран источник данных Prometheus в настройках дашборда.

**Проверка:**

```bash
curl http://localhost:9090/api/v1/query?query=up
```

**Решение:**

1. В левом меню Grafana нажмите Dashboards
2. Найдите дашборд Node Exporter Full
3. Откройте его и нажмите на иконку шестерёнки (Settings)
4. Перейдите в раздел Variables
5. Проверьте, что во всех переменных выбран правильный источник данных Prometheus
6. Либо удалите дашборд и импортируйте заново, указав источник данных при импорте

## 6. Алерты не появляются в Alertmanager

**Симптом:** В Prometheus алерты есть (`http://serv123.ru:9090/alerts`), а в Alertmanager пусто (`http://serv123.ru:9093/#/alerts`).

**Вероятная причина:** Prometheus не отправляет алерты в Alertmanager из-за неправильной конфигурации.

**Проверка:**

```bash
curl -s http://localhost:9090/api/v1/alertmanagers
sudo journalctl -u prometheus -n 30 --no-pager | grep -i alertmanager
```

**Решение:**

Откройте конфиг Prometheus:

```bash
sudo vim /etc/prometheus/prometheus.yml
```

Убедитесь, что есть секция alerting:

```yaml
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['127.0.0.1:9093']
```

Перезапустите Prometheus:

```bash
sudo systemctl restart prometheus
```

Подождите 30 секунд и проверьте Alertmanager.

## 7. Порт 9093 недоступен из браузера

**Симптом:** Страница `http://serv123.ru:9093` не открывается, браузер выдаёт ошибку «Сайт недоступен» или «Connection refused».

**Вероятная причина:** Alertmanager не запущен, либо порт 9093 закрыт в UFW.

**Проверка:**

```bash
sudo systemctl status alertmanager
sudo netstat -tlnp | grep 9093
sudo ufw status | grep 9093
```

**Решение:**

Если Alertmanager не запущен:

```bash
sudo systemctl start alertmanager
sudo systemctl enable alertmanager
```

Если порт 9093 закрыт в UFW:

```bash
sudo ufw allow 9093/tcp
sudo ufw reload
```

Перезапустите Alertmanager:

```bash
sudo systemctl restart alertmanager
```

## 8. Node Exporter не запускается после перезагрузки

**Симптом:** После перезагрузки сервера метрики Node Exporter недоступны, страница `http://serv123.ru:9100/metrics` не открывается.

**Вероятная причина:** Сервис Node Exporter не добавлен в автозагрузку.

**Проверка:**

```bash
sudo systemctl status node_exporter
sudo systemctl is-enabled node_exporter
```

**Решение:**

Добавьте Node Exporter в автозагрузку:

```bash
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter
```

Проверьте, что порт 9100 открыт:

```bash
sudo ufw allow 9100/tcp
sudo ufw reload
```

## 9. Ошибка "require_tls" в Alertmanager

**Симптом:** В логах Alertmanager ошибка: `'require_tls' is true (default) but "smtp.mail.ru:465" does not advertise the STARTTLS extension`.

**Вероятная причина:** Порт 465 не поддерживает STARTTLS, а Alertmanager требует включённого TLS.

**Проверка:**

```bash
sudo journalctl -u alertmanager -n 30 --no-pager | grep -i tls
```

**Решение:**

Откройте конфиг Alertmanager:

```bash
sudo vim /etc/alertmanager/alertmanager.yml
```

Замените порт 465 на 587 и оставьте require_tls: true:

```yaml
global:
  smtp_smarthost: 'smtp.mail.ru:587'
  smtp_require_tls: true
```

Перезапустите Alertmanager:

```bash
sudo systemctl restart alertmanager
```

## 10. Тестовое письмо не приходит

**Симптом:** TestAlert активен в Alertmanager, но письмо на почту не приходит.

**Вероятная причина:** Неправильный пароль приложения или проблемы с SMTP-сервером.

**Проверка:**

```bash
sudo journalctl -u alertmanager -n 50 --no-pager | grep -i "email\|smtp\|mail"
```

**Решение:**

1. Проверьте, что в конфиге указан правильный пароль приложения, а не обычный пароль от почты
2. Откройте конфиг Alertmanager:

```bash
sudo vim /etc/alertmanager/alertmanager.yml
```
3. Убедитесь, что настройки SMTP верны:

```yaml
global:
  smtp_smarthost: 'smtp.mail.ru:587'
  smtp_from: 'ваша_почта@mail.ru'
  smtp_auth_username: 'ваша_почта@mail.ru'
  smtp_auth_password: 'пароль_приложения'
  smtp_require_tls: true
```
4. Перезапустите Alertmanager:

```bash
sudo systemctl restart alertmanager
```
5. Подождите 30 секунд и проверьте почту

