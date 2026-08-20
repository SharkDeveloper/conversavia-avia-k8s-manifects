# Техническая справка: откуда что брать, образы, репозитории, нешаблонные решения

> **Для кого:** для ИИ-агентов и разработчиков, которым нужны **конкретные** детали: какие репозитории подключены, какие образы используются, где лежат переопределения и какие здесь есть нешаблонные решения.
> Общее описание проекта — в **`AGENTS-PROJECT.md`**.

---

## 1. Репозитории и источники

В этой системе задействовано **два независимых репозитория** и один внешний Helm-чарт. Их легко перепутать — читайте внимательно.

### 1.1. Репозиторий манифестов (этот репозиторий)

- **URL:** `https://github.com/SharkDeveloper/conversavia-avia-k8s-manifects.git`
- **Что содержит:** все Kubernetes-манифесты, ApplicationSet, values. Это **«репозиторий развёртывания»** (deployment/GitOps).
- **Роль в ArgoCD:** из него берутся `path` для Application (пути `apps/*`).

### 1.2. Репозиторий исходников InvenTree (форк)

- **URL:** `https://github.com/SharkDeveloper/rp.store.git`
- **Что содержит:** форк исходников **InvenTree** (Python/Django приложение). Здесь лежит `contrib/container/Dockerfile`, из которого собирается Docker-образ приложения.
- **Роль:** из него собирается **образ**, который потом запускается в кластере. ⚠️ Это **не** Helm-репозиторий и **не** источник чарта.

### 1.3. Helm-чарт InvenTree (внешний)

- **Источник:** официальный чарт `inventree/inventree` из Helm-репозитория Inventree.
- **Роль:** описывает Deployment/Service/Ingress InvenTree («скелет» развёртывания).
- **Важно:** чарт остаётся **официальным**, свой форк НЕ подменяет чарт (решение «Вариант A»).

> **Ключевое различие:** репозиторий `rp.store` даёт **образ**, а чарт `inventree/inventree` даёт **шаблоны развёртывания**. Это две разные вещи.

---

## 2. Образы (Docker images)

| Компонент | Образ | Откуда берётся | Где задано |
|---|---|---|---|
| **InvenTree** | `ghcr.io/sharkdeveloper/rp.store/inventree:latest` | Собирается CI из форка `rp.store`, публикуется в **GHCR** | [`apps/inventree/values-inventree.yaml`](apps/inventree/values-inventree.yaml:67) |
| **PostgreSQL** | `postgres:15-alpine` | Docker Hub (официальный) | [`apps/postgres/postgres.yaml`](apps/postgres/postgres.yaml:44) |
| **Redis** | `redis:7-alpine` | Docker Hub (официальный) | [`apps/redis/redis.yaml`](apps/redis/redis.yaml:42) |

### 2.1. Образ InvenTree — нешаблонное решение

Образ InvenTree **НЕ** берётся из официального `inventree/inventree` на Docker Hub. Вместо этого:

1. CI-пайплайн репозитория `rp.store` (`.github/workflows/docker-image.yml`) собирает образ из `./services/inventree` (Dockerfile в `contrib/container/Dockerfile`).
2. Публикует в GitHub Container Registry по имени:
   ```
   ghcr.io/sharkdeveloper/rp.store/inventree
   ```
   с тегами `latest` и `{sha7}` (первые 7 символов SHA коммита).
3. В [`values-inventree.yaml`](apps/inventree/values-inventree.yaml:67) переопределён `image.repository`, поэтому чарт тянет именно этот образ.

> ⚠️ Так как репозиторий `rp.store` публичный, образ в GHCR публичный — `imagePullSecrets` не требуется. Если сделать приватным — понадобится pull-secret.

### 2.2. Как CI обновляет образ (CI → CD)

