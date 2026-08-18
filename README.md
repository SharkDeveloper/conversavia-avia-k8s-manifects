# conversavia-avia-k8s-manifects

Репозиторий Kubernetes-манифестов для GitOps-развёртывания внутренней платформы **InvenTree** (веб-приложение управления запасами) в локальном Kubernetes-кластере.

Управление доставкой выполняется оператором **ArgoCD** (GitOps). Репозиторий содержит готовые манифесты для каждого компонента платформы; синхронизация с кластером и управление происходят через **GUI ArgoCD**.

## Назначение

Платформа состоит из трёх компонентов, развёрнутых в общем неймспейсе `inventree`:

| Компонент | Роль | Образ | Способ управления |
|---|---|---|---|
| **InvenTree** | Веб-приложение | `inventree/inventree` | Helm-чарт |
| **PostgreSQL** | База данных | `postgres:15-alpine` | Kustomize |
| **Redis** | Кеш | `redis:7-alpine` | Kustomize |

## Структура репозитория

```
conversavia-avia-k8s-manifects/
├── README.md
└── apps/
    ├── inventree/                     # InvenTree (Helm-чарт)
    │   ├── values-inventree.yaml      # реальные переопределения под кластер
    │   ├── inventree-default-values.yaml  # дефолтные values Helm-чарта
    │   ├── ingress-deploy.yaml        # Ingress (nginx, любой Host)
    │   └── certs/                     # TLS-сертификаты (⚠️ временно в git)
    ├── postgres/
    │   ├── kustomization.yaml
    │   ├── postgres.yaml              # Secret + PVC + Deployment + Service
    │   └── postgres-pv-25gi.yaml      # локальный PersistentVolume
    └── redis/
        ├── kustomization.yaml
        └── redis.yaml                 # Secret + PVC + Deployment + Service
```

> **Принцип:** один компонент = одна папка в `apps/` = один Application в ArgoCD. Каждый компонент синхронизируется независимо.

## Требования

- Kubernetes-кластер (подходит локальный: k3s / kind / minikube с `local-path` или `local-storage` StorageClass)
- ArgoCD, установленный и настроенный в кластере
- Ingress-контроллер `nginx` (используется в `values-inventree.yaml` и `ingress-deploy.yaml`)

## Инструкция по запуску

### Шаг 1. Подключите репозиторий в ArgoCD

1. Откройте **GUI ArgoCD** (по умолчанию `https://<argo-host>:8080`).
2. **Settings → Repositories → + Connect Repo**.
3. Укажите:
   - `Connection method`: HTTPS / SSH
   - `Repository URL`: URL этого репозитория
   - при необходимости — `Username` / `Password` или SSH-ключ
4. **Test Connection** → сохраните.

### Шаг 2. Создайте Application через GUI

Для **PostgreSQL** и **Redis** (тип **Directory/Kustomize**):

| Поле | Значение |
|---|---|
| Application name | `postgres` / `redis` |
| Project | `default` |
| Sync Policy | `Automatic` |
| Repository URL | этот репозиторий |
| Revision | `HEAD` |
| Path | `apps/postgres` / `apps/redis` |
| Cluster URL | локальный кластер |
| Namespace | `inventree` |

Включите опции **Prune** и **Self Heal** (вкладка *Sync Options*).

Для **InvenTree** (тип **Helm**):

| Поле | Значение |
|---|---|
| Application name | `inventree` |
| Project | `default` |
| Sync Policy | `Automatic` |
| Repository URL | этот репозиторий |
| Revision | `HEAD` |
| Path | `apps/inventree` |
| Release Name | `inventree` |
| Chart | `inventree/inventree` |
| Values | `values-inventree.yaml` |
| Namespace | `inventree` |

После создания ArgoCD применит манифесты и начнёт синхронизировать кластер с репозиторием.

### Шаг 3. Проверка

В GUI ArgoCD должны появиться 3 Application (`inventree`, `postgres`, `redis`) со статусом **Synced / Healthy**. В кластере должен существовать неймспейс `inventree` с развёрнутыми подами.

## Обновление образов из CI (CI → CD)

Автоматическое обновление образов реализуется компонентом **ArgoCD Image Updater** (управляется аннотациями на Application, без скриптов в CI).

Добавьте на Application `inventree` следующие аннотации:

```yaml
argocd-image-updater.argoproj.io/image-list: inventree=inventree/inventree
argocd-image-updater.argoproj.io/inventree.update-strategy: latest
argocd-image-updater.argoproj.io/inventree.write-back-method: git
```

- CI собирает новый образ и пушит его в registry;
- Image Updater замечает новый тег и обновляет параметр образа в Application;
- при `write-back-method: git` изменение коммитится обратно в этот репозиторий;
- ArgoCD применяет новую версию в кластере.

Для PostgreSQL/Redis обновление образов обычно не требуется (фиксированные версии), но механизм тот же — достаточно добавить аннотации на соответствующее Application.

## Добавление нового сервиса

1. Создайте папку `apps/<service>/` с манифестами.
2. Добавьте `kustomization.yaml` (для YAML-манифестов) или values (для Helm-чарта).
3. Укажите `namespace: inventree` в манифестах.
4. Создайте новый Application в GUI ArgoCD (см. Шаг 2 выше), указав `Path: apps/<service>`.
5. При необходимости добавьте аннотации Image Updater для автообновления образа.

## Примечания и известные ограничения

- **Секреты в git:** пароли в `postgres.yaml`, `redis.yaml`, `values-inventree.yaml` и TLS-ключ в `certs/tls.key` сейчас хранятся в открытом виде. Для продакшена рекомендуется вынести их в Vault / SealedSecrets / SOPS.
- **Одно окружение:** платформа разворачивается в одном неймспейсе `inventree`. Добавление окружений (dev/test/prod) выполняется добавлением новых папок и Application без перестройки существующей структуры.
- **Хардкод хостов:** `rp.store` и `192.168.5.87` заданы в `values-inventree.yaml` — при смене окружения их нужно переопределять.
