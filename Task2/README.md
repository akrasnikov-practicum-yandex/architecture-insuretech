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
| `hpa.yaml` | HorizontalPodAutoscaler (`autoscaling/v2`, min 1 / max 10, память 80% + CPU 80%) |
| `locustfile.py` | Сценарий нагрузки (`GET /`) |
| *(добавить)* `scaling-log.txt` / скриншоты | Подтверждение изменения числа реплик под нагрузкой |

## Ключевые решения

- **Метрика масштабирования — память 80%** (как требует задание) **+ CPU 80%** вторым метриком в
  том же HPA. Нагрузка `GET /` может не увеличивать память приложения; HPA выбирает максимум из
  рекомендаций по обоим метрикам, поэтому масштабирование гарантированно сработает на демонстрации.
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
> запустить Locust как pod внутри кластера против ClusterIP сервиса.

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
- У `GET /` память практически **не зависит от нагрузки** (~18–20Mi постоянно) ⇒ масштабировать по
  памяти этого приложения нагрузко-зависимо нельзя. Метрика памяти 80% оставлена по требованию задания,
  но **эффективный триггер — CPU**.
- `requests.cpu` снижен `100m → 30m`, `requests.memory` `25Mi → 20Mi`: idle CPU ≈ 19m = ~63%
  (ниже 80%, без лишнего масштабирования на простое), а под нагрузкой CPU переходит 24m (80%) ⇒
  HPA масштабирует поды `1 → N` уже при меньшей нагрузке. `limits.cpu` `250m → 200m`,
  `limits.memory` оставлен `30Mi` (по заданию).
- Нагрузка снижена до 500–1000 пользователей, чтобы трафик **успешно доходил** до пода (см. шаг 4).
- Подтверждённый прогон (нагрузка внутри кластера) — в [`scaling-log.txt`](scaling-log.txt):
  CPU 329% ⇒ реплики выросли `1 → 3 → 6 → 10`.

## Примечания
- Для production-сценария HPA дополняется Cluster Autoscaler (добавляет узлы, когда подам не хватает
  ресурсов на текущих нодах).
