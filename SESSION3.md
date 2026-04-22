# Session 3: Helm Charts & Argo CD Sync-Mechanismen

## Ziel

Helm Charts mit Argo CD verwalten, eigene Values aus Git nutzen und fortgeschrittene Sync-Features (Waves, Hooks) hands-on erleben.

## Was ist neu?

- **Helm-Integration**: Externes Chart (Grafana) mit lokalen Values kombinieren
- **Sync Waves**: Deployment-Reihenfolge deklarativ steuern (ConfigMap → Secret → App → Service)
- **Sync Hooks**: Pre/Post-Deployment-Jobs (Backups, Tests)

## Warum Grafana?

Grafana ist das ideale Dojo-Beispiel:
- Kein PostgreSQL oder sonstige DB nötig (embedded SQLite)
- Leichtgewichtig (~200 MB RAM)
- Web-UI sofort sichtbar und interaktiv
- Offizielles, stabiles Helm Chart
- Viele sinnvolle Values zum Demonstrieren (Datasources, Dashboards, Auth)

## Voraussetzung

```powershell
minikube start --driver=podman --memory=4096 --cpus=2
```

## Repo-Struktur (erweitert)

```
argo-dojo/
├── app/
│   └── deployment.yaml          # Session 2: Demo-App
├── grafana/
│   ├── values.yaml              # Custom Helm Values
│   └── application.yaml         # Argo CD Application (Multi-Source)
├── sync-waves-demo/
│   ├── wave-0-configmap.yaml    # ConfigMap zuerst
│   ├── wave-1-secret.yaml       # Secret danach
│   ├── wave-2-deployment.yaml   # Deployment (nutzt ConfigMap + Secret)
│   ├── wave-3-service.yaml      # Service zuletzt
│   └── application.yaml         # Argo App für Waves-Demo
└── hooks-demo/
    ├── presync-backup.yaml      # PreSync Hook (Job)
    ├── deployment.yaml          # Normale App
    ├── postsync-test.yaml       # PostSync Hook (Job)
    └── application.yaml         # Argo App für Hooks-Demo
```

## Teil 1: Grafana mit Helm deployen

### Schritt 1: Helm Repository in Argo CD registrieren

```powershell
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: grafana-helm-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: helm
  name: grafana
  url: https://grafana.github.io/helm-charts
EOF
```

> **Zscaler/Proxy-Problem?** `insecure: true` in `stringData` ergänzen (siehe ZSCALER-FIX.md).

Verifizieren in der UI: Settings → Repositories → sollte "grafana" Repo zeigen.

### Schritt 2: Custom Values verstehen

Die Datei `grafana/values.yaml` enthält optimierte Settings für lokales Setup:

**Key-Highlights:**
- `replicas: 1` (für Dojo ausreichend)
- `persistence.enabled: false` (kein PVC nötig)
- `adminUser/adminPassword` direkt gesetzt (NUR für Demo!)
- `service.type: NodePort` (für lokalen Zugriff)
- Datasource und Dashboard-Provider vorkonfiguriert

### Schritt 3: Application deployen

**Voraussetzung: Argo CD 2.6+ für Multi-Source Support**

Die `grafana/application.yaml` nutzt Multi-Source:
- **Source 1:** Helm Chart von grafana.github.io
- **Source 2:** Custom Values aus diesem Git Repo

```powershell
# Application deployen
kubectl apply -f grafana/application.yaml
```

**Falls Argo CD < 2.6: Inline Values nutzen**

```powershell
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: grafana
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://grafana.github.io/helm-charts
    chart: grafana
    targetRevision: 8.5.1
    helm:
      releaseName: grafana
      values: |
        replicas: 1
        service:
          type: NodePort
          port: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
        persistence:
          enabled: false
        adminUser: admin
        adminPassword: dojo-admin
        env:
          GF_AUTH_ANONYMOUS_ENABLED: "true"
          GF_AUTH_ANONYMOUS_ORG_ROLE: "Viewer"
  destination:
    server: https://kubernetes.default.svc
    namespace: grafana
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF
```

