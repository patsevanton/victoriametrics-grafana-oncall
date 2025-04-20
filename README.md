# Эффективное оповещение с OnCall в связке с VictoriaMetrics и Grafana

В современных системах мониторинга своевременное оповещение — один из ключевых аспектов стабильной работы сервисов. 
Интеграция инструментов мониторинга и алертинга позволяет быстро реагировать на инциденты. 
Рассмотрим, как **OnCall** можно эффективно использовать вместе с **VictoriaMetrics** и **Grafana** для построения мощной системы оповещений.

### Что такое OnCall?

**OnCall** — это open-source инструмент, предназначенный для управления дежурствами (on-call rotations), 
маршрутизации алертов и интеграции с различными источниками оповещений, включая Prometheus-совместимые системы. 
Его основная задача — гарантировать, что алерты доходят до нужных людей в нужное время.

Ключевые возможности:
- Гибкое расписание дежурств.
- Эскалации и повторные уведомления.
- Интеграции с мессенджерами (Slack, Telegram, email и др.).
- UI для управления алертами и расписаниями.

### Интеграция с VictoriaMetrics

**VictoriaMetrics** — это высокопроизводительная, масштабируемая база данных для хранения метрик, совместимая с Prometheus API.
Благодаря этой совместимости, она отлично подходит для использования в связке с OnCall.

Типовой сценарий:
1. Метрики поступают в VictoriaMetrics.
2. В Grafana настраиваются дашборды.
3. Алерты настраиваются кодом.
4. Alertmanager отправляет алерты в OnCall через Webhook-интеграцию.
5. OnCall маршрутизирует алерты в соответствии с дежурствами и политиками оповещений.

### Alertmanager как источник алертов

**Alertmanager** — это компонент в экосистеме Prometheus, который отвечает за обработку и маршрутизацию алертов. 
Он позволяет фильтровать, группировать и отправлять алерты в различные системы оповещений, включая **OnCall**.
Alertmanager может быть интегрирован с **OnCall** через **webhook**, что позволяет передавать алерты в OnCall для дальнейшей обработки и маршрутизации. 
Для этого необходимо настроить URL-эндпоинт OnCall в конфигурации Alertmanager и,
при необходимости, настроить шаблоны сообщений для удобства уведомлений.

### Почему стоит использовать OnCall?

Без OnCall оповещения часто теряются в общей массе уведомлений — алерты приходят всем или никому. OnCall же предлагает:
- Централизацию управления инцидентами.
- Отслеживание, кто отвечает за инцидент.
- Историю алертов и реакций.
- Автоматическую эскалацию при игнорировании алерта.

### Пример рабочего процесса

1. Пользователь настраивает алерт в Grafana (например, на недоступность сервиса).
2. Alertmanager срабатывает и отправляет webhook в OnCall.
3. OnCall определяет текущего дежурного.
4. Отправляется уведомление в Telegram или звонит.
5. Если дежурный не реагирует — алерт эскалируется следующему по очереди.

---

## Установка необходимых компонентов

Для настройки системы оповещений с OnCall, VictoriaMetrics и Grafana через Kubernetes, 
используем **Kind** для создания кластера Kubernetes и Helm-чарты для установки необходимых компонентов.

### Шаг 1: Создание Kubernetes кластера через Kind

Создайте кластер Kubernetes с помощью Kind:

```bash
kind create cluster --name victoria-metrics-oncall
```

### Шаг 2: Установка VictoriaMetrics

Для установки **VictoriaMetrics** в Kubernetes, создайте файл настроек `victoriametrics-values.yaml`:

**Файл: `victoriametrics-values.yaml`**

```yaml
# Отключаем установку Grafana
grafana:
  enabled: false
```

Теперь установите VictoriaMetrics с использованием Helm:

```bash
helm repo add victoria-metrics https://victoriametrics.github.io/helm-charts/
helm repo update
helm install victoria-metrics victoria-metrics/victoria-metrics-k8s-stack \
  --namespace monitoring \
  --create-namespace \
  -f victoriametrics-values.yaml
```

### Шаг 3: Установка Grafana с плагином OnCall

Создайте файл настроек для **Grafana** с плагином **OnCall**:

**Файл: `grafana-values.yaml`**

```yaml
adminUser: admin
adminPassword: admin
plugins:
  - grafana-oncall-app
grafana.ini:
  plugins:
    allow_loading_unsigned_plugins: grafana-oncall-app
  oncall:
    enabled: true
```

Установите **Grafana** с плагином OnCall:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install grafana grafana/grafana \
  --namespace monitoring \
  --create-namespace \
  -f grafana-values.yaml
```

### Шаг 4: Установка OnCall как отдельного сервиса

Если вы хотите установить **OnCall** как отдельный сервис (не через плагин в Grafana), создайте файл настроек `oncall-values.yaml`:

**Файл: `oncall-values.yaml`**

```yaml
grafana:
  enabled: false
django:
  secretKey: "YOUR_SECRET_KEY_HERE"
postgresql:
  auth:
    postgresPassword: "oncallpass"
```

> ⚠️ Генерируйте `secretKey` заранее: `openssl rand -hex 32`

Теперь установите **OnCall** с помощью Helm:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install grafana-oncall grafana/oncall \
  --namespace oncall \
  --create-namespace \
  -f oncall-values.yaml
```


### Заключение

Связка **OnCall + VictoriaMetrics + Grafana** — это мощный инструмент для обеспечения бесперебойной работы инфраструктуры. 
OnCall закрывает критически важный аспект — управление ответственностью за инциденты и коммуникацию,
делая процесс реагирования прозрачным, предсказуемым и управляемым.

Теперь, настроив эти компоненты, вы можете эффективно использовать их для мониторинга, 
управления инцидентами и автоматического оповещения в случае возникновения проблем с вашими сервисами.
