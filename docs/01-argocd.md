# Etap 1: GitOps z Argo CD

**Cel:** zainstalować Argo CD i zrozumieć pętlę GitOps: Git jest źródłem prawdy, a Argo CD stale porównuje klaster z repozytorium i doprowadza go do stanu z Gita.

**Plan etapu:**

| Krok | Co robisz | Czego się uczysz |
|---|---|---|
| 1.1 | Poznajesz chart Argo CD | Skąd się biorą wartości i jak je sprawdzać |
| 1.2 | Instalujesz Argo CD Helmem | Z jakich komponentów składa się Argo CD |
| 1.3 | Logujesz się do UI | Ingress, sekret startowy |
| 1.4 | Wypychasz repo na GitHuba | Argo CD czyta z Gita, nie z dysku |
| 1.5 | Tworzysz pierwszą aplikację | Zasób `Application` |
| 1.6 | Ćwiczenia: zmiana, drift, rollback, prune | Pętla GitOps w praktyce |
| 1.7 | Wdrażasz App of Apps | Jak jedna aplikacja zarządza innymi |
| 1.8 | Oddajesz Argo CD pod zarządzanie Argo CD | Self-management |

---

## 1.1 Poznaj chart Argo CD

Argo CD instalujemy oficjalnym chartem Helm. Zanim cokolwiek zainstalujesz, zobacz, co chart oferuje.

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm search repo argo/argo-cd --versions | head -5
```

**Wersja chartu a wersja aplikacji:** chart `10.17.1` instaluje Argo CD `v3.6.1`. W labie przypinamy wersję chartu, żeby instalacja była powtarzalna.

Wszystkie dostępne ustawienia (plik ma kilka tysięcy linii):

```bash
helm show values argo/argo-cd --version 10.17.1 > /tmp/argocd-defaults.yaml
```

Teraz przeczytaj [values/argocd.yaml](../values/argocd.yaml), czyli nasze nadpisania, i każde znajdź w pliku domyślnym:

| Ustawienie | Po co |
|---|---|
| `global.domain` | nazwa hosta, pod którą Argo CD będzie dostępne; z niej powstaje Ingress |
| `configs.params."server.insecure"` | serwer Argo CD domyślnie wymusza HTTPS; w labie bez certyfikatów wyłączamy to, a ruch idzie po HTTP przez Traefika |
| `configs.cm."timeout.reconciliation"` | co ile Argo CD sprawdza repo (domyślnie 180 s, u nas 60 s) |
| `server.ingress` | tworzy Ingress dla UI z klasą `traefik` |
| `dex.enabled: false` | Dex to serwer SSO; logujemy się lokalnym kontem `admin`, więc go nie potrzebujemy |

Możesz zobaczyć, jakie manifesty powstaną, jeszcze bez instalacji:

```bash
helm template argocd argo/argo-cd --version 10.17.1 -n argocd -f values/argocd.yaml | grep '^kind:' | sort | uniq -c
```

Zwróć uwagę na `CustomResourceDefinition`: `Application`, `AppProject` i `ApplicationSet` to nowe typy zasobów, które Argo CD dodaje do Kubernetesa.

## 1.2 Zainstaluj Argo CD

```bash
helm install argocd argo/argo-cd \
  --version 10.17.1 \
  --namespace argocd --create-namespace \
  -f values/argocd.yaml
```

Obserwuj start podów (Ctrl+C, gdy wszystkie są `Running`):

```bash
kubectl -n argocd get pods -w
```

**Komponenty Argo CD:**

| Pod | Rola |
|---|---|
| `argocd-server` | API i UI webowe, z nim rozmawia przeglądarka i CLI |
| `argocd-repo-server` | klonuje repo Git i renderuje manifesty (Helm, Kustomize, czysty YAML) |
| `argocd-application-controller` | serce GitOps: porównuje stan pożądany (z repo-servera) ze stanem klastra i synchronizuje |
| `argocd-applicationset-controller` | generuje wiele `Application` z szablonu (nie używamy go jeszcze) |
| `argocd-redis` | cache (wyrenderowane manifesty, stan) |
| `argocd-notifications-controller` | powiadomienia (Slack, e-mail…) o zdarzeniach |

Zwróć uwagę, na których węzłach wylądowały pody: **żaden nie trafi na worker-3** (taint).

```bash
kubectl -n argocd get pods -o wide
```

## 1.3 Zaloguj się do UI

```bash
kubectl -n argocd get ingress
```

Otwórz http://argocd.homelab.local. Login to `admin`, a hasło startowe generuje się przy instalacji i leży w sekrecie:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
```

Sekrety w Kubernetesie są tylko zakodowane w base64, nie zaszyfrowane, stąd `base64 -d`.

Po zalogowaniu zmień hasło (*User Info → Update Password*) i usuń sekret startowy, bo nie jest już potrzebny:

```bash
kubectl -n argocd delete secret argocd-initial-admin-secret
```

