# Etap 2: Metryki: Prometheus, Alertmanager i Grafana

**Cel:** wdrożyć stos metryk przez GitOps i zrozumieć model **pull**: Prometheus sam odpytuje (scrapuje) cele po HTTP, a cele wskazujemy deklaratywnie obiektami `ServiceMonitor` i `PodMonitor`.

**Warunek wstępny:** etap 1 łącznie z krokiem 1.7, czyli w klastrze działa aplikacja `root`.

```bash
kubectl -n argocd get application root
```

**Plan etapu:**

| Krok | Co robisz | Czego się uczysz |
|---|---|---|
| 2.1 | Poznajesz kube-prometheus-stack | Z czego składa się stos i czym jest Prometheus Operator |
| 2.2 | Tworzysz sekret z hasłem Grafany | Co trzymać poza Gitem i dlaczego |
| 2.3 | Piszesz `Application` i wdrażasz przez Git | Multi-source, Server-Side Apply, sync waves |
| 2.4 | Obserwujesz wdrożenie | Operator tworzy zasoby z CRD; rozmieszczenie podów |
| 2.5 | Prometheus: cele i PromQL | Service discovery, `up`, `rate()`, agregacje |
| 2.6 | Grafana | Datasource, gotowe dashboardy, Explore |
| 2.7 | Alertmanager | Reguły → alerty → Alertmanager; Watchdog |
| 2.8 | Ćwiczenia | ServiceMonitor, PodMonitor, własny alert, metryki Argo CD |

---

## 2.1 Poznaj kube-prometheus-stack

`kube-prometheus-stack` to chart Helm, który instaluje kompletny monitoring klastra:

| Komponent | Rola |
|---|---|
| **Prometheus Operator** | kontroler, który dodaje CRD (`Prometheus`, `Alertmanager`, `ServiceMonitor`, `PodMonitor`, `PrometheusRule`...) i na ich podstawie tworzy oraz konfiguruje Prometheusa i Alertmanagera |
| **Prometheus** | baza szeregów czasowych; co 30 s scrapuje cele i ocenia reguły |
| **Alertmanager** | odbiera alerty od Prometheusa, grupuje je, wycisza i wysyła dalej (e-mail, Slack…) |
| **Grafana** | wizualizacja: dashboardy i Explore |
| **node-exporter** | DaemonSet: metryki systemu operacyjnego każdego węzła (CPU, RAM, dysk, sieć) |
| **kube-state-metrics** | metryki **stanu obiektów** Kubernetes (ile replik jest gotowych, w jakiej fazie jest pod, czy PVC jest związany) |
| **ServiceMonitory i reguły** | gotowe cele (kubelet, cAdvisor, CoreDNS, apiserver) i ~30 grup reguł alertów oraz reguł nagrywających |

**Kluczowa idea Operatora:** nie edytujesz `prometheus.yml`. Tworzysz obiekt `ServiceMonitor` („scrapuj serwisy z etykietą X na porcie Y”), a Operator sam dopisuje go do konfiguracji Prometheusa i ją przeładowuje. Konfiguracja monitoringu staje się zwykłym manifestem Kubernetes, który można trzymać w Gicie obok aplikacji.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm search repo prometheus-community/kube-prometheus-stack --versions | head -3
helm show values prometheus-community/kube-prometheus-stack --version 93.1.2 > /tmp/kps-defaults.yaml
```

Przeczytaj [values/kube-prometheus-stack.yaml](../values/kube-prometheus-stack.yaml) i każdą sekcję znajdź w pliku domyślnym. Najważniejsze decyzje:

| Ustawienie | Dlaczego |
|---|---|
| `kubeControllerManager/kubeScheduler/kubeProxy/kubeEtcd.enabled: false` | W K3s te komponenty działają w jednym procesie `k3s` i nie wystawiają metryk na zewnątrz, a etcd nie istnieje (K3s z jednym serwerem używa SQLite). Bez wyłączenia cele byłyby na stałe `down` i odpalałyby alerty. |
| `nodeSelector` + `tolerations` | Backendy lądują na worker-3. Taint odpycha pody, a tolerancja tylko pozwala na worker-3, ale nie zmusza do niego; dlatego potrzebny jest też `nodeSelector`. |
| `prometheus-node-exporter.tolerations: [operator: Exists]` | DaemonSet ma działać na **każdym** węźle, także na worker-3 z taintem. |
| `*SelectorNilUsesHelmValues: false` | Domyślnie Prometheus bierze tylko ServiceMonitory z etykietą `release: kube-prometheus-stack`. Wyłączamy ten filtr: monitorowane będzie wszystko w klastrze. |
| `retention: 7d`, `retentionSize: 18GB`, PVC 20 Gi | Dane przetrwają restart; limit rozmiaru jest mniejszy niż wolumen, z zapasem na WAL. |
| `grafana.admin.existingSecret` | Hasło z sekretu poza Gitem (krok 2.2). |

Podgląd, co powstanie:

```bash
helm template kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --version 93.1.2 -n monitoring -f values/kube-prometheus-stack.yaml | grep '^kind:' | sort | uniq -c
```

Zwróć uwagę: nie ma tu żadnego `StatefulSet` dla Prometheusa, jest obiekt `kind: Prometheus`. StatefulSet utworzy dopiero Operator.

## 2.2 Sekret z hasłem Grafany

Hasło nie może trafić do Gita, bo repo jest publiczne. Tworzysz je ręcznie, a Grafana tylko wskazuje sekret po nazwie.

```bash
kubectl create namespace monitoring
kubectl -n monitoring create secret generic grafana-admin \
  --from-literal=admin-user=admin \
  --from-literal=admin-password='WYMYŚL-SILNE-HASŁO'
