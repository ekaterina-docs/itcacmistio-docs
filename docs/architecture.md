# Принцип работы

Программный компонент **acmistio** (оператор ЦУГИ ЕСАУС) обеспечивает интеграцию service mesh Istio с внешним удостоверяющим центром (УЦ) и заменяет встроенный механизм Citadel Istio на внешний центр сертификации, интегрированный через платформу ЦУГИ.

## Ключевые этапы работы

1. Оператор разворачивается в отдельном namespace (`itcacm-citadel-system`).
2. Оператор регистрирует корневой сертификат (`acm.caroot`) и устанавливает защищённое соединение с мостами ЦУГИ через указанные `configServers`.
3. При старте или перезапуске `istiod` оператор автоматически формирует запрос на сертификат (CSR) контрольной плоскости Istio.
4. CSR передаётся по цепочке:  
   Оператор → Распределённые мосты ЦУГИ → Драйвер УЦ → Удостоверяющий центр (с HSM).
5. УЦ подписывает сертификат с использованием HSM и возвращает его по обратной цепочке. Оператор сохраняет подписанный сертификат в секрет `istiod-tls` (namespace `istio-system`).
6. Применяется конфигурация Istio, отключающая встроенный Citadel и подключающая внешний CA (файл `istio-config-1.12.2.yaml`).
7. После переключения все workload-поды service mesh (через Envoy sidecar и webhook-инжектор Istio) автоматически запрашивают и получают mTLS-сертификаты от внешнего УЦ по той же цепочке.

## Схема взаимодействия

```mermaid
sequenceDiagram
    autonumber
    participant Operator as Оператор ЦУГИ ЕСАУС<br/>(acmistio)
    participant Bridges as Распределённые<br/>мосты ЦУГИ
    participant Driver as Драйвер УЦ
    participant CA as Удостоверяющий<br/>центр (с HSM)
    participant Istiod as istiod<br/>(Istio Pilot)
    participant Pods as Workload-поды<br/>service mesh

    Note over Operator: Инициализация:<br/>корневой сертификат,<br/>configServers,<br/>подключение к управляющим центрам

    Istiod->>Operator: Запрос сертификата<br/>контрольной плоскости
    Operator->>Bridges: Передача CSR
    Bridges->>Driver: Передача CSR
    Driver->>CA: Передача CSR
    CA->>CA: Подписание с HSM
    CA-->>Driver: Подписанный сертификат
    Driver-->>Bridges: Возврат
    Bridges-->>Operator: Возврат
    Operator->>Operator: Сохранение в секрет istiod-tls
    Operator-->>Istiod: Монтирование сертификата

    Note over Istiod: Встроенный Citadel отключён<br/>Работа с внешним УЦ

    loop Для каждого workload-пода
        Pods->>Istiod: Запрос mTLS-сертификата<br/>(через Envoy sidecar)
        Istiod->>Operator: Передача workload CSR
        Operator->>Bridges: Передача CSR
        Bridges->>Driver: ...
        Driver->>CA: ...
        CA->>CA: Подписание с HSM
        Note right of CA: Возврат по обратной цепочке
        Operator-->>Istiod: Подписанный сертификат
        Istiod-->>Pods: Выдача mTLS-сертификата
    end
```