CI-пайплайн в `rp.store` после сборки автоматически правит манифест в `deploy/overlays/dev/deployment.yaml` через `sed` и пушит обратно. ⚠️ **Нюанс:** этот авто-коммит трогает манифест в репозитории `rp.store`, а НЕ в этом репозитории манифестов. Если захотите автообновление образа в ArgoCD здесь — лучше добавить **ArgoCD Image Updater** (аннотации на Application), а не полагаться на sed в CI.

---

## 3. ApplicationSet — нешаблонные решения

### 3.1. Почему два ApplicationSet, а не один

Все генерируемые одним ApplicationSet Application получают **одинаковый** `source`. У нас два разных типа source, поэтому нужны два ApplicationSet:

| Файл (в `apps-of-apps/`) | ApplicationSet | Тип source | Генерирует |
|---|---|---|---|
| [`appset-rp.store.yaml`](apps-of-apps/appset-rp.store.yaml:1) | `rp.store` | Kustomize (путь) | `postgres`, `redis` |
| [`appset-rp.store-inventree.yaml`](apps-of-apps/appset-rp.store-inventree.yaml:1) | `rp.store-inventree` | Helm (`chart:` + `valueFiles`) | `inventree` |

**Почему не объединить:** внутри одного ApplicationSet нельзя сделать разный `source` (условная вставка `{{- if }}` ломает YAML-валидацию). Разделение — чистое и надёжное решение.

### 3.2. Содержимое [`appset-rp.store.yaml`](apps-of-apps/appset-rp.store.yaml:1)

- **Generator:** `list` с элементами `postgres` (`apps/postgres`) и `redis` (`apps/redis`).
- **repoURL:** `https://github.com/SharkDeveloper/conversavia-avia-k8s-manifects.git`
- **namespace:** `inventree`
- **syncPolicy:** `automated` + `prune` + `selfHeal` + `CreateNamespace=true` + retry/backoff.

### 3.3. Содержимое [`appset-rp.store-inventree.yaml`](apps-of-apps/appset-rp.store-inventree.yaml:1)

- **Generator:** `list` с одним элементом `inventree`.
- **source:** `path: apps/inventree`, `chart: inventree/inventree` (официальный), `helm.valueFiles: values-inventree.yaml`.
- **namespace:** `inventree`.

### 3.4. Как добавить новый сервис

Чтобы добавить новый Kustomize-сервис, достаточно:
1. Создать папку `apps/<service>/` с `kustomization.yaml`.
2. Добавить элемент в `list.elements` в [`apps-of-apps/appset-rp.store.yaml`](apps-of-apps/appset-rp.store.yaml:1).
3. ArgoCD сам создаст Application.

---

## 4. Переопределения InvenTree ([`values-inventree.yaml`](apps/inventree/values-inventree.yaml:1))

Этот файл переопределяет официальные values чарта. Важные нешаблонные поля:

| Поле | Значение | Замечание |
|---|---|---|
| `global.siteUrl` | `https://rp.store` | Основной хост |
| `global.dbSecrets` | PostgreSQL-параметры (`postgres.inventree.svc.cluster.local:5432`, db/user/pass = `inventree`) | Связь с PG |
| `global.cacheSecrets` | Redis-параметры (`redis.inventree.svc.cluster.local:6379`, pass = `redis`) | Связь с Redis |
| `env.INVENTREE_ALLOWED_HOSTS` | `*` | Разрешает любые Host |
| `env.INVENTREE_CORS_ORIGIN_WHITELIST` | `rp.store`, `192.168.5.87` | CORS |
| `env.INVENTREE_CSRF_TRUSTED_ORIGINS` | `rp.store`, `192.168.5.87` | CSRF |
| `image.repository` | `ghcr.io/sharkdeveloper/rp.store/inventree` | Свой образ (нешаблонно) |
| `ingress.main` | nginx, host `rp.store`, TLS `inventree-tls` | Основной Ingress |
| `persistence` | StorageClass `local-path`, 10Gi | Хранение данных |

> ⚠️ `env` в values задаёт переменные через **секрет-мапу**, не через `values` напрямую. При правках следите за структурой чарта.

---

## 5. PostgreSQL — нешаблонные решения

