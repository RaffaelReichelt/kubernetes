# Kubernetes-Lernpfad (auf dem GX10)

Dieses Verzeichnis dokumentiert den Einstieg in Kubernetes auf diesem Rechner
(dem ASUS GX10, der auch der Zielhost für die spätere Migration des
Swarm-Stacks ist — kein separates Testsystem). Ziel: den vorhandenen
Docker-Swarm-Stack (siehe `../../rollout` bzw. das jeweilige Service-Repo,
z.B. `~/Projekte/nanobot`) schrittweise nach Kubernetes zu übertragen und
dabei die Grundkonzepte zu verstehen.

## 1. Der lokale Cluster ("learn")

Wir nutzen [kind](https://kind.sigs.k8s.io/) (Kubernetes IN Docker) — jeder
Node ist ein Docker-Container, der einen kompletten Kubernetes-Node simuliert.
Das ist der Standard-Weg, um ohne Cloud-Kosten und ohne einen zweiten Rechner
einen echten Multi-Node-Cluster zum Lernen zu haben.

Aktueller Cluster:

- Name: `learn`
- 1 Control-Plane-Node + 2 Worker-Nodes
- Kubernetes-Version: v1.34.0 (Node-Image `kindest/node:v1.34.0`)
- kubectl-Context: `kind-learn`

### Cluster neu erstellen (falls nötig)

Die Konfiguration liegt in [kind-config.yaml](kind-config.yaml) und
entspricht dem, was aktuell läuft:

```bash
kind create cluster --name learn --config kind-config.yaml
```

Das legt drei Docker-Container an (`learn-control-plane`, `learn-worker`,
`learn-worker2`) und schreibt automatisch einen Kontext `kind-learn` in
`~/.kube/config`.

### Cluster stoppen / wieder starten

`kind` selbst hat keinen "stop"-Befehl — man stoppt/startet einfach die
zugrunde liegenden Docker-Container:

```bash
# stoppen (z.B. um Ressourcen zu sparen)
docker stop learn-control-plane learn-worker learn-worker2

# wieder starten
docker start learn-control-plane learn-worker learn-worker2
```

Nach einem Neustart können System-Pods kurz brauchen, bis sie wieder
`Running`/`Ready` sind (`kubectl get pods -A` zur Kontrolle,
siehe Troubleshooting unten).

### Cluster komplett löschen

```bash
kind delete cluster --name learn
```

(Nicht destruktiv ausführen, ohne vorher Rücksprache — löscht alle Workloads
im Cluster unwiderruflich.)

### Troubleshooting: "too many open files" nach Neustart

Nach einem `docker start` der Cluster-Container liefen `kube-proxy` und
`local-path-provisioner` in eine `CrashLoopBackOff` mit der Fehlermeldung
`too many open files`. Ursache: der Linux-Standardwert für inotify-Watches
(`fs.inotify.max_user_instances = 128`) ist für mehrere kind-Nodes zu niedrig
— [bekanntes kind-Problem](https://kind.sigs.k8s.io/docs/user/known-issues/#pod-errors-due-to-too-many-open-files).

Fix (einmalig, braucht sudo):

```bash
sudo sysctl -w fs.inotify.max_user_instances=512 fs.inotify.max_user_watches=524288
echo -e "fs.inotify.max_user_instances=512\nfs.inotify.max_user_watches=524288" | sudo tee /etc/sysctl.d/99-kind.conf
```

Die Datei in `/etc/sysctl.d/` sorgt dafür, dass der Wert auch einen Reboot
übersteht. Falls danach noch Pods in `CrashLoopBackOff`/`Error` hängen,
hilft `kubectl delete pod <name> -n <namespace>` — der zugehörige Controller
(Deployment/DaemonSet) erzeugt automatisch einen neuen, gesunden Pod.

### kubectl-Grundlagen, die wir benutzt haben

```bash
kubectl config get-contexts                          # welche Cluster kennt kubectl?
kubectl config use-context kind-learn                 # Cluster auswählen
kubectl config set-context --current --namespace=X    # Standard-Namespace setzen

kubectl get pods -A                                   # alle Pods, alle Namespaces
kubectl get <typ> [-l label=wert] [-o wide|yaml|json]  # Ressourcen auflisten
kubectl describe pod <name>                           # Details + Events (Fehlersuche!)
kubectl logs <pod>                                     # Container-Logs
kubectl exec <pod> -- <befehl>                         # Befehl im Container ausführen
kubectl apply -f <datei.yaml>                          # Soll-Zustand anwenden
kubectl delete pod <name>                              # Pod löschen (Controller ersetzt ihn ggf.)
kubectl rollout status deployment/<name>                # auf Rolling-Update warten
kubectl port-forward service/<name> <lokal>:<remote>    # lokalen Port zum Cluster tunneln
```

## 2. Lernpfad `01-basics/`

Alle Manifeste liegen in [01-basics/](01-basics/) und wurden in dieser
Reihenfolge angewendet. Jede Datei zeigt genau ein Konzept.

### Schritt 1 — [00-namespace.yaml](01-basics/00-namespace.yaml): Namespace

Ein Namespace ist ein logischer Unterbereich im Cluster. Trennt später z.B.
`opensearch`, `ollama`, `open-webui`, `comfyui`, `nanobot` voneinander, ohne
dass Namen kollidieren — ähnlich wie eigene Docker-Netzwerke pro Stack,
nur als Verwaltungseinheit statt Netzwerkgrenze.

### Schritt 2 — [01-deployment.yaml](01-basics/01-deployment.yaml): Pod vs. Deployment

Test: ein "nackter" Pod (`kubectl run naked-nginx ...`) wurde gelöscht und
war danach unwiderruflich weg — niemand kümmert sich um ihn.

Ein **Deployment** dagegen beschreibt einen Soll-Zustand ("2 Replicas von
Image X"). Ein Löschen eines seiner Pods führte dazu, dass sofort ein neuer
(mit anderem Namen) nachgezogen wurde. Dahinter steckt ein **ReplicaSet**,
das der Deployment-Controller verwaltet:

```
Deployment → ReplicaSet → Pods
```

Wichtige Beobachtung: Das Deployment selbst braucht eigene Labels
(`metadata.labels`), um von `kubectl get ... -l` gefunden zu werden — die
Labels im `spec.template` sind nur für die Pod-Zuordnung (Selector)
zuständig. Zwei unterschiedliche Dinge, die beide "Labels" heißen.

Später beim Ändern des Deployments (ConfigMap/Secret/PVC hinzugefügt) hat
`kubectl apply` automatisch ein **Rolling Update** ausgelöst: erst neuer
Pod hoch, dann erst alter runter — ohne Downtime.

### Schritt 3 — [02-service.yaml](01-basics/02-service.yaml): Service

Pods bekommen bei jedem (Neu-)Start eine neue interne IP. Ein **Service**
gibt eine stabile ClusterIP + einen DNS-Namen
(`<service>.<namespace>.svc.cluster.local`), der über Label-Selektoren
(`spec.selector`) automatisch auf die aktuell laufenden, passenden Pods
zeigt (sichtbar über `kubectl get endpoints <service>`).

Getestet:
- clusterintern per DNS-Name (temporärer curl-Pod)
- von außen (Host) per `kubectl port-forward service/nginx <port>:80`

Das ist das Kubernetes-Äquivalent zum Overlay-Network + interner
Service-Discovery in Swarm.

### Schritt 4 — [03-configmap.yaml](01-basics/03-configmap.yaml): ConfigMap

Konfiguration (hier: eine `index.html`) getrennt vom Container-Image im
Cluster abgelegt und per `volumeMounts`/`volumes` in den Container gemountet.
Kein neues Image nötig, um den Inhalt zu ändern. Entspricht Config-relevanten
Bind-Mounts im Swarm-Stack (z.B. `OPENSEARCH_USERS_FILE`,
`*_SERVE_JSON`-Dateien).

### Schritt 5 — [04-secret.yaml](01-basics/04-secret.yaml): Secret

Wie ConfigMap, aber für sensible Werte. Im Deployment referenziert über
`env[].valueFrom.secretKeyRef`, nicht als Literal — der Wert taucht nirgends
im Deployment-Manifest selbst auf. Entspricht `docker secret` im
Swarm-Stack (`ts_authkey`, `openai_api_key`, `opensearch_admin_password`, …).

**Wichtig:** In diesem Lern-Beispiel steht der Secret-Wert im Klartext in
`04-secret.yaml`, weil es nur ein Dummy-Wert ist. Für die echten Werte aus
`~/Projekte/rollout/secrets/` (Anthropic/OpenAI/Mistral-Keys, Tailscale-Authkey,
OpenSearch-Passwörter) **niemals** so im Git-Repo ablegen — dafür brauchen wir
bei der eigentlichen Migration eine Lösung wie Sealed Secrets oder SOPS,
die verschlüsselte Werte versioniert.

### Schritt 6 — [05-pvc.yaml](01-basics/05-pvc.yaml): PersistentVolumeClaim

Entspricht den Bind-Mounts für Daten (z.B. `OPENSEARCH_DATA_DIR`,
`COMFYUI_MODELS_DIR`, `OLLAMA_DATA_DIR`). Test: eine Datei in den gemounteten
Pfad geschrieben, den Pod gelöscht — der neu erzeugte Pod (andere IP, anderer
Name) hat die Datei weiterhin gesehen. Der Container ist flüchtig, das Volume
nicht.

Beobachtung dabei: kind nutzt standardmäßig die StorageClass `standard`
(`rancher.io/local-path`), die ein Volume an einen **bestimmten Node**
bindet (`VolumeBindingMode: WaitForFirstConsumer`, Modus `ReadWriteOnce`).
Nachdem die PVC eingebunden war, mussten beide Deployment-Replicas auf
denselben Node wandern, weil das Volume nur dort verfügbar ist — anders als
vorher, wo sie frei über beide Worker verteilt waren. Bei OpenSearch (Daten,
Single-Node-Discovery) wird das direkt relevant.

## 3. Mapping Swarm → Kubernetes

| Docker Swarm | Kubernetes | gezeigt in |
|---|---|---|
| `docker stack deploy -c compose.yml` | Deployment (+ ReplicaSet, Rolling Update) | `01-deployment.yaml` |
| Overlay-Network + interne Service-Discovery | Service (ClusterIP + DNS) | `02-service.yaml` |
| Bind-Mount für Konfigdateien | ConfigMap | `03-configmap.yaml` |
| `docker secret` | Secret | `04-secret.yaml` |
| Bind-Mount für persistente Daten | PersistentVolumeClaim | `05-pvc.yaml` |
| Tailscale-Sidecar pro Stack | weiterhin Sidecar-Container im Pod, **oder** zentral über Tailscale Kubernetes Operator | noch offen, siehe unten |

## 4. Ausblick: Migration der echten Services

Die eigentliche Migration der fünf Stacks (OpenSearch, Ollama, Open-WebUI,
ComfyUI, Nanobot) findet **nicht** hier statt, sondern jeweils im Repo des
Service selbst (z.B. `~/Projekte/nanobot`). Grund: Deployment-Manifeste
gehören zum Code des jeweiligen Service, nicht in dieses Lern-Repo.

Erster Kandidat: **Nanobot** (kein State, keine GPU — einfachster Einstieg).

Offener Punkt, bereits geklärt für die Umsetzung: Nanobot braucht weiterhin
Zugriff auf Ollama. Im Swarm-Stack läuft das nicht über das Overlay-Network,
sondern bereits über Tailscale (`OLLAMA_BASE_URL=https://ollama.example.ts.net:11434`)
— das ist Runtime-unabhängig und funktioniert unverändert unter Kubernetes,
sobald der Nanobot-Pod selbst Zugriff aufs Tailnet hat. Zwei Optionen:

1. **Tailscale-Sidecar im Pod** (1:1 wie im Swarm-Stack) — für den ersten
   Schritt vorgesehen, geringste Abweichung vom Bewährten.
2. **Tailscale Kubernetes Operator** (zentral, stellt Tailnet-Hosts als
   normale ClusterIP-Services bereit) — sinnvoll, sobald mehrere Services
   migriert sind und man nicht N Sidecars pflegen will.

Da dieser Cluster direkt auf dem GX10 läuft (kein separates Testsystem),
kann dafür der echte `ts_authkey` verwendet werden — anders als bei einem
Lern-Cluster auf einer separaten Maschine.

## 5. Hinweis für neue Claude-Code-Sitzungen

Der Speicher/Kontext einer Claude-Code-Sitzung ist an das jeweilige
Arbeitsverzeichnis gekoppelt. Eine Sitzung, die z.B. in
`~/Projekte/nanobot` gestartet wird, kennt dieses Dokument nicht
automatisch. Um nahtlos anzuknüpfen, dort einfach auf diese Datei
verweisen:

```
Schau dir /home/raffael/Projekte/kubernetes/learning/README.md an,
das ist der Kontext für die Kubernetes-Migration.
```
