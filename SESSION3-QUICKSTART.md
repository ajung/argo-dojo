# Session 3: Quick Start Guide

## Schnelleinstieg

### 1. Vorbereitungen

```powershell
minikube start --driver=podman --memory=4096 --cpus=2

# Argo CD muss laufen (aus Session 2)
kubectl get pods -n argocd
```

### 2. Helm Repository registrieren

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

### 3. Demo-Apps deployen

```powershell
kubectl apply -f grafana/application.yaml
kubectl apply -f sync-waves-demo/application.yaml
kubectl apply -f hooks-demo/application.yaml

kubectl get applications -n argocd
```

### 4. Grafana UI öffnen

```powershell
kubectl get pods -n grafana -w
kubectl port-forward -n grafana svc/grafana 3000:80
```

Browser: `http://localhost:3000` · Login: `admin` / `dojo-admin`

### 5. Cleanup

```powershell
kubectl delete application grafana sync-waves-demo hooks-demo -n argocd
```