## 1.4 Wypchnij repozytorium

Argo CD nie widzi plików na Twoim dysku. Czyta z `https://github.com/arturmachura/homlab-edu.git`, gałąź `main`. Repo jest publiczne, więc Argo CD nie potrzebuje do niego danych logowania.

```bash
git add .
git status        # przejrzyj, co trafi do repo: żadnych sekretów!
git commit -m "Etap 0-1: Argo CD, podinfo"
git push -u origin main
```

## 1.5 Pierwsza aplikacja: podinfo

Przeczytaj [apps/podinfo.yaml](../apps/podinfo.yaml). `Application` odpowiada na trzy pytania:

- **skąd** (`source`): repo, gałąź (`targetRevision`) i ścieżka (`manifests/podinfo`),
- **dokąd** (`destination`): klaster i namespace,
- **jak** (`syncPolicy`): automatycznie, z `prune` i `selfHeal`.

Utwórz ją:

```bash
kubectl apply -f apps/podinfo.yaml
kubectl -n argocd get applications -w
```

W UI kliknij `podinfo` i obejrzyj drzewo zasobów: Application → Deployment → ReplicaSet → Pod, Service, Ingress.

Otwórz http://podinfo.homelab.local.

**Dwa statusy aplikacji:**
- **Sync** (`Synced` / `OutOfSync`): czy klaster zgadza się z Gitem.
- **Health** (`Healthy` / `Progressing` / `Degraded` / `Missing`): czy zasoby faktycznie działają.

To dwie niezależne rzeczy, co pokaże ćwiczenie 3.

## 1.6 Ćwiczenia

### Ćwiczenie 1: Zmiana przez Git

1. W [manifests/podinfo/deployment.yaml](../manifests/podinfo/deployment.yaml) zmień `PODINFO_UI_MESSAGE` i `PODINFO_UI_COLOR` (np. `#2e7d32`).
2. Zrób commit i push.
3. W UI obserwuj `podinfo`: `OutOfSync`, potem sync i rolling update podów. Trwa to do 60 s; przycisk *Refresh* wymusza natychmiastowe sprawdzenie repo.
4. Odśwież http://podinfo.homelab.local.

**Wniosek:** wdrożenie to commit. Nikt nie wykonuje `kubectl apply`, a historia wdrożeń jest historią Gita.

### Ćwiczenie 2: Drift i self-heal

Zmień coś ręcznie, z pominięciem Gita:

```bash
kubectl -n podinfo scale deployment podinfo --replicas=5
kubectl -n podinfo get pods -w
```

Argo CD wykrywa rozbieżność i w ciągu kilku sekund wraca do 2 replik. Zobacz *History and Events* w UI.

Teraz wyłącz `selfHeal` (w UI: *App Details → Sync Policy → Disable Self Heal*) i powtórz `scale`. Aplikacja zostaje `OutOfSync`, a *App Diff* pokazuje różnicę. Kliknij *Sync*, żeby wrócić do stanu z Gita. Na koniec **włącz selfHeal z powrotem**.

> Zauważ, że zmiana polityki w UI to też drift: plik `apps/podinfo.yaml` w Gicie nadal ma `selfHeal: true`. Gdy w kroku 1.7 aplikacją zacznie zarządzać `root`, takie ręczne zmiany też będą cofane.

**Wniosek:** stan klastra wynika z Gita. Ręczne poprawki „na szybko” są wykrywane i cofane.

### Ćwiczenie 3: Zepsute wdrożenie i rollback

1. Zmień obraz na nieistniejący: `ghcr.io/stefanprodan/podinfo:9.9.9`, commit i push.
2. Obserwuj: aplikacja jest `Synced`, ale `Progressing`, a potem `Degraded`. Nowy pod ma `ImagePullBackOff`, a **stare pody nadal obsługują ruch**, bo Deployment robi rolling update i nie usuwa starych podów, dopóki nowe nie są gotowe (`readinessProbe`).
   ```bash
   kubectl -n podinfo get pods
   kubectl -n podinfo describe pod <nowy-pod> | tail -5
   ```
3. Rollback po GitOpsowemu:
   ```bash
   git revert HEAD
   git push
   ```

**Wniosek:** *Synced* oznacza „klaster zgadza się z Gitem”, a nie „działa”. O tym, czy działa, mówi *Health*. Rollback to `git revert`. Przycisk *History and Rollback* w UI jest zablokowany przy automatycznej synchronizacji, bo Argo i tak wróciłby do stanu z Gita.

### Ćwiczenie 4: Prune

1. Usuń plik `manifests/podinfo/ingress.yaml`, commit i push.
2. Ingress znika z klastra (`kubectl -n podinfo get ingress`), a strona przestaje działać.
3. Przywróć plik przez `git revert HEAD` i push.

**Wniosek:** `prune: true` usuwa z klastra to, czego nie ma w Gicie. Bez prune zostawałyby „sieroty”, o których nikt nie wie.

