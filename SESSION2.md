# Session 2: ArgoCD & GitOps

## Ziel

ArgoCD lokal installieren und das GitOps-Prinzip live erleben: eine Änderung im Git-Repo wird automatisch in den Cluster übernommen.

## GitOps – Drei Kernsätze

1. **Git ist die einzige Wahrheit.** Was im Repo steht, läuft im Cluster – nicht mehr, nicht weniger.
2. **ArgoCD pullt, niemand pusht in den Cluster.** Kein `kubectl apply` von Hand im Betrieb.
3. **Drift wird automatisch erkannt und geheilt.** Jemand patcht manuell? ArgoCD rollt es zurück.

## Minikube starten

```bash
minikube start --memory=4096
```

## ArgoCD installieren

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

## Warten bis alles läuft

```bash
kubectl wait --for=condition=available deployment \
  --all -n argocd --timeout=180s
```

## UI zugänglich machen

In einem separaten Terminal-Tab laufen lassen:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

## Initial-Passwort holen

```bash
kubectl get secret argocd-initial-admin-secret \
  -n argocd -o jsonpath="{.data.password}" | base64 -d
```

**Login:** `https://localhost:8080` · Benutzer: `admin` · Passwort: siehe oben

## Demo-Repo

**Repo:** https://github.com/ajung/argo-dojo.git

```
repo/
└── app/
    └── deployment.yaml   # nginx, replicas: 1
```

### deployment.yaml

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

## App in ArgoCD registrieren

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

## GitOps in Aktion

`replicas: 1` → `replicas: 3` im Repo ändern, committen und pushen. Dann beobachten:

```bash
kubectl get pods -w
```

ArgoCD erkennt den Drift und gleicht den Cluster-Zustand automatisch ab.

## Ausblick – mögliche nächste Sessions

- Multi-Environment-Setup (dev/staging/prod im selben Repo)
- Secrets-Management mit ArgoCD (Sealed Secrets, External Secrets)
- App-of-Apps-Pattern für größere Setups
- Rollback via `git revert`

## Troubleshooting

| Problem | Lösung |
|---------|--------|
| CRD-Fehler: `annotations: Too long` | `--server-side` Flag beim `kubectl apply` verwenden |
| UI nicht erreichbar | Port-Forward läuft noch? `kubectl port-forward svc/argocd-server -n argocd 8080:443` |
| Passwort-Befehl gibt leere Zeile | Secret noch nicht bereit – kurz warten, dann wiederholen |
| ArgoCD synct nicht automatisch | Default-Polling-Intervall: 3 Minuten – oder manuell „Sync" in der UI klicken |

