# homelab-edu: obserwowalność na K3s

Lab do ćwiczenia pełnego łańcucha obserwowalności na domowym klastrze K3s:

- **OpenTelemetry**: instrumentacja aplikacji ([OpenTelemetry Demo](https://github.com/open-telemetry/opentelemetry-demo)) i OpenTelemetry Collector
- **GitOps**: wszystko wdraża **Argo CD** z tego repozytorium
- **Prometheus**: metryki
- **Grafana OSS**: Grafana, Loki (logi), Tempo (trace'y)
- **Chaos engineering**: Chaos Mesh oraz feature flagi w demo
- **Grafana Cloud**: ten sam strumień telemetrii wysłany do chmury

## Architektura docelowa

```
            GitHub: arturmachura/homlab-edu
                         │  Argo CD sprawdza repo co 60 s
                         ▼
                      Argo CD  ───────────── wdraża wszystko poniżej
                         │
 ┌──────── otel-demo (k3s, worker-1, worker-2) ────┐   ┌──── monitoring (worker-3) ────┐
 │ Astronomy Shop (mikroserwisy + OTel SDK)        │   │ Prometheus ◄── scrape k8s     │
 │        │ OTLP                                   │   │ Loki                          │
 │        ▼                                        │   │ Tempo                         │
 │ OpenTelemetry Collector ── metryki/logi/trace'y ┼──►│ Grafana OSS                   │
 └────────┬────────────────────────────────────────┘   └───────────────────────────────┘
          │ OTLP
          ▼
   Grafana Cloud (Metrics / Logs / Traces, Application Observability)

 Chaos Mesh ── wstrzykuje awarie w otel-demo (sieć, pody, CPU)
```

## Klaster

| Węzeł | Rola | CPU | RAM | Dysk |
|---|---|---|---|---|
| k3s | control-plane + aplikacje | 4 | 11 GiB | 29 GiB |
| worker-1 | aplikacje | 6 | 11 GiB | 29 GiB |
| worker-2 | aplikacje | 6 | 11 GiB | 29 GiB |
| worker-3 | obserwowalność (taint `dedicated=observability:NoSchedule`) | 8 | 23 GiB | 162 GiB |

Usługi są dostępne przez Traefik pod adresami `*.homelab.local` (wpisy w pliku hosts, patrz [etap 0](docs/00-przygotowanie.md)).

## Sposób pracy

Każdy krok wykonujesz ręcznie, rozumiejąc, co robi: bez skryptów „uruchom i zapomnij”. Instrukcje w `docs/` wyjaśniają, **po co** jest każda komenda i **co sprawdzić** po jej wykonaniu.

## Etapy

| # | Etap | Instrukcja | Status |
|---|---|---|---|
| 0 | Przygotowanie | [docs/00-przygotowanie.md](docs/00-przygotowanie.md) | ✅ |
| 1 | GitOps z Argo CD | [docs/01-argocd.md](docs/01-argocd.md) | ⏳ |
| 2 | Metryki: Prometheus, Alertmanager i Grafana | [docs/02-prometheus-grafana.md](docs/02-prometheus-grafana.md) | |
| 3 | Logi i trace'y: Loki i Tempo | — | |
| 4 | Aplikacja z OpenTelemetry | — | |
| 5 | Awarie aplikacyjne (feature flagi) | — | |
| 6 | Chaos engineering (Chaos Mesh) | — | |
| 7 | Grafana Cloud | — | |
| 8 | Opcjonalnie: Pyroscope, k6, Alloy, Sealed Secrets | — | |

## Struktura repozytorium

```
bootstrap/    root application: jedyny manifest Argo CD aplikowany ręcznie
apps/         definicje aplikacji Argo CD (wzorzec app-of-apps); każdy plik to jedna aplikacja
values/       wartości chartów Helm dla aplikacji z apps/
manifests/    zwykłe manifesty Kubernetes (aplikacje, dashboardy, eksperymenty chaosu)
docs/         instrukcje do kolejnych etapów
```

**Zasada:** po zainstalowaniu Argo CD i aplikacji `root` niczego nie instalujemy przez `helm` ani `kubectl apply`. Zmiana w klastrze to commit i push do `main`, a Argo CD robi resztę.