### Schritt 4: Deployment beobachten

```powershell
# Application Status
kubectl get application -n argocd grafana

# Pods (startet in unter 1 Minute!)
kubectl get pods -n grafana -w
```

### Schritt 5: Grafana UI öffnen

```powershell
# Port-Forward
kubectl port-forward -n grafana svc/grafana 3000:80
```

Browser: `http://localhost:3000`
- Login: `admin` / `dojo-admin`

## Teil 2: Sync Waves demonstrieren

Sync Waves sorgen für die richtige Reihenfolge beim Deployment.

### Konzept

Ressourcen mit niedrigeren Wave-Nummern werden zuerst deployt:

```
Wave 0: ConfigMap, Namespace     (Basis-Konfiguration)
  ↓
Wave 1: Secret                   (Credentials)
  ↓
Wave 2: Deployment               (App nutzt ConfigMap + Secret)
  ↓
Wave 3: Service, Ingress         (Netzwerk-Layer)
```

### Live-Demo deployen

```powershell
# Demo-App mit Waves deployen
kubectl apply -f sync-waves-demo/application.yaml

# In Argo CD UI anschauen:
# - App "sync-waves-demo" öffnen
# - Sync Button drücken
# - Beobachten: Ressourcen erscheinen in Wave-Reihenfolge
```

### Waves in Argo CD UI sehen

Nach dem Sync in der UI:
1. App `sync-waves-demo` öffnen
2. Auf "App Details" klicken
3. Ressourcen sind nach Sync Wave gruppiert
4. Timestamps zeigen zeitliche Abfolge

### Experiment: Wave ändern

```yaml
# In sync-waves-demo/wave-2-deployment.yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"  # Vorher war "2"
```

Committen, pushen → Deployment kommt jetzt VOR dem Secret → Pod startet mit Fehler (Secret fehlt noch)!

**Zurücksetzen auf `"2"` und erneut committen** → Argo CD heilt automatisch.

## Teil 3: Sync Hooks für Pre/Post-Deployment

Hooks sind Jobs, die VOR, WÄHREND oder NACH dem Sync laufen.

### Hook-Typen

- `PreSync`: Läuft vor allen anderen Ressourcen (z.B. DB-Backup, Migrations)
- `Sync`: Läuft während des Sync (selten genutzt)
- `PostSync`: Läuft nach erfolgreichem Sync (z.B. Smoke-Tests, Cache-Warmup)
- `SyncFail`: Läuft bei Sync-Fehler (z.B. Rollback-Trigger, Alerting)

### Demo deployen

```powershell
# Hooks-Demo via Argo CD Application
kubectl apply -f hooks-demo/application.yaml
```

### Hook-Execution beobachten

```powershell
# In Argo CD UI: App "hooks-demo" → Sync Button

# Hook-Jobs anschauen
kubectl get jobs -n default

# Logs vom PreSync Hook
kubectl logs job/presync-backup -n default

# Logs vom PostSync Hook
kubectl logs job/postsync-smoketest -n default
```

In der Argo CD UI:
- App öffnen → Sync Button
- Während des Sync erscheinen Hook-Jobs
- Nach Erfolg: PostSync-Job läuft
- Grüner Haken bei allen Ressourcen = fertig

### Hook Delete Policies

```yaml
annotations:
  argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
  # Alternativen:
  # - HookSucceeded: Löschen nach erfolgreichem Job
  # - HookFailed: Löschen bei Fehler
  # - BeforeHookCreation: Löschen vor erneutem Run
```

## Teil 4: GitOps-Flow mit Helm Values

### Demo: Grafana Replicas ändern

```yaml
# In grafana/values.yaml
replicas: 2  # vorher: 1
```

