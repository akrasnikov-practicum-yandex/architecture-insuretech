# architecture-insuretech

Проектная работа спринта 8 «Масштабируемые и отказоустойчивые системы» (Яндекс Практикум, курс
«Software Architect»). В этом спринте я выступаю архитектором ПО для **InsureTech** — агрегатора
страхования с B2C-сайтом и B2B-API. Основной продукт компании — страхование жизни, плюс запускается
новый продукт: ОСАГО онлайн.

Каждое задание лежит в своей директории `TaskN`: схемы draw.io, манифесты и README с объяснением
принятых решений.

## Задания

| # | Задание | Артефакты | Кратко о решении |
|---|---------|-----------|------------------|
| 1 | [Технологическая архитектура (to-be)](Task1/) | [.drawio](Task1/InsureTech_технологическая_архитектура_to-be.drawio) · [README](Task1/README.md) | Active-Active в трёх зонах с независимыми кластерами K8s, GSLB + L7 + API Gateway, CDN для статики. БД — PostgreSQL под управлением Patroni с репликами и бэкапами в S3. Шардирование не понадобилось. Решение закрывает SLA 99,9%, RTO 45 мин и RPO 15 мин |
| 2 | [Динамическое масштабирование (HPA)](Task2/) | [deployment](Task2/deployment.yaml) · [service](Task2/service.yaml) · [hpa](Task2/hpa.yaml) · [locustfile](Task2/locustfile.py) · [README](Task2/README.md) | HPA `autoscaling/v2` по памяти и CPU (цель 80%, от 1 до 10 реплик) в Minikube. Под нагрузкой Locust реплики выросли с 1 до 10 — лог прогона приложен |
| 3 | [Переход на Event-Driven](Task3/) | [problems-and-risks](Task3/problems-and-risks.md) · [.drawio](Task3/InsureTech_C4_container_to-be.drawio) · [README](Task3/README.md) | Синхронные REST-опросы заменены на события через Apache Kafka. У продюсеров — Transactional Outbox, у потребителей — идемпотентность под at-least-once |
| 4 | [Продажа ОСАГО](Task4/) | [.drawio](Task4/InsureTech_C4_container_osago.drawio) · [README](Task4/README.md) | Новый сервис `osago-aggregator` с хранилищем в Redis. Предложения страховых стримятся пользователю через SSE по мере поступления, интеграции защищены паттернами Timeout, Retry, Circuit Breaker, Rate Limiting и Bulkhead |
| 5 | [GraphQL API для client-info](Task5/) | [client-info.graphql](Task5/client-info.graphql) · [README](Task5/README.md) | REST API переведён на GraphQL: типы Client, Document и Relative с вложенными полями. Запросы покрывают все REST-операции, потребитель выбирает только нужные поля |
| 6 | [—](Task6/) | [README](Task6/README.md) | Задания под этим номером в спецификации спринта нет — директория-заглушка |

## Технологический контекст

Kubernetes, PostgreSQL + Patroni, Apache Kafka, Redis, GSLB / Yandex Cloud DNS, CDN, Prometheus +
Grafana; паттерны: Active-Active, Transactional Outbox, CQRS, Rate Limiting, Circuit Breaker, Retry,
Timeout, Bulkhead, SSE.

## Документация

- Условие проектной работы: [docs/submit.md](docs/submit.md)
- Решения и обоснования — в `README.md` каждой директории `TaskN`.
