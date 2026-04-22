# Session 4: Privates GitLab Repo mit Self-Signed Cert & Access Token

## Ziel

Ein privates GitLab-Repository als Argo CD Source anbinden – inklusive der typischen Enterprise-Hürden: Self-Signed Zertifikate und Token-basierte Authentifizierung.

## Was ist neu?

- **Private Repos**: Argo CD mit authentifizierten Git-Quellen verbinden
- **Self-Signed Certificates**: TLS-Vertrauen für interne GitLab-Instanzen herstellen
- **Access Tokens**: GitLab Project/Personal Access Tokens sicher in Argo CD einbinden
- **TLS ConfigMap**: Argo CD Custom CA Bundles konfigurieren

## Warum ist das relevant?

In Enterprise-Umgebungen sind fast alle Git-Repos:
- **Privat** (kein anonymer Zugriff)
- **Hinter Corporate Proxy** (Zscaler, ZPA, etc.)
- **Mit internen CAs signiert** (Self-Signed oder Corporate Root CA)

Ohne diese Konfiguration scheitert Argo CD mit:
```
x509: certificate signed by unknown authority
rpc error: code = Unknown desc = authentication required
```

> **Praxis-Erkenntnis:** Auch wenn das Zertifikat von einer öffentlichen CA (z.B. Let's Encrypt) stammt, bricht Zscaler die TLS-Verbindung auf und ersetzt es durch ein eigenes. Der `argocd-repo-server` im Cluster kennt das Zscaler-Zertifikat nicht → gleicher Fehler wie bei Self-Signed.

## Voraussetzung

```powershell
# Cluster muss laufen
minikube start --driver=podman --memory=4096 --cpus=2

# Argo CD muss installiert sein (siehe Session 2)
kubectl get pods -n argocd
```

## Repo-Struktur (erweitert)

```
argo-dojo/
├── ...                              # Bisherige Dateien
└── private-gitlab-demo/
    ├── repo-secret.yaml             # Template: GitLab Repo-Secret mit Token
    ├── tls-configmap.yaml           # Custom CA Cert für Argo CD
    └── application.yaml             # Argo CD Application für privates Repo
```

## Vorbereitung: Manifeste im GitLab-Repo anlegen

Bevor Argo CD etwas synchronisieren kann, muss das private Repo Kubernetes-Manifeste enthalten.

> ⚠️ Dafür braucht man **Push-Rechte** auf das Repo (Rolle: Developer/Maintainer). Der `read_repository`-Token reicht dafür **nicht**. Manifeste entweder per **GitLab Web-IDE** oder mit einem Account mit Schreibrechten anlegen.

### Via GitLab Web-IDE (empfohlen für Dojo)

1. Im Browser öffnen: `https://gitlab.zt.msg.team/gbl/msg-technologie/cicd/argocd/sonarqube`
2. **Web IDE** öffnen (Button oben rechts oder `.` Taste)
3. Neuen Ordner `manifests/` erstellen
4. Datei `manifests/deployment.yaml` anlegen:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sonarqube-demo
  labels:
    app: sonarqube-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sonarqube-demo
  template:
    metadata:
      labels:
        app: sonarqube-demo
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: sonarqube-demo
  labels:
    app: sonarqube-demo
spec:
  selector:
    app: sonarqube-demo
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

5. Commit auf `main` Branch

## Teil 1: GitLab Access Token erstellen

### Schritt 1: Project Access Token generieren

In GitLab: **Project → Settings → Access Tokens**

Token erstellen:
- **Name:** `argocd-readonly`
- **Expiration:** 90 Tage (oder nach Policy)
- **Role:** `Reporter` (minimale Rolle für Lesezugriff)
- **Scopes:** `read_repository`

Token kopieren und sicher aufbewahren!

> ⚠️ **Project Access Token** ist bevorzugt (Least Privilege). Personal Access Token hat Zugriff auf ALLE Repos des Users.

### GitLab Token-Scopes Referenz

| Scope | Beschreibung | Für Argo CD nötig? |
|-------|-------------|-------------------|
| `read_repository` | Lesezugriff (pull) auf das Repo | ✅ **Ja – das reicht!** |
| `write_repository` | Lese- und Schreibzugriff (pull/push) | ❌ Nein (nur wenn Argo CD zurückschreiben soll) |
| `api` | Voller API-Zugriff | ❌ Nein (zu viele Rechte) |
| `read_api` | Lese-API-Zugriff | ❌ Nein |
| `read_registry` | Container Registry Lesezugriff | ❌ Nein (nur für Image Pull) |
| `write_registry` | Container Registry Schreibzugriff | ❌ Nein |
| `create_runner` | Runner erstellen | ❌ Nein |
| `manage_runner` | Runner verwalten | ❌ Nein |
| `k8s_proxy` | Kubernetes API via GitLab Agent | ❌ Nein |
| `self_rotate` | Token kann sich selbst rotieren | 🔄 Optional (für automatische Rotation) |
| `ai_features` | GitLab Duo AI-Features | ❌ Nein |

> **Merke:** Für Argo CD reicht `read_repository` – Argo CD liest nur, es schreibt nie ins Repo zurück.

### Schritt 2: Zugriff testen

```powershell
# Token testen – Username ist der Token-Name!
git clone https://argocd-readonly:<DEIN-TOKEN>@gitlab.zt.msg.team/gbl/msg-technologie/cicd/argocd/sonarqube.git sonarqube-test

# Aufräumen
Remove-Item sonarqube-test -Recurse -Force
```

Falls `x509: certificate signed by unknown authority` → weiter zu Teil 2.

## Teil 2: Self-Signed Certificate Challenge

### Problem verstehen

```
FATA[0000] rpc error: code = Unknown desc = x509: certificate signed by unknown authority
```

Argo CD (genauer: der `argocd-repo-server`) vertraut nur Standard-CAs. Auch wenn gitlab.zt.msg.team ein Let's Encrypt-Zertifikat hat – **Zscaler ersetzt es durch ein eigenes**. Der Container im Cluster kennt die Zscaler-CA nicht.

**Zwei Lösungswege:**

| Option | Ansatz | Wann nutzen? |
|--------|--------|-------------|
| **A: TLS ConfigMap** | CA-Zertifikat in Argo CD hinterlegen | ✅ Produktion |
| **B: Insecure Skip** | TLS-Prüfung überspringen | ⚡ Dojo / Quickfix |

### Option A: Zertifikat exportieren und einbinden

**Schritt 1: Zertifikat vom GitLab-Server holen**

```powershell
# Variante 1: PowerShell (Windows-nativ, empfohlen)
$url = "https://gitlab.zt.msg.team"
$request = [System.Net.HttpWebRequest]::Create($url)
$request.ServerCertificateValidationCallback = { $true }
try { $response = $request.GetResponse(); $response.Close() } catch {}
$cert = New-Object System.Security.Cryptography.X509Certificates.X509Certificate2($request.ServicePoint.Certificate)
$pem = "-----BEGIN CERTIFICATE-----`n" + [Convert]::ToBase64String($cert.RawData, [Base64FormattingOptions]::InsertLineBreaks) + "`n-----END CERTIFICATE-----"
$pem | Out-File -FilePath gitlab-ca.crt -Encoding ASCII -NoNewline
Write-Host "Issuer: $($cert.Issuer)"
# Erwartete Ausgabe bei Zscaler: Issuer enthält "Zscaler"
# Ohne Proxy: Issuer = "CN=E8, O=Let's Encrypt, C=US"
```

```powershell
# Variante 2: openssl (Git Bash)
openssl s_client -connect gitlab.zt.msg.team:443 -showcerts </dev/null 2>/dev/null | openssl x509 -outform PEM > gitlab-ca.crt
```

```powershell
# Variante 3: Browser-Export
# 1. https://gitlab.zt.msg.team im Browser öffnen
# 2. Schloss-Symbol → Zertifikat → Details → In Datei exportieren
# 3. Format: Base-64-codiert X.509 (.CER)
# 4. Speichern als gitlab-ca.crt
```

**Schritt 2: ConfigMap in Argo CD erstellen**

```powershell
kubectl create configmap argocd-tls-certs-cm `
  --from-file=gitlab.zt.msg.team=gitlab-ca.crt `
  -n argocd `
  --dry-run=client -o yaml | kubectl apply -f -
```

**Schritt 3: Repo-Server neustarten**

```powershell
# Repo-Server muss die neue ConfigMap laden
kubectl rollout restart deployment argocd-repo-server -n argocd
kubectl rollout status deployment argocd-repo-server -n argocd
```

### Option B: Insecure Skip (Dojo-Quickfix)

Im Repository-Secret einfach `insecure: "true"` setzen (siehe Teil 3). Das überspringt die Zertifikatsprüfung komplett – **kein Zertifikat-Export nötig**.

> ⚠️ **Option B ist NICHT für Produktion geeignet!** Nur als Workaround im Dojo oder bei Zscaler-Problemen akzeptabel.

## Teil 3: Repository in Argo CD registrieren

### Schritt 1: Repository-Secret anlegen

**Für das Dojo (mit insecure):**

```powershell
@"
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-private-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://gitlab.zt.msg.team/gbl/msg-technologie/cicd/argocd/sonarqube.git
  username: argocd-readonly
  password: <DEIN-GITLAB-TOKEN>
  insecure: "true"
"@ | kubectl apply -f -
```

**Für Produktion (mit CA-Cert, ohne insecure):**

```powershell
@"
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-private-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://gitlab.zt.msg.team/gbl/msg-technologie/cicd/argocd/sonarqube.git
  username: argocd-readonly
  password: <DEIN-GITLAB-TOKEN>
"@ | kubectl apply -f -
```

> **Hinweis:** Bei Project Access Tokens ist der `username` der **Token-Name** (hier: `argocd-readonly`), das `password` ist der **Token-Wert** (z.B. `glpat-...`).

### Schritt 2: Verbindung in Argo CD UI prüfen

```powershell
# Port-Forward (falls nicht schon aktiv)
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

In der UI: **Settings → Repositories**
- Status sollte **"Successful"** zeigen
- Falls "Failed": Fehlermeldung gibt Hinweis (Auth vs. TLS)

### Schritt 3: Verbindung via CLI testen (optional)

```powershell
# Argo CD CLI Login
argocd login localhost:8080 --insecure --username admin `
  --password $(kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d)

# Repo-Liste anzeigen
argocd repo list
```

## Teil 4: Application aus privatem Repo deployen

### Schritt 1: Application anlegen

```powershell
kubectl apply -f private-gitlab-demo/application.yaml
```

Die Application zeigt auf `path: manifests` im GitLab-Repo (dort liegen die Kubernetes-Manifeste aus der Vorbereitung).

### Schritt 2: Deployment beobachten

```powershell
# Application Status prüfen
kubectl get application -n argocd private-gitlab-app
# Erwartete Ausgabe:
# NAME                 SYNC STATUS   HEALTH STATUS
# private-gitlab-app   Synced        Healthy

# Detailstatus
kubectl describe application private-gitlab-app -n argocd

# Pods beobachten
kubectl get pods -n default -w
```

### Schritt 3: GitOps-Flow testen

1. Im GitLab-Repo Änderung committen (z.B. `replicas: 2` in `manifests/deployment.yaml`)
2. Warten (~3 Min Polling oder manueller Sync in der UI)
3. Argo CD erkennt Diff und synchronisiert automatisch

```powershell
# Manuellen Sync auslösen (statt auf Polling zu warten)
argocd app sync private-gitlab-app
```

## Aufräumen

```powershell
# Application löschen (entfernt auch die Kubernetes-Ressourcen)
kubectl delete application private-gitlab-app -n argocd

# Repository-Secret löschen
kubectl delete secret gitlab-private-repo -n argocd

# TLS ConfigMap zurücksetzen (optional)
kubectl delete configmap argocd-tls-certs-cm -n argocd
```

## Troubleshooting

| Problem | Lösung |
|---------|--------|
| **x509: certificate signed by unknown authority** | CA-Cert in `argocd-tls-certs-cm` ConfigMap eintragen ODER `insecure: "true"` im Repo-Secret |
| **authentication required** | Token falsch oder abgelaufen → neuen Token erstellen, Secret updaten |
| **repository not found** | URL prüfen (inkl. `.git` am Ende), Token-Scope prüfen (`read_repository`) |
| **Permission denied** | Token hat nicht genug Rechte oder falsches Projekt |
| **insufficient_scope (403)** | Token hat nur `read_repository` – Push braucht `write_repository`. Für Argo CD ist `read_repository` korrekt! |
| **Repo "Successful" aber App "ComparisonError"** | `path` in Application stimmt nicht mit Ordnerstruktur im Repo überein |
| **Cert nach Restart weg** | ConfigMap nicht persistiert? Deklarativ via YAML anlegen statt imperativ |
| **Zscaler ersetzt Zertifikat** | Zscaler Root CA exportieren und als Custom CA einbinden (siehe ZSCALER-FIX.md) |
| **Token expired** | GitLab zeigt 401 → neuen Token erstellen, Secret updaten: `kubectl edit secret gitlab-private-repo -n argocd` |

## Debugging-Checkliste

```powershell
# 1. Repo-Server Logs (zeigt TLS/Auth-Fehler)
kubectl logs deployment/argocd-repo-server -n argocd --tail=50

# 2. Application-Controller Logs (zeigt Sync-Fehler)
kubectl logs deployment/argocd-application-controller -n argocd --tail=50

# 3. Registrierte Repos prüfen
kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=repository

# 4. TLS ConfigMap prüfen
kubectl get configmap argocd-tls-certs-cm -n argocd -o yaml

# 5. Secret-Inhalt prüfen (Vorsicht: zeigt Token!)
kubectl get secret gitlab-private-repo -n argocd -o jsonpath="{.data.url}" | base64 -d
```

## Security Best Practices

1. **Project Access Tokens** statt Personal Access Tokens (Least Privilege)
2. **`read_repository` Scope** reicht für Argo CD (kein `write` nötig)
3. **Reporter-Rolle** reicht als Minimum für den Token
4. **Token-Rotation**: Ablaufdatum setzen, regelmäßig erneuern
5. **Kein `insecure: true` in Produktion** – immer CA-Cert einbinden
6. **Secrets nicht in Git**: Repo-Secret per `kubectl apply` oder Sealed Secrets anlegen
7. **RBAC in GitLab**: Dedizierter Service-Account mit minimalen Rechten
8. **Audit**: Token-Nutzung in GitLab unter Settings → Access Tokens überwachen

## Key Learnings

✅ **Private Repos brauchen zwei Dinge: Auth + TLS-Trust**
- Auth: GitLab Access Token als Argo CD Repository Secret
- TLS: Custom CA in `argocd-tls-certs-cm` ConfigMap (oder `insecure` als Quickfix)

✅ **Self-Signed Certs und Zscaler sind das gleiche Problem**
- Zscaler ersetzt auch öffentliche Zertifikate (Let's Encrypt → Zscaler Root CA)
- Im Container fehlt die Zscaler CA → gleicher Fehler wie Self-Signed
- Lösung: CA exportieren + ConfigMap ODER `insecure: "true"`

✅ **GitOps funktioniert auch mit privaten Repos**
- Gleicher Workflow wie mit öffentlichen Repos
- Einmalige Secret-Konfiguration, danach transparent
- Argo CD braucht nur Lesezugriff (`read_repository`)

✅ **Enterprise-Hürden sind Konfiguration, keine Blocker**
- Proxy, Self-Signed Certs, Token-Auth – alles deklarativ lösbar
- Troubleshooting-Flow: Logs → Secret → ConfigMap → Restart

## Ausblick Session 5

- **Sealed Secrets**: Secrets verschlüsselt in Git speichern
- **App-of-Apps Pattern**: Parent-App deployt alle Child-Apps
- **Multi-Environment**: dev/staging/prod mit Kustomize Overlays
- **Notifications**: Slack/Teams-Alerts bei Sync-Events

