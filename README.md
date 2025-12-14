# GitOps з Flux та Kustomize

## Структура Проєкту

Цей репозиторій організовано для підтримки GitOps розгортання з використанням Flux та Kustomize.

```
├── base/                   # Спільні маніфести (Deployment, Service, Ingress, ConfigMap)
├── infrastructure/         # Маніфести для інфраструктури (Dragonfly Operator)
├── overlays/
│   ├── development/        # Конфігурація для Dev середовища (1 репліка, Dragonfly DB instance)
│   └── production/         # Конфігурація для Prod середовища (3 репліки, Dragonfly Cluster, HPA)
└── clusters/
    └── my-cluster/         # Конфігурації Flux (GitRepository, Kustomizations, ImageAutomation, Infrastructure)
```

## Середовища

### Development
- **Namespace:** `development`
- **Репліки:** 1
- **База даних:** Dragonfly (Одиночний інстанс)
- **Інсталяція:** Автоматично керується Flux (`apps-dev` Kustomization).
- **Політика оновлення:** Автоматично оновлюється при появі нових тегів `dev-*` в Docker Hub.
- **Домен:** `course-app.dev.local`

### Production
- **Namespace:** `production`
- **Репліки:** 3
- **База даних:** Dragonfly (Replica Set, 2 репліки)
- **Масштабування:** Увімкнено HPA (масштабування 3-10 реплік на основі CPU)
- **Інсталяція:** Автоматично керується Flux (`apps-prod` Kustomization).
- **Політика оновлення:** Автоматично оновлюється при появі нових тегів `prod-*` в Docker Hub.
- **Домен:** `course-app.prod.local`

## Налаштування Flux

Flux відстежує директорію `clusters/my-cluster`. Він автоматично застосовує:

1.  `app-dev.yaml` -> Синхронізує `overlays/development` у простір імен `development`.
2.  `app-prod.yaml` -> Синхронізує `overlays/production` у простір імен `production`.
3.  `infrastructure.yaml` -> Синхронізує `infrastructure` у простір імен `flux-system`.
4.  `image-automation.yaml` -> Синхронізує `image-automation` у простір імен `flux-system`.


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
NAMESPACE       NAME            LAST SCAN                       SUSPENDED       READY   MESSAGE                                                
flux-system     course-app      2025-12-14T17:42:28+02:00       False           True    successful scan: found 15 tags with checksum 851326249
flux get image policy -A
NAMESPACE       NAME            IMAGE                   TAG             READY   MESSAGE                                                                                                           
flux-system     course-app-dev  viktor1sss/course-app   dev-v1.0.5      True    Latest image tag for viktor1sss/course-app resolved to dev-v1.0.5 (previously viktor1sss/course-app:dev-v1.0.4)  
flux-system     course-app-prod viktor1sss/course-app   prod-v1.0.5     True    Latest image tag for viktor1sss/course-app resolved to prod-v1.0.5 (previously viktor1sss/course-app:prod-v1.0.4)
flux get image update -A
NAMESPACE       NAME                    LAST RUN                        SUSPENDED       READY   MESSAGE               
flux-system     course-app-automation   2025-12-14T17:43:13+02:00       False           True    repository up-to-date
```

## Вимоги

- Kubernetes Cluster
- Flux встановлено та налаштовано (bootstrapped)
- Dragonfly Operator встановлено

## Перевірка

Перевірємо статус синхронізації Kustomizations:

```bash
flux get kustomizations -A
NAMESPACE       NAME            REVISION                SUSPENDED       READY   MESSAGE                              
flux-system     app-dev         main@sha1:a30a68e6      False           True    Applied revision: main@sha1:a30a68e6
flux-system     app-prod        main@sha1:a30a68e6      False           True    Applied revision: main@sha1:a30a68e6
flux-system     flux-system     main@sha1:a30a68e6      False           True    Applied revision: main@sha1:a30a68e6
flux-system     infrastructure  main@sha1:a30a68e6      False           True    Applied revision: main@sha1:a30a68e6
```

Перевіряємо поди додатку:

```bash
k get pods -n development
NAME                                     READY   STATUS    RESTARTS   AGE
course-app-deployment-7849f66fdf-xk7r5   1/1     Running   0          3h28m
dragonfly-0                              1/1     Running   0          6h22m

k get pods -n production
NAME                                     READY   STATUS    RESTARTS   AGE
course-app-deployment-7755cf8865-7l2dp   1/1     Running   0          3h29m
course-app-deployment-7755cf8865-9c7kx   1/1     Running   0          3h28m
course-app-deployment-7755cf8865-j4cqj   1/1     Running   0          3h28m
dragonfly-0                              1/1     Running   0          6h22m
dragonfly-1                              1/1     Running   0          6h22m
```
## перевіряєм видалення і автоматичне відновлення 
```bash
k get svc -n development
NAME             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
course-app-svc   ClusterIP   10.43.63.220   <none>        8080/TCP   6h22m
dragonfly        ClusterIP   10.43.21.42    <none>        6379/TCP   6h22m

k delete svc dragonfly -n development
service "dragonfly" deleted from development namespace

k get svc -n development
NAME             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
course-app-svc   ClusterIP   10.43.63.220   <none>        8080/TCP   6h23m
dragonfly        ClusterIP   10.43.15.153   <none>        6379/TCP   7s
```
## Опціонально: "Чистий" GitOps

Якщо хочете "чистий" GitOps, налаштуйте встановлення самого Dragonfly Operator через Flux. Для цього в окремій папці (наприклад `infrastructure/controllers/dragonfly`) створіть **HelmRepository** (джерело чарту оператора) та **HelmRelease** (інсталяція оператора).
