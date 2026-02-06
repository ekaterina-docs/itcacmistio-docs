# Принцип работы

Оператор реализует механизм **Kubernetes CSR API** и **Istio CA External Plugin**, позволяя `istiod` запрашивать сертификаты у внешнего центра сертификации вместо встроенного Citadel.

## Ключевые этапы работы

1. Оператор разворачивается в отдельном namespace (`itcacm-citadel-system`) и регистрирует корневой сертификат (`acm.caroot`).
2. При старте или перезапуске `istiod` оператор автоматически формирует запрос на сертификат (CSR) и отправляет его на серверы ACM (`acm.configServers`).
3. ACM подписывает сертификат корпоративным CA и возвращает его. Оператор помещает сертификат в секрет `istiod-tls` (namespace `istio-system`).
4. Конфигурация Istio переключается на внешний CA (отключение встроенного Citadel).
5. Все новые и существующие поды в service mesh получают сертификаты от внешнего CA автоматически (через sidecar-инжектор Istio).

## Схема взаимодействия

```mermaid
sequenceDiagram
    autonumber
    participant Operator as Оператор itcacmistio<br/>(itcacm-citadel-system)
    participant ACM as Серверы ACM<br/>(configServers)
    participant Secret as Секрет istiod-tls<br/>(istio-system)
    participant Istiod as istiod
    participant Pods as Поды service mesh

    Note over Operator: Инициализация с корневым<br/>сертификатом (acm.caroot)
    
    Istiod->>Operator: Запрос сертификата (CSR API)
    Operator->>ACM: Отправка CSR
    ACM->>ACM: Подписание корпоративным CA
    ACM-->>Operator: Подписанный сертификат
    Operator->>Secret: Сохранение в istiod-tls
    Secret-->>Istiod: Монтирование сертификата
    
    Note over Istiod: Работа с внешним CA<br/>(Citadel отключен)
    
    loop Для каждого пода
        Pods->>Istiod: Запрос mTLS-сертификата
        Istiod-->>Pods: Сертификат от внешнего CA
    end
```

## Преимущества

- Полная автоматизация выдачи и ротации сертификатов.
- Соответствие корпоративным требованиям безопасности.

## Ограничения

- Требуется постоянная доступность серверов ACM.
- После установки рекомендуется перезапустить workloads для применения новых сертификатов.