```

Dlaczego nie hasło losowane przez chart? Argo CD renderuje chart przy każdym porównaniu. Losowa wartość byłaby za każdym razem inna, więc aplikacja wiecznie byłaby `OutOfSync`, a hasło zmieniałoby się przy każdym syncu.

> To najprostsze podejście do sekretów w GitOps: sekret istnieje tylko w klastrze. Wersja „pełna” (Sealed Secrets / External Secrets) jest w etapie 8.

## 2.3 Wdrożenie przez Git

Utwórz plik `apps/kube-prometheus-stack.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kube-prometheus-stack
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  sources:
    - repoURL: https://prometheus-community.github.io/helm-charts
      chart: kube-prometheus-stack
      targetRevision: 93.1.2
      helm:
        releaseName: kube-prometheus-stack
        valueFiles:
          - $values/values/kube-prometheus-stack.yaml
    - repoURL: https://github.com/arturmachura/homlab-edu.git
      targetRevision: main
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

Co tu jest nowego względem `podinfo`:

| Element | Znaczenie |
|---|---|
| `sources` (liczba mnoga) | **Multi-source**: chart pochodzi z repo Helm, a values z naszego repo Git. `ref: values` nadaje drugiemu źródłu alias, a `$values/...` to ścieżka w tym repo. |
| `helm.releaseName` | Nazwa „release'u” wpływa na nazwy zasobów (`kube-prometheus-stack-grafana` itd.). Argo CD nie robi `helm install`, tylko renderuje szablony (`helm template`) i sam je aplikuje, dlatego `helm list -A` nie pokaże tej instalacji. |
| `ServerSideApply=true` | CRD Prometheus Operatora są ogromne. Zwykły `kubectl apply` zapisuje cały obiekt w adnotacji `last-applied-configuration` z limitem 256 KB, więc sync by się nie udał. Server-Side Apply nie używa tej adnotacji. |
| `sync-wave: "-1"` | Kolejność w obrębie `root`: ta aplikacja synchronizuje się przed aplikacjami z falą 0 (domyślną). Przyda się, gdy inne aplikacje zaczną używać `ServiceMonitor`. |
| `CreateNamespace=true` | Namespace już istnieje (krok 2.2), więc ta opcja niczego nie zmieni; zostawiamy ją dla odtwarzalności. |

Wdróż:

```bash
git add apps/kube-prometheus-stack.yaml values/kube-prometheus-stack.yaml docs/02-prometheus-grafana.md
git commit -m "Etap 2: kube-prometheus-stack"
git push
```

## 2.4 Obserwuj wdrożenie

W UI Argo CD: `root` zauważa nowy plik i tworzy aplikację `kube-prometheus-stack`, która zaczyna się synchronizować. Pierwszy sync trwa 2–4 minuty (pobieranie obrazów).

```bash
kubectl -n argocd get application kube-prometheus-stack -w
kubectl -n monitoring get pods -o wide -w
```

Na co zwrócić uwagę:

1. **Kolejność:** najpierw CRD, potem Job `admission-create` (generuje certyfikat dla webhooka walidującego reguły), Operator i reszta, na końcu Job `admission-patch`. W Argo CD to *hooki*, widoczne w drzewie aplikacji tylko w trakcie syncu.
2. **Kto tworzy Prometheusa:**
   ```bash
   kubectl -n monitoring get prometheus,alertmanager
   kubectl -n monitoring get statefulset
   kubectl -n monitoring get statefulset prometheus-kube-prometheus-stack-prometheus -o jsonpath='{.metadata.ownerReferences[0].kind}'; echo
   # Prometheus: właścicielem StatefulSetu jest obiekt Prometheus, nie Argo CD
   ```
