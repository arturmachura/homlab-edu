# Etap 3: Logi i trace'y: Loki, Tempo i OpenTelemetry Collector

**Cel:** zbudować drogę dla wszystkich trzech sygnałów telemetrii i połączyć je w Grafanie, tak żeby z metryki przejść do trace'a, a z trace'a do logów.

**Warunek wstępny:** etap 2 działa (`kube-prometheus-stack` jest `Synced` / `Healthy`).

## Architektura etapu

```
                     ┌──────────────── worker-3 (monitoring) ───────────────┐
 aplikacje ──OTLP──┐ │                                                      │
                   ▼ │                                                      │
 ┌─ każdy węzeł ──────────────┐  trace'y  ┌───────┐  span metrics           │
 │ OpenTelemetry Collector    ├──────────►│ Tempo ├──remote write─┐         │
 │ (DaemonSet "agent")        │  logi     ├───────┤               ▼         │
 │  • odbiera OTLP            ├──────────►│ Loki  │        ┌────────────┐   │
 │  • czyta /var/log/pods     │  metryki  └───────┘        │ Prometheus │   │
 │  • dokleja metadane k8s    ├───────────────────────────►└────────────┘   │
 └────────────────────────────┘ │                              ▲            │
                     │          Grafana ── zapytania do Loki, Tempo, Prometheusa
                     └──────────────────────────────────────────────────────┘
```

**Dlaczego kolektor, a nie wysyłka prosto do Loki i Tempo?**
- Aplikacja zna tylko jeden adres (`otel-collector:4317`), a o tym, dokąd trafiają dane, decyduje konfiguracja kolektora. W etapie 7 dołożymy Grafana Cloud bez dotykania aplikacji.
- Kolektor wzbogaca dane o metadane Kubernetes (namespace, pod, deployment, węzeł), których aplikacja sama nie zna.
- Kolektor buforuje, ponawia, filtruje i próbkuje.

**Dlaczego DaemonSet?** Agent na każdym węźle czyta pliki logów wszystkich podów z dysku węzła (`/var/log/pods`), a aplikacje wysyłają telemetrię do agenta na **swoim** węźle (`internalTrafficPolicy: Local`), więc ruch nie przechodzi między węzłami.

**Plan etapu:**

| Krok | Co robisz | Czego się uczysz |
|---|---|---|
| 3.1 | Poznajesz charty i values | Tryby wdrożenia Loki i Tempo, budowa pipeline'u kolektora |
| 3.2 | Wdrażasz Loki i Tempo | Backendy logów i trace'ów, API do diagnostyki |
| 3.3 | Rozbudowujesz Prometheusa i Grafanę | Remote write, OTLP, exemplary, powiązane datasource'y |
| 3.4 | Wdrażasz kolektor | Receivery, procesory, exportery, pipeline'y |
| 3.5 | Przeglądasz logi klastra | LogQL, etykiety a structured metadata |
| 3.6 | Instrumentujesz podinfo przez Git | Zmienne `OTEL_*`, pierwsze trace'y, TraceQL |
| 3.7 | Korelujesz sygnały | Metryka → exemplar → trace → logi |
| 3.8 | Ćwiczenia | |

---

## 3.1 Poznaj charty

Charty Loki, Tempo i Grafany zostały przeniesione z `grafana` do repozytorium **`grafana-community`**. Stare wersje w `grafana/...` nie są rozwijane.

```bash
helm repo add grafana-community https://grafana-community.github.io/helm-charts
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update
helm search repo grafana-community/loki
helm search repo grafana-community/tempo
helm search repo open-telemetry/opentelemetry-collector
```

| Chart | Wersja | Aplikacja |
|---|---|---|
| `grafana-community/loki` | 21.2.0 | Loki 3.7.9 |
| `grafana-community/tempo` | 3.2.1 | Tempo 3.1.1 |
| `open-telemetry/opentelemetry-collector` | 0.178.3 | Collector (contrib) 0.163.2 |

Przeczytaj values i znajdź każde ustawienie w `helm show values`:

**[values/loki.yaml](../values/loki.yaml)**

