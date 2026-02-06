# Установка оператора

## Предварительные требования

| Компонент | Требование | Проверка |
|-----------|------------|----------|
| Kubernetes | Доступ к кластеру, настроенный `kubectl` | `kubectl cluster-info` |
| Helm | Версия 3.0 или выше | `helm version` |
| Istio | Версия 1.12+ с запущенным `istiod` | `kubectl get pods -n istio-system` |
| Корневой сертификат | Файл `Root_CA.pem` в формате PEM | — |
| Файлы чарта | Распакованный архив с директориями `./chart` и `./config/samples/` | — |

Если образы оператора хранятся в приватном registry (например, Yandex Container Registry), подготовьте учётные данные для доступа.

## Пошаговая установка

### Шаг 1. Создание namespace

Создайте отдельное пространство имён для компонентов оператора:

```bash
kubectl create namespace itcacm-citadel-system
```

Убедитесь, что namespace создан:

```bash
kubectl get namespaces | grep itcacm-citadel-system
```

### Шаг 2. Настройка доступа к приватному registry (опционально)

> **Примечание:** Пропустите этот шаг, если образы оператора доступны публично.

Создайте секрет с учётными данными для Docker registry:

```bash
kubectl create secret docker-registry regcred \
  --docker-server=cr.yandex \
  --docker-username=<ваш-логин> \
  --docker-password=<ваш-пароль> \
  --namespace itcacm-citadel-system
```

### Шаг 3. Установка оператора через Helm

Перейдите в директорию с распакованным чартом и выполните установку:

```bash
helm upgrade --install acmistio ./chart \
  --namespace itcacm-citadel-system \
  --create-namespace \
  -f ./chart/cr.yandex.yaml \
  --set acm.caroot="$(cat ~/Root_CA.pem)" \
  --set cluster.name=cwi-prod-1 \
  --set cluster.environment=PROD \
  --set image.appVersion=2023.2.1-dev \
  --set acm.configServers="{https://p0esau-ap2301wn.domain.ru/api/acmcd,https://p0esau-ap2302lk.domain.ru/api/acmcd}"
```

#### Параметры конфигурации

| Параметр | Описание | Пример значения |
|----------|----------|-----------------|
| `acm.caroot` | Содержимое корневого сертификата CA | `"$(cat ~/Root_CA.pem)"` |
| `cluster.name` | Уникальный идентификатор кластера в системе ACM | `cwi-prod-1` |
| `cluster.environment` | Тип окружения | `DEV`, `TEST` или `PROD` |
| `image.appVersion` | Версия образа оператора | `2023.2.1-dev` |
| `acm.configServers` | Адреса серверов ACM (через запятую в фигурных скобках) | `"{https://server1/api,https://server2/api}"` |

Проверьте, что под оператора запустился:

```bash
kubectl get pods -n itcacm-citadel-system
```

Ожидаемый результат: под `acmistio-*` в статусе `Running`.

### Шаг 4. Выпуск сертификата для istiod

Создайте запрос на сертификат для центрального компонента Istio:

```bash
kubectl apply -f ./config/samples/istiod-autocert.yaml -n istio-system
```

Убедитесь, что секрет с сертификатом создан:

```bash
kubectl get secrets -n istio-system | grep istiod-tls
```

### Шаг 5. Переключение Istio на внешний CA

Примените конфигурацию для замены встроенного Citadel:

```bash
kubectl apply -f ./config/samples/istio-config-1.12.2.yaml
```

Перезапустите istiod для применения изменений:

```bash
kubectl rollout restart deployment istiod -n istio-system
```

Дождитесь готовности (1–2 минуты):

```bash
kubectl rollout status deployment istiod -n istio-system
```

## Проверка работоспособности

После завершения установки убедитесь, что все компоненты работают корректно:

```bash
# Статус пода оператора
kubectl get pods -n itcacm-citadel-system

# Наличие секрета с сертификатом
kubectl get secrets -n istio-system | grep istiod-tls

# Статус istiod
kubectl get pods -n istio-system | grep istiod
```

Все поды должны быть в статусе `Running`, секрет `istiod-tls` должен присутствовать.

## Устранение неполадок

### Просмотр логов оператора

```bash
kubectl logs -l app=acmistio -n itcacm-citadel-system
```

### Просмотр событий

```bash
kubectl get events -n itcacm-citadel-system --sort-by=.metadata.creationTimestamp
```

### Возможные проблемы

| Симптом | Возможная причина | Решение |
|---------|-------------------|---------|
| Под оператора в статусе `ImagePullBackOff` | Ошибка доступа к registry | Проверьте секрет `regcred` и параметр `imagePullSecrets` |
| Под в статусе `CrashLoopBackOff` | Некорректный сертификат или недоступны серверы ACM | Проверьте логи оператора, доступность `acm.configServers` |
| Секрет `istiod-tls` не создаётся | Ошибка связи с ACM | Проверьте сетевую доступность серверов конфигурации |
