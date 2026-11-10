# Etap 0: Przygotowanie

## Wymagania na stacji roboczej

| Narzędzie | Do czego | Sprawdzenie |
|---|---|---|
| `kubectl` | praca z klastrem | `kubectl --context homelab get nodes` |
| `helm` (v3.14+ lub v4) | jednorazowa instalacja Argo CD i przeglądanie chartów | `helm version` |
| `git` | repozytorium, czyli źródło prawdy dla Argo CD | `git --version` |
| `argocd` CLI (opcjonalnie) | podgląd i sterowanie Argo CD z terminala | `argocd version --client` |

Instalacja `argocd` CLI na Windows: `winget install argoproj.argocd`.

Wygodnie jest ustawić domyślny kontekst, żeby nie pisać `--context homelab` przy każdej komendzie:

```bash
kubectl config use-context homelab
```

Instrukcje dalej zakładają, że kontekst `homelab` jest aktywny.

## Stan klastra przed startem

Klaster jest wyczyszczony do świeżej instalacji K3s: działają tylko CoreDNS, Traefik, local-path-provisioner i metrics-server.

```bash
kubectl get ns
# default, kube-node-lease, kube-public, kube-system
kubectl get pods -A
```

## Węzły i rozmieszczenie podów

```bash
kubectl get nodes -L role
kubectl describe node worker-3 | grep -A1 Taints
```

- **worker-3** ma taint `dedicated=observability:NoSchedule` i etykietę `role=observability`. Taint odpycha pody, więc trafią tam tylko te, które go jawnie tolerują: Prometheus, Loki, Tempo, Grafana. Ich dane (PVC na `local-path`) lądują na dysku 162 GiB.
- **k3s, worker-1, worker-2** obsługują resztę: Argo CD, aplikację i Chaos Mesh.

Chaos wstrzykiwany w aplikację nie uderza w system, który ją obserwuje.

## Dostęp do usług: `*.homelab.local`

**Jak to działa:** K3s ma wbudowany Traefik (Ingress Controller), który nasłuchuje na porcie 80 każdego węzła (`kubectl -n kube-system get svc traefik`). Przeglądarka wysyła nagłówek `Host: argocd.homelab.local`, a Traefik na jego podstawie wybiera Ingress i serwis. Wystarczy więc, że nazwy rozwiązują się na adres dowolnego węzła.

Plik hosts nie obsługuje wildcardów, więc każdą nazwę trzeba dopisać osobno. Otwórz Notatnik **jako administrator**, edytuj `C:\Windows\System32\drivers\etc\hosts` i dopisz:

```
192.168.1.192  argocd.homelab.local podinfo.homelab.local grafana.homelab.local prometheus.homelab.local alertmanager.homelab.local shop.homelab.local chaos.homelab.local
```

Sprawdzenie:

```bash
ping -n 1 argocd.homelab.local              # 192.168.1.192
curl -s -o /dev/null -w "%{http_code}\n" http://argocd.homelab.local/
# 404 = Traefik odpowiada, ale nie ma jeszcze Ingressu dla tej nazwy (to prawidłowo)
```

> **Uwaga o `.local`:** domena `.local` jest zarezerwowana dla mDNS (Bonjour/Avahi). Na Windows wpisy w pliku hosts działają bez problemu. Na macOS i Linuksie zapytania o `.local` mogą najpierw iść do mDNS i trwać kilka sekund. Gdyby to przeszkadzało, alternatywą jest `homelab.home.arpa` (domena zarezerwowana dla sieci domowych, RFC 8375).

| Usługa | Adres | Etap |
|---|---|---|
| Argo CD | http://argocd.homelab.local | 1 |
| podinfo (aplikacja do ćwiczeń GitOps) | http://podinfo.homelab.local | 1 |
| Grafana | http://grafana.homelab.local | 2 |
| Prometheus | http://prometheus.homelab.local | 2 |
| Alertmanager | http://alertmanager.homelab.local | 2 |
| Sklep (OpenTelemetry Demo) | http://shop.homelab.local | 4 |
| Chaos Dashboard | http://chaos.homelab.local | 6 |

## Grafana Cloud

Konto jest już założone. Dane dostępowe przydadzą się dopiero w etapie 7. Nie zapisujemy ich w Git: trafią do sekretu Kubernetes tworzonego ręcznie.
