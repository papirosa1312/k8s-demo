# Звіт до Практичної роботи 10: Kubernetes (практична частина)

## Рівень 60 балів

### 1. Підготовка репозиторію та локальний запуск
Репозиторій `vivalafit/k8s-demo` форкнуто, клоновано, встановлено залежності (`npm install`) та запущено локально (`npm run dev`).

Результати перевірки ендпоінтів:
* `GET /api/status` — повертає JSON зі статусом сервісу (`"ok"`), поточним режимом `appMode`, версією застосунку та поточною міткою часу (`timestamp`).
* `GET /api/items?filter=a` — повертає відфільтрований JSON-масив елементів, у назвах яких є підрядок `a`.
* `GET /api/slow` — імітує повільну обробку запиту (відповідає зі штучною затримкою).
* `GET /api/error` — навмисно повертає серверну помилку (`500 Internal Server Error`) для перевірки обробки збоїв і логування.
* `GET /healthz` — liveness-ендпоінт, повертає `200 OK`, підтверджуючи, що процес живий.
* `GET /ready` — readiness-ендпоінт, повертає `200 OK`, підтверджуючи готовність сервісу приймати трафік.

### 2. Контейнеризація застосунку
* **Команда збірки образу:**
  ```bash
  docker build -t k8s-demo-app:1.0.0 .
  ```
* **Команда запуску контейнера:**
  ```bash
  docker run --rm -p 8080:8080 -e SECRET_TOKEN=local k8s-demo-app:1.0.0
  ```
* **Приклад відповіді `GET /api/status` із контейнера:**
  ```json
  <ВСТАВИТИ_JSON_З_DOCKER>
  ```

### 3. Запуск Minikube та перевірка кластера
Кластер запущено командою `minikube start --memory=4096 --cpus=2 --driver=docker`.
* **Активний контекст kubectl (`kubectl config current-context`):** `minikube`
* **Статус ноди (`kubectl get nodes`):**
  ```text
  NAME       STATUS   ROLES           AGE   VERSION
  minikube   Ready    control-plane   57s   v1.37.0
  ```

### 4. Розгортання маніфестів (Namespace, ConfigMap, Secret, Deployment, Service)
Застосовано маніфести з каталогу `k8s/` та зібрано образ усередині Minikube (`minikube image build -t maxnavrotsky/k8s-demo-app:1.0.0 .`). Стан ресурсів (`kubectl get deployment,rs,pods,svc -n k8s-demo`):
```text
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/k8s-demo-api   2/2     2            2           3m30s

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/k8s-demo-api-dc45fcc9d    2         2         2       33s

NAME                                READY   STATUS    RESTARTS   AGE
pod/k8s-demo-api-dc45fcc9d-qfcdf    1/1     Running   0          27s
pod/k8s-demo-api-dc45fcc9d-t87r7    1/1     Running   0          33s

NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/k8s-demo-api   ClusterIP   10.96.38.138   <none>        80/TCP    3m26s
```
Усі pod-и мають статус `Running` та `READY 1/1`.

### 5. Доступ до сервісу через Service та `kubectl port-forward`
Виконано `kubectl port-forward svc/k8s-demo-api -n k8s-demo 8080:80`. При зверненні до `http://localhost:8080/api/status` та `/api/items` запит проходить через тунель `kubectl` на **Service (`k8s-demo-api`, ClusterIP)**, який балансує його на один із активних **Pod-ів** Deployment-а (`Service → Pod`).

Приклад відповідей:
```json
{"status":"ok","appMode":"prod","version":"1.0.0","timestamp":"2026-11-25T22:22:56.373Z"}
[{"id":1,"name":"Item A"},{"id":2,"name":"Item B"},{"id":3,"name":"Item C"}]
```

---

## Рівень 75 балів

### 1–2. Робота з ConfigMap та зміна `APP_MODE` без рестарту pod-ів
У `k8s/configmap.yaml` значення `APP_MODE` змінено на `"prod"` і застосовано командою `kubectl apply -f k8s/configmap.yaml`.

**Чому значення всередині запущеного pod-а не змінюється без рестарту:** змінні оточення (`env`) передаються в контейнер лише **один раз — на етапі запуску (створення) контейнера**. Оновлення об'єкта `ConfigMap` у кластері не модифікує середовище вже запущених процесів усередині існуючих pod-ів.