3. **Rozmieszczenie:** Prometheus, Alertmanager, Grafana, Operator i kube-state-metrics na worker-3, a node-exporter na wszystkich 4 węzłach.
   ```bash
   kubectl -n monitoring get pods -o wide
   ```
4. **Wolumeny:**
   ```bash
   kubectl -n monitoring get pvc
   kubectl get pv -o custom-columns=NAME:.metadata.name,CLAIM:.spec.claimRef.name,NODE:.spec.nodeAffinity.required.nodeSelectorTerms[0].matchExpressions[0].values[0]
   ```
   `local-path` tworzy katalog na dysku węzła, a PV jest na stałe przypięty do tego węzła.

Gotowe, gdy aplikacja jest `Synced` / `Healthy`.

## 2.5 Prometheus: cele i PromQL

Otwórz http://prometheus.homelab.local.

### Cele (Status → Target health)

To lista wszystkiego, co Prometheus scrapuje: apiserver, kubelet (także `/metrics/cadvisor`, czyli metryki kontenerów), CoreDNS, node-exporter (4 cele, po jednym na węzeł), kube-state-metrics, Operator, Alertmanager, sam Prometheus. Wszystkie powinny być `UP`.

*Status → Service discovery* pokazuje, jak z `ServiceMonitor` powstają cele: ile endpointów znaleziono i ile odrzucono w relabelingu.

### PromQL: wpisuj w zakładce *Query*

```promql
# 1 = cel odpowiada, 0 = nie. Najważniejsza metryka w Prometheusie.
up

# Ile celów w każdym jobie jest UP
sum by (job) (up)

# Wolna pamięć na węzłach w GiB (node-exporter)
node_memory_MemAvailable_bytes / 1024^3

# Zajętość dysku głównego w %
100 * (1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})

# Zużycie CPU przez pody w namespace argocd (cAdvisor), w rdzeniach
sum by (pod) (rate(container_cpu_usage_seconds_total{namespace="argocd", container!=""}[5m]))

# Pamięć podów podinfo
container_memory_working_set_bytes{namespace="podinfo", container="podinfo"}

# Stan obiektów (kube-state-metrics): ile replik gotowych w każdym deploymencie
kube_deployment_status_replicas_available
```

Pojęcia:
- **Counter** (np. `container_cpu_usage_seconds_total`) tylko rośnie, więc liczy się z niego tempo przez `rate(...[5m])`.
- **Gauge** (np. `node_memory_MemAvailable_bytes`) to bieżąca wartość, używana wprost.
- **Etykiety** (`{namespace="podinfo"}`) filtrują, a `sum by (...)` agreguje.

Przełącz się na zakładkę *Graph*, żeby zobaczyć przebieg w czasie.

## 2.6 Grafana

Otwórz http://grafana.homelab.local i zaloguj się jako `admin` hasłem z kroku 2.2.

1. **Connections → Data sources**: datasource `Prometheus` jest już skonfigurowany. Utworzył go sidecar z ConfigMapy w chartcie, bez klikania.
2. **Dashboards**: kilkadziesiąt gotowych dashboardów z chartu. Zacznij od:
   - *Kubernetes / Compute Resources / Cluster*: CPU i RAM całego klastra,
   - *Kubernetes / Compute Resources / Namespace (Pods)*: wybierz `argocd`, potem `podinfo`,
   - *Node Exporter / Nodes*: porównaj worker-3 z resztą,
   - *Kubernetes / Persistent Volumes*: zajętość PVC Prometheusa.
3. **Explore**: wpisz zapytanie z kroku 2.5 i porównaj z UI Prometheusa. Explore służy do doraźnego badania, a dashboard do stałego obserwowania.

Zrób test: edytuj któryś panel gotowego dashboardu i spróbuj zapisać. Grafana nie pozwoli, bo dashboardy z ConfigMap są *provisioned*, czyli zarządzane kodem. Swoją wersję zapiszesz przez *Save as*.

## 2.7 Alertmanager

1. W Prometheusie otwórz *Alerts*: lista reguł i ich stanów (`inactive`, `pending`, `firing`).
2. Otwórz http://alertmanager.homelab.local.

Zawsze zobaczysz alert **Watchdog**: celowo odpala się bez przerwy. To „dead man's switch”: jeśli kiedykolwiek przestanie docierać do zewnętrznego systemu, oznacza to, że cały łańcuch alertowania nie działa.