| Ustawienie | Po co |
|---|---|
| `deploymentMode: Monolithic` | Wszystkie komponenty Loki w jednym procesie. Pozostałe tryby (SimpleScalable, Distributed) wymagają magazynu obiektowego (S3). |
| `read/write/backend.replicas: 0` | Wyłączenie trybu SimpleScalable; chart nie pozwala, by oba tryby miały repliki. |
| `auth_enabled: false` | Bez multi-tenancy; nie trzeba wysyłać nagłówka `X-Scope-OrgID`. |
| `storage.type: filesystem` + `schemaConfig` | Chunki i indeks (TSDB, schemat v13) na PVC zamiast S3. |
| `allow_structured_metadata: true` | Atrybuty OTLP, które nie są etykietami (np. `trace_id`), trafiają do *structured metadata*. Są przeszukiwalne, ale nie tworzą nowych strumieni. |
| `retention_period` + `compactor.retention_enabled` | Logi starsze niż 7 dni są usuwane. Bez compactora retencja nie działa. |
| `gateway/chunksCache/resultsCache/lokiCanary: false` | Komponenty dla dużych instalacji; cache to memcached zjadający kilka GB RAM. |

**[values/tempo.yaml](../values/tempo.yaml)**

| Ustawienie | Po co |
|---|---|
| `retention: 72h` | Trace'y zajmują dużo miejsca; 3 dni wystarczą. |
| `metricsGenerator` + `overrides.defaults.metrics_generator.processors` | Tempo liczy z trace'ów metryki RED (`span-metrics`) i graf zależności serwisów (`service-graphs`) i wysyła je do Prometheusa przez *remote write*. |
| `send_exemplars: true` | Do metryk dołączane są przykładowe `trace_id`; dzięki nim z wykresu przejdziesz do konkretnego trace'a. |

**[values/otel-collector.yaml](../values/otel-collector.yaml)**: najważniejszy plik etapu.

| Ustawienie | Po co |
|---|---|
| `mode: daemonset` | Agent na każdym węźle. |
| `image.repository: ...-contrib` | Dystrybucja z komponentami spoza rdzenia (`k8sattributes`, `filelog`). |
| `presets.logsCollection` | Dodaje receiver `filelog` (czyta `/var/log/pods/*/*/*.log`), montuje katalogi węzła i zapisuje pozycję odczytu (`storeCheckpoints`). |
| `presets.kubernetesAttributes` | Dodaje processor `k8sattributes` i uprawnienia RBAC do odczytu podów. |
| `tolerations: [operator: Exists]` | Agent także na worker-3. |
| `service.enabled: true` | Dla DaemonSet Service trzeba włączyć jawnie; chart ustawia `internalTrafficPolicy: Local`. |
| `config.exporters` | Trzy cele: Tempo (OTLP gRPC), Loki (OTLP HTTP, `/otlp`), Prometheus (OTLP HTTP, `/api/v1/otlp`). |
| `config.service.pipelines` | Dla każdego sygnału: skąd (`receivers`) → co zrobić (`processors`) → dokąd (`exporters`). |

Zobacz konfigurację, którą faktycznie dostanie kolektor, czyli z dołożonymi presetami:

```bash
helm template otel-collector open-telemetry/opentelemetry-collector --version 0.183.2 \
  -n otel -f values/otel-collector.yaml | sed -n '/relay: |/,/^---/p'
```

Zwróć uwagę na nowe nazwy komponentów (`otlp_grpc`, `otlp_http`, `k8s_attributes`, `file_log`). Kolektor od wersji ~0.160 przechodzi na `snake_case`, a chart przepisuje stare nazwy (`rewriteDeprecatedComponentNames`). W starszych poradnikach zobaczysz `otlp`, `otlphttp`, `k8sattributes`, `filelog`.

## 3.2 Loki i Tempo

Utwórz dwa pliki w `apps/`.

`apps/loki.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: loki
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  sources:
    - repoURL: https://grafana-community.github.io/helm-charts
      chart: loki
      targetRevision: 21.2.0
      helm:
        releaseName: loki
        valueFiles:
          - $values/values/loki.yaml
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
```

`apps/tempo.yaml`: analogicznie. Samodzielnie zmień `name`, `chart` (`tempo`), `targetRevision` (`3.2.1`), `releaseName` i plik values.

