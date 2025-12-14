# GitOps з Flux та Kustomize

## Структура Проєкту

Цей репозиторій організовано для підтримки GitOps розгортання з використанням Flux та Kustomize.

```
├── base/                   # Спільні маніфести (Deployment, Service, Ingress, ConfigMap)
├── overlays/
10: │   ├── development/        # Конфігурація для Dev середовища (1 репліка, Dragonfly DB instance)
11: │   └── production/         # Конфігурація для Prod середовища (3 репліки, Dragonfly Cluster, HPA)
└── clusters/
    └── my-cluster/         # Конфігурації Flux (GitRepository, Kustomizations, ImageAutomation)
```

## Середовища

### Development
- **Namespace:** `development`
- **Репліки:** 1
- **База даних:** Dragonfly (Одиночний інстанс)
- **Інсталяція:** Автоматично керується Flux (`apps-dev` Kustomization).
- **Політика оновлення:** Автоматично оновлюється при появі нових тегів `dev-*` в Docker Hub.

### Production
- **Namespace:** `production`
- **Репліки:** 3
- **База даних:** Dragonfly (Replica Set, 2 репліки)
- **Масштабування:** Увімкнено HPA (масштабування 3-10 реплік на основі CPU)
- **Інсталяція:** Автоматично керується Flux (`apps-prod` Kustomization).
- **Політика оновлення:** Автоматично оновлюється при появі нових тегів `prod-*` в Docker Hub.

## Налаштування Flux

Flux відстежує директорію `clusters/my-cluster`. Він автоматично застосовує:

1.  `app-dev.yaml` -> Синхронізує `overlays/development` у простір імен `development`.
2.  `app-prod.yaml` -> Синхронізує `overlays/production` у простір імен `production`.

### Автоматизація оновлення імеджів (Image Automation)

Налаштовано автоматичне стеження за новими версіями Docker-імеджів:

1.  **ImageRepository**: Flux регулярно сканує Docker Hub (`viktor1sss/course-app`).
2.  **ImagePolicy**:
    *   `course-app-dev`: Відбирає теги, що відповідають шаблону `dev-*` (сортування за SemVer).
    *   `course-app-prod`: Відбирає теги, що відповідають шаблону `prod-*`.
3.  **ImageUpdateAutomation**:
    *   Якщо знайдено нову версію, яка відповідає політиці, Flux **автоматично робить коміт** у цей Git-репозиторій, оновлюючи поле `newTag` у файлах `kustomization.yaml`.
    *   Після коміту Flux застосовує зміни до кластера.

> [!IMPORTANT]
> Для роботи автоматизації (запису в Git), Deploy Key, який використовує Flux, **мусить мати права на запис (Write access)**.

Перевірка статусу імеджів:
```bash
flux get image repository -A
flux get image policy -A
flux get image update -A
```

## Вимоги

- Kubernetes Cluster
- Flux встановлено та налаштовано (bootstrapped)
- Dragonfly Operator встановлено

## Перевірка

Перевірити статус синхронізації Kustomizations:

```bash
flux get kustomizations
```

Перевірити поди додатку:

```bash
kubectl get pods -n development
kubectl get pods -n production
```