### 3. Перезапуск Deployment (`rollout restart`) та оновлена конфігурація
Після виконання `kubectl rollout restart deployment k8s-demo-api -n k8s-demo` Kubernetes перестворив pod-и, і нові екземпляри підхопили актуальну конфігурацію:
```text
> kubectl get pods -n k8s-demo
NAME                            READY   STATUS    RESTARTS   AGE
k8s-demo-api-6555b8bc9d-7htmc   1/1     Running   0          29s
k8s-demo-api-6555b8bc9d-xjpnj   1/1     Running   0          35s

> kubectl exec -n k8s-demo deploy/k8s-demo-api -- env | findstr APP_MODE
APP_MODE=prod

> curl http://localhost:8080/api/status
{"status":"ok","appMode":"prod","version":"1.0.0","timestamp":"2026-11-25T22:24:24.263Z"}
```
**Роль `kubectl rollout restart`:** команда ініціює плавне перестворення (rolling update) pod-ів Deployment-а без простою сервісу — нові pod-и стартують із оновленими `env`-змінними, а старі поступово завершуються.

**Рекомендований підхід до оновлення конфігурації через ConfigMap:** використовувати версіонування ConfigMap або додавати хеш вмісту ConfigMap в анотацію шаблону pod-а (`PodTemplate`) у Deployment, щоб Kubernetes автоматично запускав rolling update при кожній зміні конфігурації.

### 4. Аналіз health-checkів (`liveness` та `readiness`) та логи pod-а
Параметри з `kubectl describe pods -n k8s-demo`:
```text
Liveness:   http-get http://:8080/healthz delay=5s timeout=1s period=10s #success=1 #failure=3
Readiness:  http-get http://:8080/ready delay=3s timeout=1s period=5s #success=1 #failure=3
```
* **Liveness probe:** шлях `/healthz` (`initialDelaySeconds: 5`, `periodSeconds: 10`, `failureThreshold: 3`).
* **Readiness probe:** шлях `/ready` (`initialDelaySeconds: 3`, `periodSeconds: 5`, `failureThreshold: 3`).

Приклад логів (`kubectl logs -n k8s-demo deploy/k8s-demo-api --tail=15`):
```text
Found 2 pods, using pod/k8s-demo-api-6555b8bc9d-xjpnj
[2026-11-25T22:25:07.370Z] GET /ready 200 2ms
[2026-11-25T22:25:11.381Z] GET /healthz 200 1ms
[2026-11-25T22:25:12.371Z] GET /ready 200 2ms
[2026-11-25T22:25:17.369Z] GET /ready 200 1ms
[2026-11-25T22:25:21.381Z] GET /healthz 200 1ms
[2026-11-25T22:25:22.370Z] GET /ready 200 1ms
[2026-11-25T22:25:27.370Z] GET /ready 200 2ms
[2026-11-25T22:25:31.379Z] GET /healthz 200 1ms
[2026-11-25T22:25:32.370Z] GET /ready 200 1ms
[2026-11-25T22:25:37.370Z] GET /ready 200 1ms
[2026-11-25T22:25:41.379Z] GET /healthz 200 0ms
[2026-11-25T22:25:42.371Z] GET /ready 200 1ms
[2026-11-25T22:25:47.369Z] GET /ready 200 1ms
[2026-11-25T22:25:51.379Z] GET /healthz 200 1ms
[2026-11-25T22:25:52.370Z] GET /ready 200 2ms
```

---

## Відповіді на контрольні запитання

1. **У чому відмінність між ConfigMap та Secret?**
   `ConfigMap` призначений для зберігання неконфіденційних налаштувань у відкритому вигляді (режими роботи, URL, таймаути). `Secret` використовується для чутливих даних (паролі, токени, ключі API) — значення кодуються в Base64, можуть шифруватися в etcd і мають жорсткіший контроль доступу (RBAC).

2. **У чому відмінність між liveness- та readiness-перевірками?**
   * **Liveness probe** перевіряє, чи живий сам процес усередині контейнера; якщо перевірка провалюється — `kubelet` **перезапускає контейнер**.
   * **Readiness probe** перевіряє, чи готовий pod обробляти вхідні запити; якщо вона провалюється — pod **не перезапускається**, а тимчасово виключається з балансування трафіку в Service.

3. **Через які ресурси проходить запит?**
   * **Через `kubectl port-forward`:** `Client (localhost:8080) → kubectl tunnel → Service (k8s-demo-api) → Pod`.
   * **Через `http://k8s-demo.local` (Ingress):** `Client → DNS (/etc/hosts) → Ingress Controller (k8s-demo-ingress) → Service (k8s-demo-api) → Pod`.

4. **Як рекомендується оновлювати конфігурацію, якщо вона надходить через env-змінні з ConfigMap?**
   Оскільки `env`-змінні зчитуються лише при старті контейнера, після оновлення `ConfigMap` потрібно запускати rolling update pod-ів — вручну через `kubectl rollout restart deployment` або автоматично через додавання хешу `ConfigMap` в анотації `PodTemplate` (чи створення нового версіонованого ConfigMap).