```powershell
git commit -am "Scale Grafana to 2 replicas"
git push

# Argo CD erkennt Change (nach max. 3 Min)
kubectl get pods -n grafana -w
# → Zweiter Pod startet automatisch
```

### Demo: Drift Detection

```powershell
# Manuell im Cluster ändern
kubectl scale deployment grafana -n grafana --replicas=3

# Argo CD zeigt "OutOfSync"
kubectl get application -n argocd grafana

# Self-Heal aktiviert → nach ~1 Min zurück auf 2 Replicas
kubectl get pods -n grafana -w
```

## Cheat Sheet: Wichtige Befehle

```powershell
# Argo CD Apps auflisten
kubectl get applications -n argocd

# App-Status details
kubectl describe application grafana -n argocd

# App löschen (inkl. Ressourcen im Cluster)
kubectl delete application grafana -n argocd

# Helm Repos in Argo CD auflisten
kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=repository

# Argo CD Password neu holen
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```

## Troubleshooting

| Problem | Lösung |
|---------|--------|
| **Helm Repo nicht erreichbar (Zscaler/Proxy)** | `insecure: true` in Repository Secret setzen (siehe ZSCALER-FIX.md) |
| **x509: certificate signed by unknown authority** | Corporate Proxy bricht TLS auf → `insecure: true` oder Zertifikat injizieren |
| **DNS Timeout in Minikube (WSL)** | CoreDNS auf Google DNS (8.8.8.8) umstellen → siehe DNS-Fix in [SESSION1.md](SESSION1.md#dns-problem-in-minikube-wsl--podman-driver) |
| Grafana Pod bleibt in `Pending` | Zu wenig RAM → `minikube start --memory=4096` |
| Helm Chart nicht gefunden | Repo registriert? `kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=repository` |
| Sync Waves ignoriert | Annotations müssen Strings sein: `"0"` nicht `0` |
| Hook-Job bleibt in `Error` | Logs prüfen: `kubectl logs job/<job-name>`, dann Hook löschen und neu deployen |
| Values aus Git werden nicht geladen | Multi-Source Feature prüfen (Argo CD >= 2.6) oder Inline-Values nutzen |

## Best Practices

1. **Chart-Versionen pinnen**: `targetRevision: 8.5.1` statt `latest`
2. **Waves sinnvoll nummerieren**: 0, 10, 20, 30 statt 0, 1, 2, 3 (Platz für Zwischenschritte)
3. **Secrets nie in values.yaml**: Sealed Secrets oder External Secrets nutzen (Session 4)
4. **Resource Limits setzen**: Cluster-Überlastung vermeiden
5. **Hook Delete Policy**: `BeforeHookCreation` für wiederholbare Hooks

## Key Learnings

✅ **Helm + Argo CD = Best of Both Worlds**
- Vendor-Charts nutzen (Grafana, Prometheus, ...)
- Eigene Konfiguration in Git
- Chart vom Vendor, Values von uns

✅ **Sync Waves = Deklarative Dependencies**
- Keine `kubectl wait` oder Init-Container-Hacks
- Argo CD kümmert sich um Reihenfolge

✅ **Hooks = CI/CD-Integration**
- Pre-Deploy: Backups, Schema-Migrations
- Post-Deploy: Tests, Cache-Warmup

✅ **GitOps-Flow auch für komplexe Apps**
- Jede Änderung via Git Commit
- Self-Heal korrigiert Cluster-Drift
- Audit-Trail durch Git History

## Ausblick Session 4

- **Private Repos**: GitLab mit Access Token und Self-Signed Cert anbinden
- **Self-Signed Certificates**: Custom CA in Argo CD konfigurieren
- **Token-Auth**: GitLab Project Access Tokens sicher einbinden
- **Enterprise-Hürden**: TLS/Auth-Troubleshooting meistern

👉 **[Weiter zu Session 4 →](SESSION4.md)**
