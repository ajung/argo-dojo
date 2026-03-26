# Argo CD Coding Dojo

| Aspekt          | Beschreibung                                                            |
|-----------------|-------------------------------------------------------------------------|
| Ziel            | Förderung von praktischem Lernen und kontinuierlicher Weiterentwicklung |
| Format          | Team Programming (gemeinsames, kollaboratives Coden)                    |
| Fokus           | Hands-on-Erfahrung statt Theorie                                        |
| Inhalte         | Deep Dives in neue Technologien und Themen                              |
| Experimentieren | Raum für Ausprobieren neuer Ansätze und Lösungen                        |
| Teilnahme       | Offen für alle Teammitglieder                                           |
| Rhythmus        | Regelmäßig, einmal pro Woche                                            |
| Dauer           | Zeitlich begrenzte Serie von 4–6 Terminen                               |

---

## Session 1: Grundsetup – Podman & Minikube

### Ziel

Aufsetzen einer lokalen Kubernetes-Umgebung mit Podman und Minikube als Basis für weitere Sessions.

### Voraussetzungen

- Windows 10/11
- Administratorrechte
- PowerShell

### Installation Podman

```powershell
winget install -e --id RedHat.Podman
```

Nach der Installation: System neu starten oder PowerShell neu öffnen.

### Podman initialisieren

```powershell
podman machine init
podman machine start
podman info
```

### Installation Minikube

```powershell
winget install -e --id Kubernetes.minikube
```

### Minikube starten (mit Podman als Treiber)

```powershell
minikube start --driver=podman
```

### Installation kubectl

```powershell
winget install -e --id Kubernetes.kubectl
```

### Cluster prüfen

```powershell
kubectl get nodes
```

### Dashboard starten (optional)

```powershell
minikube dashboard
```

### Hinweise

- Podman läuft in einer VM (`podman machine`) unter Windows
- Minikube nutzt Podman als Container-Runtime
- Falls Probleme auftreten: `minikube delete` und neu starten

---

### Erfahrungen aus der Praxis (Teilnehmer-Feedback)

#### Variante: `--container-runtime=containerd` + Rootless-Konfiguration

Bei manchen Setups schlägt `minikube start --driver=podman` ohne weitere Flags fehl.
Lösung: Podman auf **rootless** konfigurieren und dann mit `--container-runtime=containerd` starten:

```powershell
minikube start --driver=podman --container-runtime=containerd
```

#### Variante: Einfach nochmal probieren / `minikube delete`

Bei einigen Teilnehmern hat `minikube start --driver=podman` beim ersten Versuch nicht funktioniert.
Ein `minikube delete` gefolgt von einem erneuten `minikube start --driver=podman` hat das Problem gelöst – ohne weitere Anpassungen.

#### Variante: Ressourcen der Podman-Machine erhöhen

Bei Problemen nach einer Neu-Installation von Minikube kann es helfen, die Podman-Machine mit mehr Ressourcen neu aufzusetzen:

```powershell
podman machine rm
podman machine init --cpus 2 --memory 4096 --disk-size 60
podman machine start
minikube delete
minikube start --driver=podman
```

#### Fallback: Minikube über WSL installieren

Falls Minikube unter Windows gar nicht erkannt wird (der Befehl bleibt unbekannt), kann Minikube direkt **in der WSL** installiert und gestartet werden.
Das hat bei einigen Teilnehmern letztendlich funktioniert, nachdem die native Windows-Installation nicht zum Ziel geführt hat.

---

## Session 2: ArgoCD & GitOps

### Ziel

ArgoCD lokal installieren und das GitOps-Prinzip live erleben: eine Änderung im Git-Repo wird automatisch in den Cluster übernommen.

### GitOps – Drei Kernsätze

1. **Git ist die einzige Wahrheit.** Was im Repo steht, läuft im Cluster – nicht mehr, nicht weniger.
2. **ArgoCD pullt, niemand pusht in den Cluster.** Kein `kubectl apply` von Hand im Betrieb.
3. **Drift wird automatisch erkannt und geheilt.** Jemand patcht manuell? ArgoCD rollt es zurück.

### Minikube starten

```bash
minikube start --memory=4096
```

### ArgoCD installieren

> ⚠️ Das Standard-`kubectl apply` schlägt mit folgendem Fehler fehl:
> `The CustomResourceDefinition "applicationsets.argoproj.io" is invalid: metadata.annotations: Too long: may not be more than 262144 bytes`
>
> **Fix:** `--server-side` verwenden – die CRDs sind zu groß für client-side apply.

```bash
kubectl create namespace argocd

kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml \
  --server-side
```

### Warten bis alles läuft

```bash
kubectl wait --for=condition=available deployment \
  --all -n argocd --timeout=180s
```

### UI zugänglich machen

In einem separaten Terminal-Tab laufen lassen:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### Initial-Passwort holen

```bash
kubectl get secret argocd-initial-admin-secret \
  -n argocd -o jsonpath="{.data.password}" | base64 -d
```

**Login:** `https://localhost:8080` · Benutzer: `admin` · Passwort: siehe oben

### Demo-Repo

**Repo:** https://github.com/ajung/argo-dojo.git

```
repo/
└── app/
    └── deployment.yaml   # nginx, replicas: 1
```

#### deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
  labels:
    app: demo-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
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
  name: demo-app
spec:
  selector:
    app: demo-app
  ports:
    - port: 80
      targetPort: 80
```

### App in ArgoCD registrieren

```bash
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: demo-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/ajung/argo-dojo.git
    targetRevision: HEAD
    path: app
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF
```

### GitOps in Aktion

`replicas: 1` → `replicas: 3` im Repo ändern, committen und pushen. Dann beobachten:

```bash
kubectl get pods -w
```

ArgoCD erkennt den Drift und gleicht den Cluster-Zustand automatisch ab.

### Ausblick – mögliche nächste Sessions

- Multi-Environment-Setup (dev/staging/prod im selben Repo)
- Secrets-Management mit ArgoCD (Sealed Secrets, External Secrets)
- App-of-Apps-Pattern für größere Setups
- Rollback via `git revert`

### Troubleshooting

| Problem | Lösung |
|---------|--------|
| CRD-Fehler: `annotations: Too long` | `--server-side` Flag beim `kubectl apply` verwenden |
| UI nicht erreichbar | Port-Forward läuft noch? `kubectl port-forward svc/argocd-server -n argocd 8080:443` |
| Passwort-Befehl gibt leere Zeile | Secret noch nicht bereit – kurz warten, dann wiederholen |
| ArgoCD synct nicht automatisch | Default-Polling-Intervall: 3 Minuten – oder manuell „Sync" in der UI klicken |
