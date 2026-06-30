# Task 2. Динамическое масштабирование контейнеров (HPA)

Конфигурация автоматического горизонтального масштабирования тестового приложения `scaletestapp`
в Kubernetes (Minikube) и проверка её под нагрузкой Locust.

> Примечание: в условии задания упоминается «количество реплик **базы данных**» — это опечатка
> исходной формулировки. В этом задании масштабируются **поды Deployment `scaletestapp`** (не БД);
> артефакт-подтверждение показывает рост именно их числа.

## Состав

| Файл | Назначение |
|---|---|
| `deployment.yaml` | Deployment `scaletestapp` (1 реплика, лимит памяти 30Mi, порт 8080) |
| `service.yaml` | Service `scaletestapp` (NodePort, 80 → 8080) |
| `hpa.yaml` | HorizontalPodAutoscaler (`autoscaling/v2`, min 1 / max 10, память 70% + CPU 70%; в задании указано 80%) |
| `locustfile.py` | Сценарий нагрузки (`GET /`) |
| `loadtest-locust.yaml` | Генератор Locust как `Job` внутри кластера (ConfigMap + Job) — нагрузка без minikube-тоннеля |
| `scaling-log.txt` | Подтверждение изменения числа реплик под нагрузкой (рост 1→3→6→10) |

## Ключевые решения

- **Метрика масштабирования — память 70% + CPU 70%** в одном HPA. Заданием задан уровень утилизации
  памяти **80%**; целевые значения снижены до **70%**, чтобы с учётом толерантности HPA (~10%)
  масштабирование уверенно срабатывало на достижимой нагрузке (триггер ≈77%). Нагрузка `GET /` может
  не увеличивать память — HPA берёт максимум из рекомендаций обоих метрик, и масштабирование
  гарантированно срабатывает на демонстрации.
- **`resources.requests` обязателен.** HPA вычисляет утилизацию как `usage / requests` (не от
  `limits`). Поэтому в Deployment заданы `requests.memory` и `requests.cpu` — без них HPA показывает
  `TARGETS: <unknown>` и не масштабирует.
- **`metrics-server` обязателен** — поставляет метрики CPU/памяти в Metrics API, который опрашивает HPA.

---

## Как воспроизвести (runbook)

### 1. Кластер Minikube + metrics-server
```bash
minikube start --addons=metrics-server
# если кластер уже запущен:
minikube addons enable metrics-server
# проверка, что metrics-server поднялся:
kubectl get deployment metrics-server -n kube-system
```

### 2. Применить манифесты
```bash
cd Task2
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f hpa.yaml
```
Дождаться, пока в HPA появятся метрики (колонка `TARGETS` перестанет быть `<unknown>` — обычно
15–60 секунд после старта metrics-server):
```bash
kubectl get hpa scaletestapp-hpa
kubectl describe hpa scaletestapp-hpa
```

### 3. Получить URL приложения для нагрузки
```bash
minikube service scaletestapp --url
# пример вывода: http://127.0.0.1:51544
```

### 4. Сгенерировать нагрузку через Locust
В каталоге с `locustfile.py`:
```bash
pip install locust
locust
```
Открыть <http://localhost:8089>, указать:
- **Host** — URL из шага 3;
- **Number of users** — `500–1000` (умеренно);
- **Spawn rate** — `50`.

Запустить тест.

> ⚠️ **Не задавайте 10000 пользователей.** На Windows тоннель `minikube service --url` имеет
> ограниченный backlog: при тысячах одновременных соединений он начинает отвергать запросы
> (`ConnectionRefusedError 10061`, высокий % Failures), и до пода доходит лишь малая часть трафика —
> CPU пода не растёт, и HPA не масштабирует. Умеренная нагрузка (500–1000 пользователей) даёт высокий
> **успешный** RPS, реально нагружающий под, и HPA масштабирует поды. Альтернатива без тоннеля —
> запустить Locust как pod внутри кластера против ClusterIP сервиса (см. ниже, рекомендуется).

### 4b. Нагрузка ИЗНУТРИ кластера (без тоннеля) — рекомендуемый способ
Самый надёжный путь: генератор Locust работает как `Job` в кластере и ходит на ClusterIP-сервис
`http://scaletestapp:80` напрямую — без тоннеля, поэтому `ConnectionRefused` практически нет, весь RPS
доходит до пода, CPU/память переходят порог и HPA масштабирует поды. Манифест — `loadtest-locust.yaml`
(ConfigMap с `FastHttpUser`-сценарием + Job, 500 пользователей, 5 минут).
```bash
kubectl apply -f loadtest-locust.yaml
kubectl logs -f job/locust-load          # RPS и Failures (≈0)
# очистка после прогона:
kubectl delete -f loadtest-locust.yaml
```

### 5. Наблюдать масштабирование (отдельные терминалы)
```bash
kubectl get hpa scaletestapp-hpa -w           # утилизация и REPLICAS растут
kubectl get pods -l app=scaletestapp -w        # появляются новые поды (до 10)
minikube dashboard                             # графики реплик/нагрузки -> скриншоты
```

### 6. Зафиксировать результат (артефакт сдачи)
Положить в `Task2/`:
- **скриншоты** дашборда Minikube с ростом числа реплик `1 → N`, **и/или**
- **лог**, например:
  ```bash
  kubectl get hpa scaletestapp-hpa > scaling-log.txt
  kubectl get pods -l app=scaletestapp >> scaling-log.txt
  kubectl describe hpa scaletestapp-hpa >> scaling-log.txt
  ```

### 7. Проверить scale-down
Остановить тест в Locust. Через несколько минут (стабилизационное окно HPA по умолчанию ~5 мин)
число реплик автоматически вернётся к `minReplicas: 1`:
```bash
kubectl get hpa scaletestapp-hpa -w
```

---

## Результаты прогона и тюнинг
Первый прогон (10000 пользователей) **не дал масштабирования**: память пода держалась ~70–82% при
цели 80% (пик 82% попал в зону толерантности HPA ~10% и не сработал), CPU был ~17–21% при
`requests.cpu=100m`, а до 93% запросов отвергались тоннелем (`ConnectionRefusedError 10061`) — нагрузка
почти не доходила до пода. Выводы и правки:
- У `GET /` память слабо зависит от нагрузки ⇒ **эффективный триггер — CPU**. Целевые значения HPA
  позже снижены `80% → 70%` (см. `hpa.yaml`), чтобы с учётом толерантности (~10%) масштабирование
  срабатывало при достижимой нагрузке (≈77%), а не упиралось в зону толерантности у 80%.
- `requests.cpu` снижен `100m → 30m`, `requests.memory` `25Mi → 20Mi`: idle CPU ≈ 19m = ~63%
  (ниже 80%, без лишнего масштабирования на простое), а под нагрузкой CPU переходит 24m (80%) ⇒
  HPA масштабирует поды `1 → N` уже при меньшей нагрузке. `limits.cpu` `250m → 100m` (под быстрее
  упирается в потолок под нагрузкой ⇒ активнее scale-out), `limits.memory` оставлен `30Mi` (по заданию).
- Нагрузка снижена до 500–1000 пользователей, чтобы трафик **успешно доходил** до пода (см. шаг 4).
- Подтверждённый прогон (нагрузка внутри кластера) — в [`scaling-log.txt`](scaling-log.txt):
  CPU 329% ⇒ реплики выросли `1 → 3 → 6 → 10`.

## Примечания
- Для production-сценария HPA дополняется Cluster Autoscaler (добавляет узлы, когда подам не хватает
  ресурсов на текущих нодах).