Mogą też pojawić się alerty o K3s, np. `KubeClientCertificateExpiration`. Przeczytaj je i zastanów się, czy to faktyczny problem.

**Droga alertu:** reguła w `PrometheusRule` → Prometheus ocenia wyrażenie co 30 s → warunek spełniony przez czas `for` → `pending` przechodzi w `firing` → Prometheus wysyła alert do Alertmanagera → Alertmanager grupuje i wysyła do odbiorców (na razie nie ma żadnego).

## 2.8 Ćwiczenia

Rozwiązania są zwinięte. Spróbuj najpierw sam.

### Ćwiczenie 1: ServiceMonitor dla podinfo

podinfo wystawia metryki Prometheusa na porcie `9797` (`http-metrics` w [service.yaml](../manifests/podinfo/service.yaml)), ale nikt ich nie zbiera.

1. Napisz `manifests/podinfo/servicemonitor.yaml`, który scrapuje serwis z etykietą `app: podinfo` na porcie `http-metrics`.
2. Commit i push: aplikacja `podinfo` wdroży go razem z resztą swoich manifestów.
3. Sprawdź w *Target health*, że pojawił się cel `serviceMonitor/podinfo/podinfo/0` z 2 endpointami (2 repliki).
4. Wygeneruj ruch i znajdź metryki HTTP podinfo:
   ```bash
   for i in $(seq 1 300); do curl -s -o /dev/null http://podinfo.homelab.local/; done
   ```
   ```promql
   sum by (status) (rate(http_request_duration_seconds_count{namespace="podinfo"}[1m]))
   histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket{namespace="podinfo"}[5m])))
   ```

<details>
<summary>Rozwiązanie</summary>

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: podinfo
spec:
  selector:
    matchLabels:
      app: podinfo
  endpoints:
    - port: http-metrics
      interval: 15s
```

`port` odnosi się do **nazwy** portu w Service, nie do numeru. Namespace nie jest potrzebny, bo Argo CD wdraża do `podinfo`, a ServiceMonitor domyślnie szuka serwisów we własnym namespace.
</details>

**Wniosek:** monitoring aplikacji żyje w repo obok aplikacji i jest wdrażany tym samym commitem.

### Ćwiczenie 2: PodMonitor dla Traefika (nowa aplikacja Argo CD)

Traefik z K3s wystawia metryki na porcie kontenera `metrics` (9100), ale nie ma Service, który by go wystawiał. Do takich przypadków służy `PodMonitor`: scrapuje pody bezpośrednio.

1. Utwórz katalog `manifests/monitoring-extras/` z plikiem `traefik-podmonitor.yaml`. Pody Traefika są w `kube-system` i mają etykietę `app.kubernetes.io/name: traefik`.
2. Utwórz `apps/monitoring-extras.yaml`: nową `Application` wdrażającą ten katalog do namespace `monitoring`. Wzoruj się na [apps/podinfo.yaml](../apps/podinfo.yaml).
3. Commit i push, a potem sprawdź cel w Prometheusie.
4. Zapytania:
   ```promql
   sum by (service, code) (rate(traefik_service_requests_total[5m]))
   ```
5. Zaimportuj dashboard Traefika: w Grafanie *Dashboards → New → Import*, ID **17346**, datasource Prometheus.

<details>
<summary>Rozwiązanie</summary>

`manifests/monitoring-extras/traefik-podmonitor.yaml`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: traefik
spec:
  namespaceSelector:
    matchNames: [kube-system]
  selector:
    matchLabels:
      app.kubernetes.io/name: traefik
  podMetricsEndpoints:
    - port: metrics
```

`apps/monitoring-extras.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: monitoring-extras
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/arturmachura/homlab-edu.git
    targetRevision: main
    path: manifests/monitoring-extras
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```
</details>

### Ćwiczenie 3: Własny alert

Napisz regułę, która odpala się, gdy podinfo ma mniej niż 2 gotowe repliki przez minutę.

1. Utwórz `manifests/podinfo/prometheusrule.yaml` (podpowiedź: metryka `kube_deployment_status_replicas_available` z kube-state-metrics).
2. Commit i push, a potem znajdź regułę w Prometheusie (*Alerts* oraz *Status → Rule health*).
3. Wywołaj alert **przez Git**: zmień `replicas: 2` na `1` w `deployment.yaml`, commit i push.
4. Obserwuj: alert przechodzi `inactive` → `pending` → po minucie `firing`, a w Alertmanagerze pojawia się `PodinfoReplicasMissing`.
5. Przywróć 2 repliki (`git revert HEAD`, push) i obserwuj, jak alert gaśnie.