Файлы: [`apps/postgres/postgres.yaml`](apps/postgres/postgres.yaml:1), [`apps/postgres/postgres-pv-25gi.yaml`](apps/postgres/postgres-pv-25gi.yaml:1).

### 5.1. Ресурсы в `postgres.yaml`

| Kind | Имя | Детали |
|---|---|---|
| Secret | `postgres-secret` | `POSTGRES_USER`/`POSTGRES_PASSWORD`/`POSTGRES_DB` = `inventree` |
| PVC | `postgres-pvc` | 25Gi, StorageClass `local-path` |
| Deployment | `postgres` | образ `postgres:15-alpine`, Recreate, envFrom secret |
| Service | `postgres` | порт 5432 |

### 5.2. Явный PersistentVolume — нешаблонное решение

Дополнительно к PVC создан **явный локальный PV** [`postgres-pv-25gi.yaml`](apps/postgres/postgres-pv-25gi.yaml:1):

- **capacity:** 25Gi, StorageClass `local-storage`.
- **reclaimPolicy:** `Retain`.
- **local path:** `/var/lib/postgres-data-25gi`.
- **nodeAffinity:** привязка к ноде `convers-avia-k8s-node` (⚠️ захардкожено имя ноды!).

> ⚠️ Это **нешаблонное решение**: сочетание PVC (`local-path`) + явного PV (`local-storage`). Нужно проверить, какая связка реально используется в кластере — не конфликтуют ли они. Данные хранятся на конкретной ноде, что делает миграцию ноды нетривиальной.

---

## 6. Redis — нешаблонные решения

Файл: [`apps/redis/redis.yaml`](apps/redis/redis.yaml:1).

| Kind | Имя | Детали |
|---|---|---|
| Secret | `redis-secret` | `REDIS_PASSWORD` = `redis` |
| PVC | `redis-pvc` | 2Gi, StorageClass `local-path` |
| Deployment | `redis` | образ `redis:7-alpine`, **пароль через команду** |
| Service | `redis` | порт 6379 |

### 6.1. Пароль Redis — нешаблонное решение

Redis включён **с паролем**, и пароль задаётся **не через env-переменную**, а через командную строку контейнера:

```yaml
command: ["redis-server", "--requirepass", "$(REDIS_PASSWORD)"]
```

- Пароль берётся из секрета `redis-secret` (envFrom).
- `$(REDIS_PASSWORD)` — это Kubernetes-синтаксис подстановки переменной окружения внутри `command`.
- Это важно: если убрать `--requirepass`, InvenTree не сможет подключиться (в values задан `INVENTREE_CACHE_PASSWORD`).

---

## 7. Ingress — нешаблонное решение

Помимо Ingress внутри Helm-чарта (`ingress.main` в values), есть **дополнительный** Ingress [`ingress-deploy.yaml`](apps/inventree/ingress-deploy.yaml:1):

- **Имя:** `inventree-ip-access`.
- **Служит** для доступа по IP (`192.168.5.87`).
- **Критично:** в rules **нет поля `host`** — он матчит **ЛЮБОЙ** Host-заголовок (fallback). ⚠️ Это может конфликтовать с основным Ingress по `rp.store`. Нужно проверять порядок/приоритет nginx-ingress.

---

## 8. Секреты и безопасность

| Секрет | Где | Пароль/ключ |
|---|---|---|
| `postgres-secret` | `apps/postgres/postgres.yaml` | `inventree` / `inventree` |
| `redis-secret` | `apps/redis/redis.yaml` | `redis` |
| `inventree-admin-secret` (ожидается) | `values-inventree.yaml` (`global.adminSecret`) | создаётся отдельно |
| `inventree-tls` (TLS) | `apps/inventree/certs/` | самоподписанный |

⚠️ **Все секреты лежат в git в открытом виде.** Для продакшена — Vault / SealedSecrets / SOPS.

---

## 9. Два способа запуска системы