```bash
git add apps/loki.yaml apps/tempo.yaml values/loki.yaml values/tempo.yaml docs/03-loki-tempo-otel.md
git commit -m "Etap 3: Loki i Tempo"
git push
```

Sprawdzenie:

```bash
kubectl -n monitoring get pods -o wide | grep -E 'loki|tempo'   # oba na worker-3
kubectl -n monitoring get pvc | grep -E 'loki|tempo'
```

Loki i Tempo nie mają UI; rozmawia się z nimi przez API. Przyda się to przy diagnostyce:

```bash
kubectl -n monitoring port-forward svc/loki 3100 &
curl -s localhost:3100/ready                      # ready
curl -s localhost:3100/loki/api/v1/labels         # na razie pusto: nikt nie wysyła logów
kill %1

kubectl -n monitoring port-forward svc/tempo 3200 &
curl -s localhost:3200/ready
curl -s localhost:3200/status/config | grep -A3 metrics_generator
kill %1
```

## 3.3 Prometheus i Grafana: odbiorniki i datasource'y

Teraz Ty rozbudowujesz istniejący plik [values/kube-prometheus-stack.yaml](../values/kube-prometheus-stack.yaml). Uważaj na wcięcia i scal bloki z istniejącymi kluczami, nie twórz duplikatów `prometheus:` ani `grafana:`.

**a) W `prometheus.prometheusSpec` dopisz:**

```yaml
    # Tempo wysyła metryki z trace'ów metodą push (remote write).
    enableRemoteWriteReceiver: true
    # Kolektor wysyła metryki aplikacji przez OTLP (endpoint /api/v1/otlp).
    enableOTLPReceiver: true
    # Przechowywanie exemplarów: przykładowych trace_id przy próbkach metryk.
    enableFeatures:
      - exemplar-storage
```

Prometheus to nie tylko pull. Te dwie opcje włączają odbiór danych wypychanych (push): przez remote write (format Prometheusa) i przez OTLP (format OpenTelemetry).

**b) W `grafana` dopisz:**

```yaml
  sidecar:
    datasources:
      # Metryka z exemplarem → link do trace'a w Tempo.
      exemplarTraceIdDestinations:
        datasourceUid: tempo
        traceIdLabelName: trace_id
        urlDisplayLabel: Pokaż trace
  additionalDataSources:
    - name: Loki
      uid: loki
      type: loki
      url: http://loki.monitoring.svc.cluster.local:3100
      access: proxy
      jsonData:
        # Log z polem trace_id → link do trace'a w Tempo.
        derivedFields:
          - name: TraceID
            matcherType: label        # szukaj w etykietach/structured metadata, nie w treści
            matcherRegex: trace_id
            datasourceUid: tempo
            url: "$${__value.raw}"    # $$ = dosłowny $ w provisioningu Grafany
            urlDisplayLabel: Pokaż trace
    - name: Tempo
      uid: tempo
      type: tempo
      url: http://tempo.monitoring.svc.cluster.local:3200
      access: proxy
      jsonData:
        tracesToLogsV2:               # span → logi z tego samego trace'a
          datasourceUid: loki
          spanStartTimeShift: "-5m"
          spanEndTimeShift: "5m"
          filterByTraceID: true
        tracesToMetrics:              # span → metryki serwisu
          datasourceUid: prometheus
        serviceMap:                   # mapa serwisów z metryk service-graphs
          datasourceUid: prometheus
        nodeGraph:
          enabled: true
        lokiSearch:
          datasourceUid: loki
```

`uid` to stały identyfikator datasource'a, przez który datasource'y odwołują się do siebie nawzajem (`datasourceUid: tempo`). Datasource Prometheusa z chartu ma `uid: prometheus`.

Sprawdź przed commitem, czy chart się renderuje:

```bash
helm template kube-prometheus-stack prometheus-community/kube-prometheus-stack --version 93.3.0 \
  -n monitoring -f values/kube-prometheus-stack.yaml > /dev/null && echo OK
```

Commit i push. Operator zrestartuje Prometheusa z nowymi flagami (dane zostają na PVC), a Grafana przeładuje datasource'y.

```bash
kubectl -n monitoring get pod prometheus-kube-prometheus-stack-prometheus-0 \
  -o jsonpath='{.spec.containers[0].args}' | tr ',' '\n' | grep -E 'remote-write|otlp|exemplar'
```