### Ćwiczenie 5 (opcjonalne): Argo CD CLI

```bash
argocd login argocd.homelab.local --username admin --grpc-web --plaintext
argocd app list
argocd app get podinfo
argocd app diff podinfo
argocd app history podinfo
```

## 1.7 App of Apps

Do tej pory `Application` tworzyłeś przez `kubectl apply`, co jest ręczne i niezgodne z GitOps. Rozwiązaniem jest aplikacja, która wdraża inne aplikacje.

Przeczytaj [bootstrap/root-app.yaml](../bootstrap/root-app.yaml): `source.path: apps` oznacza, że `root` wdraża **wszystkie pliki z katalogu `apps/`**, czyli obiekty `Application`.

```bash
kubectl apply -f bootstrap/root-app.yaml
kubectl -n argocd get applications
```

Co się stało:
- `root` przejął istniejący obiekt `podinfo`, bo to ten sam plik z `apps/`, więc nic się nie zmieniło w działaniu.
- `root` utworzył nową aplikację `argocd` z [apps/argocd.yaml](../apps/argocd.yaml) (opis w kroku 1.8).

W UI otwórz `root`: drzewo pokazuje aplikacje jako jego zasoby.

**Od teraz to jedyny `kubectl apply` w labie.** Kolejne etapy to nowe pliki w `apps/` i commit.

Sprawdź to: usuń `apps/podinfo.yaml`, zrób commit i push. Dzięki `prune` w `root` i finalizerowi w `podinfo` znika aplikacja **i** wszystkie jej zasoby. Pusty namespace `podinfo` zostaje, bo Argo CD go utworzył (`CreateNamespace=true`), ale nim nie zarządza. Przywróć przez `git revert HEAD` i push.

## 1.8 Argo CD zarządza samym sobą

Argo CD zainstalowałeś Helmem, ale od kroku 1.7 aplikacja `argocd` renderuje **ten sam chart z tymi samymi values** i trzyma go w zgodzie z Gitem. To nie powoduje reinstalacji, bo Argo CD widzi, że zasoby już wyglądają tak, jak powinny.

Konsekwencje:
- aktualizacja Argo CD to zmiana `targetRevision` w `apps/argocd.yaml` i commit,
- zmiana konfiguracji to edycja `values/argocd.yaml` i commit,
- `helm upgrade` już nie jest potrzebny; Helm był tylko „rozrusznikiem”.

Ćwiczenie: w `values/argocd.yaml` zmień `timeout.reconciliation` na `90s`, zrób commit i push. Sprawdź, że zmiana dotarła:

```bash
kubectl -n argocd get cm argocd-cm -o jsonpath='{.data.timeout\.reconciliation}'; echo
```

Przywróć `60s`.

> Dlaczego `apps/argocd.yaml` ma `prune: false`? Błąd w values mógłby sprawić, że Argo CD usunąłby własne komponenty i nie miałby już czym się naprawić.

## Rozwiązywanie problemów

| Objaw | Przyczyna i rozwiązanie |
|---|---|
| `ComparisonError: ... remote repository is empty` | Nic nie zostało wypchnięte na GitHuba (pominięty krok 1.4). Argo CD nie ma skąd wziąć manifestów, więc nie tworzy zasobów; strona aplikacji zwraca 404. Zrób commit i push, potem *Refresh*. |
| `ComparisonError: repository not found` | Błędny `repoURL` albo prywatne repo. Dla prywatnego repo dodaj token w *Settings → Repositories*. |
| Sync `Unknown`, Health `Healthy` | Argo CD nie mógł porównać stanu (błąd repo lub manifestów). „Healthy” niczego tu nie dowodzi, bo aplikacja bez zasobów nie ma czego być chorą. Sprawdź komunikat: `kubectl -n argocd get application <app> -o jsonpath='{.status.conditions[*].message}'`. |
| Aplikacja długo `OutOfSync` | *Refresh* w UI albo `argocd app get <app> --refresh`. |
| http://argocd.homelab.local nie odpowiada | Sprawdź wpis w hosts (`ping argocd.homelab.local`) oraz `kubectl -n argocd get ingress`. |
| `404 page not found` | Odpowiada Traefik, ale nie ma Ingressu dla tej nazwy: literówka w hoście lub aplikacja jeszcze się nie zsynchronizowała. |

## Podsumowanie

- Git jest jedynym źródłem prawdy; wdrożenie = commit, rollback = `git revert`.
- Argo CD wykrywa drift i go cofa (`selfHeal`) oraz sprząta usunięte zasoby (`prune`).
- *Sync* ≠ *Health*.
- App of Apps: jeden ręczny `kubectl apply`, potem wszystko z katalogu `apps/`.

**Następny etap:** Prometheus i Grafana (`kube-prometheus-stack`) na worker-3. Wdrożysz je, dodając jeden plik do `apps/`.
