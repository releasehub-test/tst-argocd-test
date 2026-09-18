# Środowisko testowe ArgoCD (scenariusze `DEPLOY-CICD-ARGOCD-*`)

Postawione 2026-09-18 na **wbudowanym Kubernetesie Docker Desktop** (tryb `kind`, jeden węzeł
`desktop-control-plane`, k8s 1.36.1) — nie na osobnym k3d, bo Docker Desktop ma własny klaster
i nie trzeba instalować niczego dodatkowego.

| Element | Wartość |
|---|---|
| ArgoCD | v3.5.3, namespace `argocd`, serwer w trybie `server.insecure=true` (HTTP) |
| Wystawienie w klastrze | NodePort **30080** (http) / 30443, węzeł w sieci docker `kind` |
| Tunel | kontener `argocd-cloudflared` → `http://desktop-control-plane:30080` |
| Adres publiczny | **losowy**, `https://<słowa>.trycloudflare.com` — odczytaj z logów tunelu |
| Konto dla pluginu | `releasehub` (`apiKey`), token **bezterminowy**, auth typu Bearer |
| Konto admina | `admin`, hasło w sekrecie `argocd-initial-admin-secret` |

Licencja: ArgoCD jest open source (Apache 2.0), Kubernetes w Docker Desktop mieści się w licencji
Dockera dla tego stanowiska — środowisko nie generuje kosztu.

## Uwaga: kubectl z hosta NIE działa

Avast Web Shield przechwytuje połączenie do kube-apiservera na `127.0.0.1` i podstawia własny
certyfikat (`issuer=Avast Web/Mail Shield Untrusted Root`), więc kubectl kończy na
`x509: certificate signed by unknown authority`. **Nie obchodź tego wyłączaniem weryfikacji TLS.**
Wszystkie operacje na klastrze rób wewnątrz węzła:

```bash
docker exec desktop-control-plane kubectl --kubeconfig /etc/kubernetes/admin.conf get pods -A
```

W Git Bash ustaw `MSYS_NO_PATHCONV=1`, inaczej `/etc/kubernetes/...` zostanie przepisane na ścieżkę
Windows. `docker.exe` jest zainstalowany per-user w
`C:\Users\jb\AppData\Local\Programs\DockerDesktop\resources\bin`, nie w `Program Files`.

Węzeł ma `curl` i wychodzi do internetu **bez** przechwytywania przez Avast, więc manifesty pobiera
się tam, a nie na hoście.

## Uruchomienie od zera

1. **Kubernetes w Docker Desktop.** `docker desktop kubernetes` umie tylko `status` / `images` /
   `reset-cluster` — **nie włącza** klastra. Włącz go w GUI (Settings → Kubernetes) albo wpisem
   `"KubernetesEnabled": true` w `%APPDATA%\Docker\settings-store.json` przy **całkowicie
   zatrzymanym** Docker Desktop (przy działającym procesie plik zostaje nadpisany przy wyjściu i
   traci też inne klucze). Klucz `KubernetesInitialInstallPerformed` jest ignorowany.

2. **ArgoCD** (wewnątrz węzła). `kubectl apply` bez `--server-side` wywala się na CRD
   `applicationsets.argoproj.io` — `metadata.annotations: Too long: may not be more than 262144 bytes`:

   ```bash
   docker exec desktop-control-plane sh -c '
     K="kubectl --kubeconfig /etc/kubernetes/admin.conf"
     $K create namespace argocd
     curl -sSL https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml -o /tmp/argocd.yaml
     $K apply -n argocd --server-side --force-conflicts -f /tmp/argocd.yaml
     $K -n argocd wait --for=condition=available --timeout=420s deployment --all
   '
   ```

3. **Tryb insecure + NodePort** — serwer oddaje HTTP, dzięki czemu tunel nie musi ufać
   samopodpisanemu certyfikatowi (to zdejmuje EDGE-5 ze scenariusza `ARGOCD-001`):

   ```bash
   $K -n argocd patch configmap argocd-cmd-params-cm --type merge -p '{"data":{"server.insecure":"true"}}'
   $K -n argocd patch svc argocd-server -p '{"spec":{"type":"NodePort","ports":[
       {"name":"http","port":80,"targetPort":8080,"nodePort":30080},
       {"name":"https","port":443,"targetPort":8080,"nodePort":30443}]}}'
   $K -n argocd rollout restart deployment argocd-server
   ```