W Grafanie: *Connections → Data sources*: Loki i Tempo, przycisk *Test* na dole każdego.

## 3.4 OpenTelemetry Collector

`apps/otel-collector.yaml`: napisz sam. Wskazówki:
- repo Helm: `https://open-telemetry.github.io/opentelemetry-helm-charts`, chart `opentelemetry-collector`, wersja `0.178.3`,
- `releaseName: otel-collector`, values: `values/otel-collector.yaml`,
- namespace docelowy: **`otel`** (z `CreateNamespace=true`). Ta nazwa jest ważna, bo preset `logsCollection` wyklucza własne logi kolektora po ścieżce `/var/log/pods/otel_otel-collector*`.

Commit i push, potem:

```bash
kubectl -n otel get pods -o wide              # 4 pody: po jednym na węzeł, także worker-3
kubectl -n otel get svc otel-collector -o jsonpath='{.spec.internalTrafficPolicy}'; echo
kubectl -n otel logs ds/otel-collector-agent | tail -20
```

W logach kolektora nie powinno być błędów eksportu (`Exporting failed`). Pojedyncze ostrzeżenia na starcie są normalne.

Kolektor od razu zaczyna czytać logi **wszystkich** podów w klastrze. Sprawdź:

```bash
kubectl -n monitoring port-forward svc/loki 3100 &
curl -s localhost:3100/loki/api/v1/labels | tr ',' '\n'
curl -s localhost:3100/loki/api/v1/label/k8s_namespace_name/values
kill %1
```

W Prometheusie (*Target health*) pojawił się `podMonitor/otel/otel-collector-agent/0`: 4 cele z metrykami samego kolektora. Zapytanie:

```promql
sum by (exporter) (rate(otelcol_exporter_sent_log_records_total[5m]))
```

## 3.5 Logi w Grafanie

*Explore* → datasource **Loki**.

**Etykiety:** Loki zamienia atrybuty zasobu OTLP na etykiety, a kropki na podkreślenia (`k8s.namespace.name` → `k8s_namespace_name`). Etykiety tworzą *strumienie* i muszą mieć małą liczbę wartości. Reszta atrybutów trafia do *structured metadata*.

LogQL, wpisuj w trybie *Code*:

```logql
# Wszystkie logi z namespace argocd
{k8s_namespace_name="argocd"}

# Tylko linie zawierające "error" (wielkość liter ignorowana)
{k8s_namespace_name="argocd"} |~ "(?i)error"

# Logi Argo CD są w formacie JSON: parsowanie i filtr po polu
{k8s_namespace_name="argocd", k8s_container_name="argocd-application-controller"} | json | level="info"

# Metryka z logów: liczba linii na sekundę per namespace
sum by (k8s_namespace_name) (rate({k8s_namespace_name=~".+"}[5m]))
```

Otwórz też **Drilldown → Logs**: przeglądanie bez pisania zapytań, z zakładką *Patterns*, czyli automatycznie wykrytymi wzorcami linii (`pattern_ingester`).

## 3.6 Instrumentacja podinfo

podinfo ma wbudowany OpenTelemetry SDK (Go). Wysyła trace'y i logi przez OTLP gRPC, gdy dostanie nazwę serwisu, a adres kolektora bierze ze standardowych zmiennych `OTEL_*`, takich samych w każdym języku.

W [manifests/podinfo/deployment.yaml](../manifests/podinfo/deployment.yaml):

1. do `command` dopisz argument:
   ```yaml
               - --otel-service-name=podinfo
   ```
2. do `env` dopisz:
   ```yaml
               # Adres kolektora: agent na tym samym węźle (internalTrafficPolicy: Local)
               - name: OTEL_EXPORTER_OTLP_ENDPOINT
                 value: http://otel-collector.otel.svc.cluster.local:4317
               - name: OTEL_EXPORTER_OTLP_INSECURE
                 value: "true"
   ```

Commit i push, potem generuj ruch, w tym wolne żądania i błędy:

```bash
for i in $(seq 1 100); do
  curl -s -o /dev/null http://podinfo.homelab.local/
  curl -s -o /dev/null http://podinfo.homelab.local/delay/1
  curl -s -o /dev/null http://podinfo.homelab.local/status/500
done
```