Систему можно развернуть двумя способами. Оба приводят к одному результату — появятся Application `inventree`, `postgres`, `redis` в статусе **Synced / Healthy**.

### Способ 1 — CLI: `kubectl apply` (прямой)

Самый быстрый. Применяет ApplicationSet'ы напрямую в кластер.

**Предварительно:** репозиторий манифестов должен быть подключён в ArgoCD (Settings → Repositories), а также чарт-репозиторий Inventree.

```bash
# Применяем оба ApplicationSet
kubectl apply -f apps-of-apps/appset-rp.store.yaml -n argocd
kubectl apply -f apps-of-apps/appset-rp.store-inventree.yaml -n argocd

# Проверяем, что ArgoCD создал Application
kubectl get applications -n argocd
```

**Что происходит:**
1. ArgoCD видит новый ApplicationSet `rp.store` → генерирует Application `postgres` и `redis`.
2. ArgoCD видит ApplicationSet `rp.store-inventree` → генерирует Application `inventree`.
3. Благодаря `syncPolicy.automated` + `selfHeal` + `CreateNamespace=true` всё применяется автоматически.

**Плюс:** минимум действий, работает сразу.
**Минус:** состояние ApplicationSet создаётся вручную (`kubectl`), а не из Git — при «запуске с нуля» на другом кластере нужно снова выполнять `apply`.

### Способ 2 — App of Apps (GitOps, рекомендован)

Правильный GitOps-подход: ApplicationSet'ы кладутся в Git, а их применяет **корневой Application** (`rp.store-root`), который отслеживает папку в репозитории.

**1. Убедитесь, что ApplicationSet'ы лежат в Git** в папке `apps-of-apps/` (они уже находятся там в этом репозитории):
```
apps-of-apps/
├── appset-rp.store.yaml
└── appset-rp.store-inventree.yaml
```

**2. Создайте в GUI ArgoCD один корневой Application** (`+ New App`):

| Поле | Значение |
|---|---|
| Application name | `rp.store-root` |
| Repository URL | `https://github.com/SharkDeveloper/conversavia-avia-k8s-manifects.git` |
| Path | `apps-of-apps` |
| Cluster URL | локальный кластер |
| Namespace | `argocd` |
| Sync Policy | Automatic + Prune + Self Heal |

**3. Нажмите Create → Sync.**

**Что происходит:**
1. Корневой Application `rp.store-root` синхронизирует папку `apps-of-apps/`.
2. ArgoCD применяет оба ApplicationSet из Git.
3. ApplicationSet генерируют три Application (`inventree`, `postgres`, `redis`).
4. Дальше всем стеком управляет одна кнопка **Sync** на `rp.store-root`.

**Плюс:** Git — единственный источник правды (GitOps). ApplicationSet живут в Git, их можно пересоздать на любом кластере одним `apply` корневого Application.
**Минус:** нужен один шаг подготовки (папка в Git + корневой Application).

### Сравнение способов

| Критерий | Способ 1 (CLI) | Способ 2 (App of Apps) |
|---|---|---|
| Команд | 2× `kubectl apply` | 1× Sync в GUI |
| ApplicationSet в Git | ❌ нет | ✅ да |
| GitOps / источник правды | частично | ✅ полностью |
| Пересоздание на новом кластере | повторять `apply` | применить корневой App |
| Управление из GUI | да, после apply | да, сразу |

**Рекомендация:** для одноразового поднятия — Способ 1. Для постоянного управления и мультиокруженности — Способ 2 (App of Apps).

---

## 10. Чек-лист для нового разработчика/нейронки

1. Сначала прочитай [`AGENTS-PROJECT.md`](AGENTS-PROJECT.md) — общая картина.
2. Потом этот файл — конкретные детали.
3. Проверь актуальные значения в самих манифестах (values, postgres.yaml, redis.yaml) — они источник истины.
4. При внесении изменений помни про **нешаблонные решения** (свой образ, явный PV, пароль Redis через command, доп. Ingress) — их легко сломать, не зная контекста.