4. **Konto dla pluginu.** JWT z `/api/v1/session` żyje 24 h i nie nadaje się do testów — dlatego
   osobne konto z bezterminowym tokenem:

   ```bash
   $K -n argocd patch configmap argocd-cm --type merge -p '{"data":{"accounts.releasehub":"apiKey"}}'
   # argocd-rbac-cm → policy.csv: applications get/sync/action dla role:releasehub, g, releasehub, role:releasehub
   $K -n argocd rollout restart deployment argocd-server
   # następnie: POST /api/v1/session (admin) → JWT, potem POST /api/v1/account/releasehub/token
   ```

5. **Tunel:** `docker compose up -d` w tym katalogu, potem `docker logs argocd-cloudflared`
   i wyłuskaj adres `https://*.trycloudflare.com`.

## Podpięcie pod Release Hub

ArgoCD jest presetem **prywatnego hosta** (`PRIVATE_HOST_PRESET_KEYS` w
`src/domain/installationconfig/PrivateHostPresetPolicy.ts`), więc Forge nie wykona fetcha wprost:

1. Release Hub **Integrations** → **Endpointy HTTP** → dodaj adres tunelu.
2. W pluginie: Ustawienia → Konfiguracja instalacji → szablon z presetem **ArgoCD**, endpoint proxy
   = powyższy wpis, auth **Bearer** = token konta `releasehub`.
3. Adres tunelu zmienia się po każdym restarcie kontenera — wtedy trzeba go podmienić w punkcie 1.

## Repozytorium z manifestami

Ten katalog **jest** repozytorium `https://github.com/releasehub-test/tst-argocd-test` (gałąź
`main`) — tym, które obserwuje ArgoCD. `k8s/` zawiera wdrażane manifesty (Deployment + Service,
nginx, namespace `default`).

**To samo repo musi zostać dodane w pluginie jako repozytorium modułu testowego.** Plugin wysyła
w polu `revision` SHA commita repo modułu, więc jeśli Application patrzy gdzie indziej, sync
zwróci błąd o nieznanej rewizji — a wygląda to na błąd pluginu, nie konfiguracji.

Żeby kolejne SHA dawały widocznie różny stan w klastrze, zmieniaj przy każdym commicie adnotację
`releasehub.test/revision` (albo tag obrazu) — inaczej dwa różne commity produkują identyczny
obiekt i rollback wygląda jak brak zmian.

Poświadczenia **nie leżą w tym repo** — hasło admina i token konta `releasehub` są w
`..\argocd-credentials.txt` (poziom wyżej), a `.gitignore` blokuje `credentials.txt`.

Zastosowanie Application (po wypchnięciu pierwszego commita na GitHub):

```bash
docker cp argocd-application.yaml desktop-control-plane:/tmp/app.yaml
docker exec desktop-control-plane kubectl --kubeconfig /etc/kubernetes/admin.conf apply -f /tmp/app.yaml
```

## Sprawdzenie obwodu

```bash
# z wnętrza węzła, przez publiczny adres tunelu:
curl -sS https://<adres>.trycloudflare.com/api/version
curl -sS -H "Authorization: Bearer <token>" https://<adres>.trycloudflare.com/api/v1/applications

# ręczny sync na konkretny SHA (to samo żądanie, które wysyła plugin):
curl -sS -X POST -H "Authorization: Bearer <token>" -H "Content-Type: application/json" \
  -d '{"revision":"<sha>","prune":false,"dryRun":false,"strategy":{"hook":{}}}' \
  https://<adres>.trycloudflare.com/api/v1/applications/module-test-deployment/sync
```

## Znany rozjazd w kodzie

Preset deklaruje `statusCheck.statusJsonPath: $.status.sync.status`, ale poller
(`src/application/installation/PollStatusExtractor.ts`) czyta `$.status.operationState.phase`.
Scenariusz `DEPLOY-CICD-ARGOCD-002` opisuje mapowanie sync/health, którego kod nie realizuje —
do rozstrzygnięcia przed uznaniem wyniku tego scenariusza za wiążący.
