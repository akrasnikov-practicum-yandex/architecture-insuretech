# architecture-insuretech

Проектная работа **спринта 8** «Масштабируемые и отказоустойчивые системы» (Яндекс Практикум,
курс «Software Architect»). Роль — архитектор ПО для **InsureTech**: агрегатор страхования с B2C-сайтом
и B2B-API, основной продукт — страхование жизни, плюс новый продукт ОСАГО.

Каждое задание — в отдельной директории `TaskN` со схемами draw.io, манифестами и пояснительными
README.

## Задания

| # | Задание | Артефакты | Кратко о решении |
|---|---------|-----------|------------------|
| 1 | [Технологическая архитектура (to-be)](Task1/) | [.drawio](Task1/InsureTech_технологическая_архитектура_to-be.drawio) · [README](Task1/README.md) | Active-Active, 3 зоны; независимые кластеры K8s; GSLB + L7 + API Gateway; PostgreSQL + Patroni (sync/async реплики, etcd, HAProxy, pgBackRest→S3); CDN; без шардирования. Под SLA 99,9% / RTO 45м / RPO 15м |
| 2 | [Динамическое масштабирование (HPA)](Task2/) | [deployment](Task2/deployment.yaml) · [service](Task2/service.yaml) · [hpa](Task2/hpa.yaml) · [locustfile](Task2/locustfile.py) · [README](Task2/README.md) | HPA `autoscaling/v2` (память 80% + CPU 80%, min 1 / max 10), Minikube + Locust |
| 3 | [Переход на Event-Driven](Task3/) | [problems-and-risks](Task3/problems-and-risks.md) · [.drawio](Task3/InsureTech_C4_container_to-be.drawio) · [README](Task3/README.md) | Apache Kafka (топики `insurance-products-updated`, `policies-issued`); Transactional Outbox у продюсеров; at-least-once + идемпотентность |
| 4 | [Продажа ОСАГО](Task4/) | [.drawio](Task4/InsureTech_C4_container_osago.drawio) · [README](Task4/README.md) | Сервис `osago-aggregator` (+ Redis), SSE для стрима предложений, Kafka `osago-offers`, паттерны Timeout/Retry/Circuit Breaker/Rate Limiting/Bulkhead, API Gateway |
| 5 | [GraphQL API для client-info](Task5/) | [client-info.graphql](Task5/client-info.graphql) · [README](Task5/README.md) | GraphQL вместо REST: типы Client/Document/Relative, вложенные поля, queries покрывают все REST-операции |
| 6 | [—](Task6/) | [README](Task6/README.md) | Задание в спецификации спринта отсутствует; директория-заглушка |

## Технологический контекст
Kubernetes, PostgreSQL + Patroni, Apache Kafka, Redis, GSLB / Yandex Cloud DNS, CDN, Prometheus +
Grafana; паттерны: Active-Active, Transactional Outbox, CQRS, Rate Limiting, Circuit Breaker, Retry,
Timeout, Bulkhead, SSE.

## Документация
- Условие проектной работы: [docs/submit.md](docs/submit.md)
- Решения и обоснования — в `README.md` каждой директории `TaskN`.

> Схемы выполнены базовыми фигурами draw.io (боксы/цилиндры/комментарии) с полной расстановкой связей и
> health-check; при необходимости иконки заменяются на библиотеки Yandex Cloud.
