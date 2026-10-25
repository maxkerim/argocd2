# GitOps-проект на Kubernetes

Учебный GitOps-проект: развёртывание приложений в Kubernetes через Argo CD и Helm. Dev и prod управляются отдельными корневыми Argo CD Applications, используют отдельные namespace и values-файлы.

## Архитектура

```mermaid
flowchart LR
   Git[GitHub] --> RootDev[Корневое Application dev]
   Git --> RootProd[Корневое Application prod]
   RootDev --> Dev[Dev Applications]
   RootProd --> Prod[Prod Applications]
    Dev --> Chart[Helm chart]
   Prod --> Chart
    Chart --> K8s[Deployment и Service в Kubernetes]
```

`root.yaml` следит только за `demo_dev/applications`, а `root_prod.yaml` — только за `demo_prod/applications`. Оба окружения используют `HelmCharts/MyChart1`, но разные values-файлы.

| Окружение | Файл values | Реплики | Образ |
| --- | --- | ---: | --- |
| Dev | `values_dev.yaml` | 4 | `nginx:alpine` |
| Prod | `values_prod.yaml` | 3 | `nginx:alpine` |

В чарте настроены HTTP readiness/liveness probes и ограничения CPU/памяти. Service по умолчанию имеет тип `ClusterIP`. Ingress можно включить отдельно, задав домен, класс контроллера и TLS.

## Структура

```text
.
├── root.yaml                    # Корневое Argo CD Application для dev
├── root_prod.yaml               # Корневое Argo CD Application для prod
├── demo_dev/applications/       # Applications для dev
├── demo_prod/applications/      # Applications для prod
├── .github/workflows/           # Проверки Helm в GitHub Actions
└── HelmCharts/
    ├── MyChart1/                # Chart, используемый Applications
    └── MyChart2/                # Дополнительный пример chart
```

## Требования

- Kubernetes-кластер и `kubectl`
- Argo CD, установленный в кластере
- Helm 3
- Доступ Argo CD к репозиторию `https://github.com/maxkerim/argocd2` (для приватного репозитория настройте credentials в Argo CD)

## Запуск

1. Если репозиторий закрытый, добавьте его credentials в Argo CD.
2. Зарегистрируйте корневое Application для нужного окружения (можно применить оба файла):

   ```sh
   kubectl apply -f root.yaml
   kubectl apply -f root_prod.yaml
   ```

Каждый корневой Application создаёт дочерние Applications только из своего каталога. Чтобы развернуть только dev или только prod, примените соответствующий файл.

3. Проверьте chart локально:

   ```sh
   helm lint HelmCharts/MyChart1
   helm lint HelmCharts/MyChart2
   helm template demo HelmCharts/MyChart1 -f HelmCharts/MyChart1/values_dev.yaml
   helm template demo HelmCharts/MyChart1 -f HelmCharts/MyChart1/values_prod.yaml
   ```

GitHub Actions автоматически запускает lint и проверяет рендеринг values при push и pull request.

## Как открыть приложение снаружи

`ClusterIP` доступен только внутри кластера. Для локальной проверки можно временно пробросить порт:

```sh
kubectl port-forward -n app1-prod svc/myapp1-prod-service 8080:80
```

После этого откройте `http://localhost:8080`.

В облачном кластере для внешнего IP можно изменить `service.type` в `HelmCharts/MyChart1/values.yaml` на `LoadBalancer` и отправить изменение в Git. Затем проверьте адрес:

```sh
kubectl get svc -n app1-prod
```

У этого способа обычно создаётся отдельный балансировщик для каждого Service, что может стоить денег. Для нескольких приложений обычно используют Ingress: один внешний балансировщик направляет запросы к разным сервисам по домену или URL-пути. В чарте Ingress доступен, но по умолчанию выключен.

### Настройка Ingress

Ingress требует установленного в кластере ingress-controller (например, NGINX Ingress Controller) и DNS-записи, указывающей на адрес этого контроллера. Чтобы включить его для production, настройте `HelmCharts/MyChart1/values_prod.yaml`:

```yaml
ingress:
   enabled: true
   className: nginx
   hosts:
      - host: app.example.com
         paths:
            - path: /
               pathType: Prefix
   tls:
      - secretName: app-tls
         hosts:
            - app.example.com
```

Замените `app.example.com` на свой домен, создайте DNS-запись на адрес ingress-controller и создайте TLS Secret `app-tls` в namespace приложения. Если TLS пока не нужен, задайте `tls: []` или удалите блок. Argo CD применит Ingress вместе с остальными ресурсами после отправки изменений в Git.

Также поддерживается `NodePort`, но он открывает порт на узлах кластера и обычно требует дополнительной настройки сети и firewall.

## Проверки и следующие шаги

- GitOps-доставка с автоматической синхронизацией, prune и self-heal.
- Разделение dev/prod по namespace и параметрам Helm.
- Проверки Helm chart в CI.
- Добавить тест валидации Kubernetes-манифестов.
- Добавить скриншоты статуса Argo CD после реального развёртывания.