*Explore* → **Tempo** → *Search*: Service Name `podinfo` → kliknij trace. Zobaczysz span HTTP z atrybutami (`http.route`, `http.response.status_code`) i zasobem (`service.name`, `k8s.pod.name`, `k8s.node.name`, które dokleił kolektor).

TraceQL (zakładka *TraceQL*):

```traceql
{ resource.service.name = "podinfo" }
{ resource.service.name = "podinfo" && status = error }
{ resource.service.name = "podinfo" && duration > 500ms }
{ resource.service.name = "podinfo" } | rate() by (span.http.route)
```

Ostatnie zapytanie to **TraceQL metrics**: metryka liczona prosto z trace'ów, w formie wykresu.

**Drilldown → Traces** pokazuje to samo bez pisania zapytań: rate, błędy i czasy z podziałem na atrybuty.

Metryki z `metrics_generator` Tempo trafiły do Prometheusa:

```promql
# RED: liczba wywołań per operacja
sum by (span_name) (rate(traces_spanmetrics_calls_total{service="podinfo"}[5m]))
# p95 czasu
histogram_quantile(0.95, sum by (le, span_name) (rate(traces_spanmetrics_latency_bucket{service="podinfo"}[5m])))
```

## 3.7 Korelacja sygnałów

To najważniejszy efekt etapu.

1. **Metryka → trace:** w *Explore* (Prometheus) wpisz zapytanie `histogram_quantile` z poprzedniego kroku i włącz *Exemplars*. Na wykresie pojawią się kropki: każda to konkretny trace. Najedź na kropkę przy wysokim czasie i kliknij *Pokaż trace*.
2. **Trace → logi:** w widoku trace'a kliknij ikonę logów przy spanie (*Logs for this span*). Grafana zapyta Loki o logi z tym `trace_id`.
3. **Log → trace:** w Loki wpisz `{service_name="podinfo"}`, rozwiń linię, która ma pole `trace_id`, i kliknij *Pokaż trace*.
4. **Mapa serwisów:** Tempo → *Service Graph*. Na razie jeden węzeł; w etapie 4 będzie ich kilkanaście.

## 3.8 Ćwiczenia

### Ćwiczenie 1: Dwie drogi logów

Logi podinfo mogą trafiać do Loki dwiema drogami: z pliku na węźle (`filelog` czyta stderr kontenera) i z SDK przez OTLP. Sprawdź, co faktycznie się dzieje.

1. Wpisz `{service_name="podinfo"}` i rozwiń kilka linii. Czy widzisz dwie kopie tego samego zdarzenia? Po czym poznasz źródło (podpowiedź: logi z pliku mają `log_file_path`)?
2. Które linie mają `trace_id` i `span_id`? Dlaczego tylko one?
3. Zastanów się: która droga jest lepsza dla aplikacji, którą kontrolujesz, a która dla gotowego oprogramowania (np. Traefik, CoreDNS)?

<details>
<summary>Komentarz</summary>

Logi z SDK mają kontekst trace'a (`trace_id`, `span_id`), poziom logu w polu `severity` i atrybuty strukturalne bez parsowania. Logi z pliku to surowy tekst, który trzeba parsować, ale działają dla każdego kontenera bez zmian w kodzie. W produkcji wybiera się jedną drogę na aplikację, żeby nie płacić podwójnie za przechowywanie. Na przykład `filelog` wyklucza namespace'y aplikacji, które wysyłają logi przez OTLP (`exclude` w receiverze), albo aplikacja wyłącza OTLP dla logów.
</details>

### Ćwiczenie 2: Rozproszony trace

Jeden serwis to jeszcze nie rozproszony trace. podinfo potrafi przekazywać żądanie dalej: endpoint `/echo` wywołuje adresy z `--backend-url`.

1. Dodaj w `manifests/podinfo/` drugi Deployment i Service `podinfo-backend` (skopiuj i zmień nazwy oraz etykiety, `--otel-service-name=podinfo-backend`, 1 replika).
2. W głównym podinfo dodaj argument `--backend-url=http://podinfo-backend:80/echo`.
3. Commit i push, potem `curl -X POST -d 'hello' http://podinfo.homelab.local/echo`.
4. W Tempo znajdź trace z dwoma serwisami. Jak kontekst trace'a przeszedł między nimi? (Podpowiedź: nagłówek `traceparent`, W3C Trace Context.)
5. Zobacz *Service Graph*: krawędź `podinfo → podinfo-backend`.

