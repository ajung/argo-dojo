# Session 3: Helm Charts & Argo CD Sync-Mechanismen

## Ziel

Helm Charts mit Argo CD verwalten, eigene Values aus Git nutzen und fortgeschrittene Sync-Features (Waves, Hooks) hands-on erleben.

## Was ist neu?

- **Helm-Integration**: Externe Charts (SonarQube) mit lokalen Values kombinieren
- **Sync Waves**: Deployment-Reihenfolge deklarativ steuern (ConfigMap → Secret → App → Service)
- **Sync Hooks**: Pre/Post-Deployment-Jobs (Backups, Tests)
- **Multi-Component Apps**: Realistische Deployment-Szenarien

## Warum SonarQube?

SonarQube ist ein realistisches Beispiel mit mehreren Komponenten:
- Eigene Datenbank (PostgreSQL)
- Stateful-Optionen (für spätere Sessions)
- Web-UI für sofortige Verifikation
- Offizielles Helm Chart von SonarSource

## Voraussetzung: Mehr Ressourcen

SonarQube braucht mehr RAM als die Demo-App aus Session 2:

```powershell
minikube delete
minikube start --driver=podman --memory=6144 --cpus=4
```

## Repo-Struktur (erweitert)

```
argo-dojo/
├── app/
│   └── deployment.yaml          # Session 2: Demo-App
├── sonarqube/
│   ├── namespace.yaml           # Wave 0: Namespace
│   ├── values.yaml              # Custom Helm Values
│   └── application.yaml         # Argo CD Application
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

## Teil 1: SonarQube mit Helm deployen

### Schritt 1: Helm Repository in Argo CD registrieren

```powershell
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: sonarqube-helm-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: helm
  name: sonarqube
  url: https://SonarSource.github.io/helm-chart-sonarqube
EOF
```

Verifizieren in der UI: Settings → Repositories → sollte "sonarqube" Repo zeigen

### Schritt 2: Custom Values verstehen

Die Datei `sonarqube/values.yaml` enthält optimierte Settings für lokales Setup:

**Key-Highlights:**
- `replicaCount: 1` (für Dojo ausreichend)
- `persistence.enabled: false` (ephemeral OK für Demo)
- `postgresql.enabled: true` (embedded DB)
- `service.type: NodePort` (für lokalen Zugriff)
- Reduzierte Resource Requests/Limits

### Schritt 3: Application deployen

```powershell
# Alle Dateien sind bereits im Repo
git add sonarqube/ sync-waves-demo/ hooks-demo/
git commit -m "Add Session 3: Helm Charts, Sync Waves & Hooks"
git push

# SonarQube Application deployen
kubectl apply -f sonarqube/application.yaml
```

**Alternative: Inline Values (falls Argo CD < 2.6)**

```powershell
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sonarqube
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://SonarSource.github.io/helm-chart-sonarqube
    chart: sonarqube
    targetRevision: 10.6.0+3033
    helm:
      releaseName: sonarqube
      values: |
        replicaCount: 1
        resources:
          requests:
            cpu: 400m
            memory: 1.5Gi
        persistence:
          enabled: false
        postgresql:
          enabled: true
          persistence:
            enabled: false
        service:
          type: NodePort
        account:
          adminPassword: admin
  destination:
    server: https://kubernetes.default.svc
    namespace: sonarqube
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      CreateNamespace=true
EOF
```

### Schritt 4: Deployment beobachten

```powershell
# Application Status
kubectl get application -n argocd sonarqube

# Pods (dauert 2-3 Minuten bis Running)
kubectl get pods -n sonarqube -w

# Logs bei Problemen
kubectl logs -n sonarqube -l app=sonarqube --tail=50 -f
```

### Schritt 5: SonarQube UI öffnen

```powershell
# NodePort-URL holen
minikube service sonarqube-sonarqube -n sonarqube --url