<details>
<summary>Rozwiązanie</summary>

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: podinfo
spec:
  groups:
    - name: podinfo
      rules:
        - alert: PodinfoReplicasMissing
          expr: kube_deployment_status_replicas_available{namespace="podinfo", deployment="podinfo"} < 2
          for: 1m
          labels:
            severity: warning
          annotations:
            summary: "podinfo ma {{ $value }} z 2 replik"
            description: "Deployment podinfo działa w niepełnym składzie od ponad minuty."
```

Zwróć uwagę, że ręczne `kubectl scale` nie wywoła alertu na dłużej: `selfHeal` z etapu 1 przywróci repliki szybciej niż minutę z `for`.
</details>

> Webhook walidujący Operatora odrzuci regułę z błędem składni PromQL już przy syncu, a Argo CD pokaże błąd. Sprawdź to, celowo psując `expr`.

### Ćwiczenie 4: Metryki samego Argo CD

Argo CD wystawia własne metryki (liczba syncy, czas renderowania, stan aplikacji). Włącz je **przez Git**: Argo CD zarządza samym sobą (krok 1.8), więc wystarczy zmienić [values/argocd.yaml](../values/argocd.yaml).

1. Włącz `metrics.enabled` i `metrics.serviceMonitor.enabled` dla `controller`, `server` i `repoServer` (podpowiedź: `helm show values argo/argo-cd --version 10.17.1 | grep -n -A12 "^  metrics:"`).
2. Commit i push, a potem 3 nowe cele w Prometheusie.
3. Zapytania:
   ```promql
   argocd_app_info
   sum by (name, phase) (increase(argocd_app_sync_total[1h]))
   ```
4. Zaimportuj dashboard Argo CD: ID **14584**.

<details>
<summary>Rozwiązanie</summary>

Dopisz do `values/argocd.yaml` (scalając z istniejącym blokiem `server:`):

```yaml
controller:
  metrics:
    enabled: true
    serviceMonitor:
      enabled: true
server:
  metrics:
    enabled: true
    serviceMonitor:
      enabled: true
repoServer:
  metrics:
    enabled: true
    serviceMonitor:
      enabled: true
```
</details>

### Ćwiczenie 5 (opcjonalne): Drift w monitoringu

Usuń ręcznie Grafanę i obserwuj, co zrobi Argo CD:

```bash
kubectl -n monitoring delete deployment kube-prometheus-stack-grafana
```

Czy po odtworzeniu nadal jesteś zalogowany i czy Twój zapisany dashboard (*Save as* z kroku 2.6) przetrwał? Dlaczego? (Podpowiedź: `persistence` w values.)

## Rozwiązywanie problemów

| Objaw | Przyczyna i rozwiązanie |
|---|---|
| Sync: `metadata.annotations: Too long` | Brak `ServerSideApply=true` w `syncOptions`. |
| Grafana `CreateContainerConfigError` | Brak sekretu `grafana-admin` w `monitoring` (krok 2.2): `kubectl -n monitoring describe pod -l app.kubernetes.io/name=grafana`. |
| Pod `Pending` z komunikatem `untolerated taint` | Komponent bez tolerancji próbuje trafić na worker-3, albo ma `nodeSelector` bez tolerancji. `kubectl describe pod` pokaże szczegóły. |
| ServiceMonitor nie tworzy celu | Sprawdź po kolei: etykiety Service vs `selector`; **nazwa** portu w Service vs `endpoints.port`; *Status → Service discovery* w Prometheusie (ile odrzuconych). |
| `kubectl exec` do Prometheusa nie działa | Obrazy są *distroless* (bez powłoki). Do diagnostyki używaj UI albo `kubectl debug`. |
| `KubeClientCertificateExpiration` / inne alerty K3s | Reguły z chartu zakładają „pełny” Kubernetes. Przeanalizuj, a jeśli to fałszywy alarm, wyłącz regułę w `defaultRules.disabled`. |

## Podsumowanie

- Prometheus działa w modelu **pull**, a cele opisujemy deklaratywnie (`ServiceMonitor`, `PodMonitor`), trzymając je w Gicie obok aplikacji.
- Prometheus Operator zamienia CRD na konfigurację i zasoby (StatefulSet, ConfigMapy, reguły).
- Alert to reguła PromQL + `for` → Alertmanager.
- Zmiany monitoringu, łącznie z monitoringiem samego Argo CD, wdrażamy commitem.

**Następny etap:** Loki i Tempo, czyli logi i trace'y w Grafanie obok metryk.