Jeśli zamiast jednego trace'a zobaczysz dwa osobne (każdy z jednym serwisem), klient HTTP w aplikacji nie wstrzykuje nagłówka `traceparent`. To częsty błąd instrumentacji: serwer jest zinstrumentowany, a klient nie. Zanotuj to, bo przy instrumentacji „od zera” w etapie 4 ten temat wróci.

### Ćwiczenie 3: Debugowanie kolektora

1. Przez Git dodaj do `values/otel-collector.yaml` exporter `debug` z `verbosity: detailed` i dopisz go do pipeline'u `traces` (pamiętaj, że listę `exporters` trzeba podać w całości).
2. Wygeneruj ruch do podinfo i obejrzyj `kubectl -n otel logs ds/otel-collector-agent`: zobaczysz pełne spany w formie tekstowej.
3. Wycofaj zmianę (`git revert`), bo debug przy dużym ruchu zalewa logi.

### Ćwiczenie 4: Kto najwięcej loguje?

1. W *Drilldown → Logs* sprawdź, który namespace i który kontener produkuje najwięcej logów.
2. To samo zapytaniem: `topk(5, sum by (k8s_namespace_name, k8s_container_name) (bytes_over_time({k8s_namespace_name=~".+"}[1h])))`.
3. Zastanów się, czy wszystkie te logi są potrzebne. W etapie 7 (Grafana Cloud) za każdy GB się płaci.

## Rozwiązywanie problemów

| Objaw | Przyczyna i rozwiązanie |
|---|---|
| VS Code podkreśla `values/loki.yaml` („Property ... is not allowed”) | Rozszerzenie YAML dopasowało do nazwy pliku schemat konfiguracji samego Loki, a nie values chartu. Fałszywy alarm: rozstrzyga `helm template`. |
| `helm template` Loki: „more than zero replicas configured for both the monolithic and simple scalable targets” | Brak `read/write/backend.replicas: 0`. |
| Kolektor: `Exporting failed ... 404` dla Prometheusa | Brak `enableOTLPReceiver: true` (krok 3.3) albo Prometheus jeszcze się nie zrestartował. |
| Tempo: błędy remote write w logach (`/api/v1/write ... 404`) | Brak `enableRemoteWriteReceiver: true`. |
| Kolektor: `Exporting failed ... connection refused` dla Loki/Tempo | Backend jeszcze nie działa albo zła nazwa serwisu. Sprawdź `kubectl -n monitoring get svc loki tempo`. |
| Brak logów z danego węzła | Czy pod kolektora działa na tym węźle? `kubectl -n otel get pods -o wide`. |
| W Tempo brak trace'ów podinfo | `kubectl -n podinfo logs deploy/podinfo` (błędy eksportera OTLP?), zmienne `OTEL_*` w podzie, logi kolektora. |
| Brak *Pokaż trace* przy logach | Link pojawia się tylko przy liniach z `trace_id` w structured metadata, czyli przy logach z SDK, nie z pliku. |
| Brak exemplarów na wykresie | Exemplary mają tylko metryki z Tempo (`traces_spanmetrics_*`); potrzebna flaga `exemplar-storage` i włączony przełącznik *Exemplars*. |

## Podsumowanie

- Trzy sygnały, trzy backendy: metryki w Prometheusie, logi w Loki, trace'y w Tempo. Jeden kolektor zbiera i rozprowadza wszystko.
- Prometheus przyjmuje dane także metodą push: remote write i OTLP.
- Instrumentacja aplikacji to SDK w kodzie plus standardowe zmienne `OTEL_*`. Dokąd dane trafiają, decyduje kolektor.
- Wartość obserwowalności bierze się z **korelacji**: `trace_id` łączy metrykę (exemplar), trace i logi.

**Następny etap:** OpenTelemetry Demo, czyli kilkanaście zinstrumentowanych mikroserwisów wysyłających telemetrię do zbudowanej tu infrastruktury.
