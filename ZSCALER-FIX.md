# Quick Fix: Helm Repository mit Zscaler/Proxy

## Problem
Argo CD kann Helm Repositories nicht erreichen wegen Corporate Proxy (Zscaler).

Fehlermeldung: `x509: certificate signed by unknown authority`

---

## ⚡ Schnellste Lösung (Empfohlen für Dojo)

TLS-Verifizierung deaktivieren:

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
  tlsClientConfig: |
    insecure: true
EOF
```

Dann in Argo CD UI prüfen: Settings → Repositories → "sonarqube" sollte Status "Successful" haben.

---


## 🔒 Produktive Lösung: Zscaler-Zertifikat injizieren

### Schritt 1: Zertifikat exportieren

**Chrome/Edge:**
1. Einstellungen → Datenschutz und Sicherheit → Sicherheit
2. Zertifikate verwalten
3. Tab "Vertrauenswürdige Stammzertifizierungsstellen"
4. Finde "Zscaler Root CA" (oder ähnlich)
5. Exportieren → Base-64-codiert X.509 (.CER)
6. Speichern als `zscaler-root.crt`

**Firefox:**
1. Einstellungen → Datenschutz & Sicherheit → Zertifikate
2. Zertifikate anzeigen → Zertifizierungsstellen
3. "ZscalerRootCertificate" (oder ähnlich)
4. Exportieren → Speichern als `zscaler-root.crt`

### Schritt 2: In Argo CD injizieren

```powershell
# ConfigMap erstellen
kubectl create configmap zscaler-ca -n argocd --from-file=ca.crt=zscaler-root.crt

# Repo Server patchen
kubectl patch deployment argocd-repo-server -n argocd --type json -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/volumes/-",
    "value": {
      "name": "custom-ca",
      "configMap": {
        "name": "zscaler-ca"
      }
    }
  },
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/volumeMounts/-",
    "value": {
      "name": "custom-ca",
      "mountPath": "/etc/ssl/certs/zscaler.crt",
      "subPath": "ca.crt"
    }
  },
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/env/-",
    "value": {
      "name": "SSL_CERT_FILE",
      "value": "/etc/ssl/certs/zscaler.crt"
    }
  }
]'

# Pods neu starten
kubectl rollout restart deployment argocd-repo-server -n argocd

# Warten bis bereit
kubectl rollout status deployment argocd-repo-server -n argocd
```

### Schritt 3: Repository normal registrieren

Jetzt OHNE `insecure: true`:

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

---

## ✅ Verifizierung

```powershell
# Repo-Status prüfen
kubectl get secret -n argocd sonarqube-helm-repo

# In Argo CD UI:
# Settings → Repositories → "sonarqube" sollte grün sein

# Logs prüfen bei Problemen
kubectl logs -n argocd deployment/argocd-repo-server --tail=50
```

---

## 📝 Empfehlung

- **Für Dojo/Demo:** Schnellste Lösung (`insecure: true`)
- **Für Produktion:** Produktive Lösung (Zertifikat injizieren)

