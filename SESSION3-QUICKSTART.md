# Session 3: Quick Start Guide

## Schnelleinstieg für Teilnehmer

### 1. Vorbereitungen

```powershell
# Minikube mit mehr Ressourcen starten
minikube delete
minikube start --driver=podman --memory=6144 --cpus=4

# Argo CD muss laufen (aus Session 2)
kubectl get pods -n argocd
```

### 2. Helm Repository registrieren

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

### 3. Demo-Apps deployen

```powershell
# Alle Demo-Apps auf einmal deployen
kubectl apply -f sonarqube/application.yaml
kubectl apply -f sync-waves-demo/application.yaml
kubectl apply -f hooks-demo/application.yaml

# Status prüfen
kubectl get applications -n argocd
```

### 4. SonarQube UI öffnen

```powershell
# Warten bis SonarQube läuft (2-3 Minuten)
kubectl get pods -n sonarqube -w

# UI öffnen
minikube service sonarqube-sonarqube -n sonarqube --url
# Oder: kubectl port-forward -n sonarqube svc/sonarqube-sonarqube 9000:9000
```

Login: `admin` / `admin`

### 5. Sync Waves beobachten

```powershell
# In Argo CD UI: https://localhost:8080
# App "sync-waves-demo" öffnen
# Sync Button → Ressourcen erscheinen in Reihenfolge:
#   Wave 0: ConfigMap
#   Wave 1: Secret
#   Wave 2: Deployment
#   Wave 3: Service
```

### 6. Hooks testen

```powershell
# In Argo CD UI: App "hooks-demo" öffnen
# Sync Button drücken
# Hook-Jobs erscheinen:
#   PreSync: presync-backup (läuft VOR Deployment)
#   PostSync: postsync-smoketest (läuft NACH Deployment)

# Logs anschauen
kubectl logs job/presync-backup -n default
kubectl logs job/postsync-smoketest -n default
```

### 7. GitOps-Änderung testen

```powershell
# In sonarqube/values.yaml:
# replicaCount: 1  →  replicaCount: 2

git commit -am "Scale SonarQube to 2 replicas"
git push

# Warten (max. 3 Min), dann:
kubectl get pods -n sonarqube -w
# → Zweiter Pod startet automatisch
```

### 8. Drift Detection demonstrieren

```powershell
# Manuell im Cluster ändern
kubectl scale deployment wave-demo-app -n default --replicas=5

# Argo CD erkennt Drift
kubectl get application -n argocd sync-waves-demo
# STATUS: OutOfSync

# Self-Heal korrigiert automatisch (nach ~1 Min)
kubectl get pods -n default -l app=wave-demo -w
# → Zurück auf 2 Replicas
```

## Troubleshooting

### SonarQube startet nicht

```powershell
# Logs prüfen
kubectl logs -n sonarqube -l app=sonarqube --tail=50

# Häufigste Ursache: Zu wenig RAM
minikube delete
minikube start --driver=podman --memory=8192 --cpus=4
```

### Waves werden nicht beachtet

```yaml
# Häufiger Fehler: Zahl statt String
#argocd.argoproj.io/sync-wave: 0  # FALSCH
argocd.argoproj.io/sync-wave: "0"  # RICHTIG
```

### Hook-Job bleibt hängen

```powershell
# Job löschen
kubectl delete job presync-backup -n default

# App neu syncen
kubectl patch application hooks-demo -n argocd --type merge -p '{"operation":{"sync":{}}}'
```

## Cleanup

```powershell
# Alle Demo-Apps löschen
kubectl delete application sonarqube sync-waves-demo hooks-demo -n argocd

# Oder nur einzelne Apps
kubectl delete application sonarqube -n argocd
```