# Oder Port-Forward
kubectl port-forward -n sonarqube svc/sonarqube-sonarqube 9000:9000
```

Browser: `http://localhost:9000`
- Login: `admin` / `admin`
- Beim ersten Login: Passwort ändern zu `admin123` (oder beliebig)

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
# Manuellen Sync triggern (um Hooks zu sehen)
# In Argo CD UI: App "hooks-demo" → Sync Button

# Hook-Jobs anschauen
kubectl get jobs -n default | Select-String -Pattern "presync|postsync"

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

### Demo: SonarQube Replicas ändern

```yaml
# In sonarqube/values.yaml
replicaCount: 2  # vorher: 1
```

```powershell
git commit -am "Scale SonarQube to 2 replicas"
git push

# Argo CD erkennt Change (nach max. 3 Min)
kubectl get pods -n sonarqube -w
# → Zweiter Pod startet automatisch
```

### Demo: Drift Detection

```powershell
# Manuell im Cluster ändern
kubectl scale deployment sonarqube-sonarqube -n sonarqube --replicas=3

# Argo CD zeigt "OutOfSync"
kubectl get application -n argocd sonarqube

# Self-Heal aktiviert → nach ~1 Min zurück auf 2 Replicas
kubectl get pods -n sonarqube -w
```

## Cheat Sheet: Wichtige Befehle

```powershell
# Argo CD Apps auflisten
kubectl get applications -n argocd

# App-Status details
kubectl describe application sonarqube -n argocd

# Manuell Sync triggern (via kubectl)
kubectl patch application sonarqube -n argocd --type merge -p '{\"operation\":{\"sync\":{}}}'

# App löschen (inkl. Ressourcen im Cluster)
kubectl delete application sonarqube -n argocd

# Helm Repos in Argo CD auflisten
kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=repository

# Argo CD Password neu holen
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```

## Troubleshooting

| Problem | Lösung |
|---------|--------|
| SonarQube Pod bleibt in `Pending` | Zu wenig RAM → `minikube delete` und mit `--memory=6144` neu starten |
| PostgreSQL Pod crasht | Init dauert lange → `kubectl logs -n sonarqube <pod>` checken, 2-3 Min warten |
| Helm Chart nicht gefunden | Repo registriert? `kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=repository` prüfen |
| Sync Waves ignoriert | Annotations müssen Strings sein: `"0"` nicht `0` |
| Hook-Job bleibt in `Error` | Logs prüfen: `kubectl logs job/<job-name>`, dann Hook löschen und neu deployen |
| OutOfSync obwohl nichts geändert | Helm Chart hat `generateName` o.ä. → Ignorieren oder Chart pinnen |
| Values aus Git werden nicht geladen | Multi-Source Feature prüfen (Argo CD >= 2.6) oder Inline-Values nutzen |

## Best Practices

1. **Chart-Versionen pinnen**: z. B. `targetRevision: 10.6.0+3033` statt `latest` — konkrete Version vor Nutzung prüfen, z. B. mit `helm search repo sonarqube/sonarqube --versions`
2. **Waves sinnvoll nummerieren**: 0, 10, 20, 30 statt 0, 1, 2, 3 (Platz für Zwischenschritte)
3. **Secrets nie in values.yaml**: Sealed Secrets oder External Secrets nutzen (Session 4)
4. **Resource Limits setzen**: Cluster-Überlastung vermeiden
5. **Hook Delete Policy**: `BeforeHookCreation` für wiederholbare Hooks

## Key Learnings

✅ **Helm + Argo CD = Best of Both Worlds**
- Vendor-Charts nutzen (SonarQube, PostgreSQL, ...)
- Eigene Konfiguration in Git

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

- **Secrets Management**: Sealed Secrets oder External Secrets Operator
- **Multi-Environment**: dev/staging/prod mit Kustomize Overlays
- **App-of-Apps Pattern**: Parent-App deployt Child-Apps
- **Webhooks**: Instant-Sync statt 3-Min-Polling
- **Progressive Delivery**: Canary/Blue-Green mit Argo Rollouts

