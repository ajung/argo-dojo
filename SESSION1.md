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

