# Session 1: Grundsetup – Podman & Minikube

## Ziel

Aufsetzen einer lokalen Kubernetes-Umgebung mit Podman und Minikube als Basis für weitere Sessions.

## Voraussetzungen

- Windows 10/11
- Administratorrechte
- PowerShell

## Installation Podman

```powershell
winget install -e --id RedHat.Podman
```

Nach der Installation: System neu starten oder PowerShell neu öffnen.

## Podman initialisieren

```powershell
podman machine init
podman machine start
podman info
```

## Installation Minikube

```powershell
winget install -e --id Kubernetes.minikube
```

## Minikube starten (mit Podman als Treiber)

```powershell
minikube start --driver=podman
```

## Installation kubectl

```powershell
winget install -e --id Kubernetes.kubectl
```

## Cluster prüfen

```powershell
kubectl get nodes
```

## Dashboard starten (optional)

```powershell
minikube dashboard
```

## Hinweise

- Podman läuft in einer VM (`podman machine`) unter Windows
- Minikube nutzt Podman als Container-Runtime
- Falls Probleme auftreten: `minikube delete` und neu starten

---

## Erfahrungen aus der Praxis (Teilnehmer-Feedback)

### Variante: `--container-runtime=containerd` + Rootless-Konfiguration

Bei manchen Setups schlägt `minikube start --driver=podman` ohne weitere Flags fehl.
Lösung: Podman auf **rootless** konfigurieren und dann mit `--container-runtime=containerd` starten:

```powershell
minikube start --driver=podman --container-runtime=containerd
```

### Variante: Einfach nochmal probieren / `minikube delete`

Bei einigen Teilnehmern hat `minikube start --driver=podman` beim ersten Versuch nicht funktioniert.
Ein `minikube delete` gefolgt von einem erneuten `minikube start --driver=podman` hat das Problem gelöst – ohne weitere Anpassungen.

### Variante: Ressourcen der Podman-Machine erhöhen

Bei Problemen nach einer Neu-Installation von Minikube kann es helfen, die Podman-Machine mit mehr Ressourcen neu aufzusetzen:

```powershell
podman machine rm
podman machine init --cpus 2 --memory 4096 --disk-size 60
podman machine start
minikube delete
minikube start --driver=podman
```

### Fallback: Minikube über WSL installieren

Falls Minikube unter Windows gar nicht erkannt wird (der Befehl bleibt unbekannt), kann Minikube direkt **in der WSL** installiert und gestartet werden.
Das hat bei einigen Teilnehmern letztendlich funktioniert, nachdem die native Windows-Installation nicht zum Ziel geführt hat.

---

### DNS-Problem in Minikube (WSL / Podman-Driver)

**Symptom:** ArgoCD (oder andere Pods) können keine externen Hosts erreichen – Timeout-Fehler wie:
```
Get "https://github.com/...": context deadline exceeded
```

**Ursache:** DNS-Auflösung in Minikube funktioniert nicht korrekt (bekanntes Problem mit Podman-Driver in WSL).

**Lösung – 5 Schritte:**

#### 1. DNS im Minikube-Host setzen

```powershell
wsl bash -c "minikube ssh 'echo nameserver 8.8.8.8 | sudo tee /etc/resolv.conf && echo nameserver 8.8.4.4 | sudo tee -a /etc/resolv.conf'"
```

#### 2. CoreDNS ConfigMap erstellen

```powershell
wsl bash -c "cat > /tmp/coredns-config-fixed.yaml << 'YAML'
apiVersion: v1
data:
  Corefile: |
    .:53 {
        log
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        hosts {
           192.168.49.1 host.minikube.internal
           fallthrough
        }
        forward . 8.8.8.8 8.8.4.4 {
           max_concurrent 1000
        }
        cache 30 {
           disable success cluster.local
           disable denial cluster.local
        }
        loop
        reload
        loadbalance
    }
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
YAML
"
```

#### 3. CoreDNS ConfigMap anwenden

```powershell
wsl minikube kubectl -- apply -f /tmp/coredns-config-fixed.yaml
```

#### 4. CoreDNS neu starten

```powershell
wsl minikube kubectl -- rollout restart deployment coredns -n kube-system
```

#### 5. ArgoCD Pods neu starten

```powershell
wsl minikube kubectl -- delete pods --all -n argocd
```

**Verifizierung** (nach ca. 1 Minute):

```powershell
wsl minikube kubectl -- get pods -n argocd
```

> **Hinweis:** Diese Änderung ist permanent (überlebt Minikube-Restarts). CoreDNS verwendet dann direkt Google DNS (8.8.8.8, 8.8.4.4).

