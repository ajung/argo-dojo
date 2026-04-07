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

## 📚 Session-Übersicht

Jede Session ist als eigene Markdown-Datei dokumentiert mit vollständigen Schritt-für-Schritt-Anleitungen:

### [Session 1: Grundsetup – Podman & Minikube](SESSION1.md)
**Ziel:** Lokale Kubernetes-Umgebung aufsetzen
- Podman Installation & Konfiguration
- Minikube mit Podman-Driver
- kubectl Setup
- Troubleshooting-Tipps aus der Praxis

👉 **[Zur vollständigen Session 1 Dokumentation →](SESSION1.md)**

---

### [Session 2: ArgoCD & GitOps](SESSION2.md)
**Ziel:** GitOps-Prinzipien live erleben
- Argo CD Installation
- Erste Application deployen
- Automated Sync & Self-Heal
- Drift Detection demonstrieren

👉 **[Zur vollständigen Session 2 Dokumentation →](SESSION2.md)**

---

### [Session 3: Helm Charts & Sync-Mechanismen](SESSION3.md)
**Ziel:** Fortgeschrittene Argo CD Features
- Helm Integration (Grafana)
- Sync Waves für Deployment-Reihenfolge
- Sync Hooks (PreSync/PostSync)
- GitOps mit externen Charts

👉 **[Zur vollständigen Session 3 Dokumentation →](SESSION3.md)**

Siehe auch: **[Session 3 Quickstart Guide](SESSION3-QUICKSTART.md)** für schnellen Einstieg

---

## 🚀 Quick Start

### Erste Session starten

```powershell
# 1. Podman installieren
winget install -e --id RedHat.Podman

# 2. Minikube installieren
winget install -e --id Kubernetes.minikube

# 3. kubectl installieren
winget install -e --id Kubernetes.kubectl

# 4. Cluster starten
podman machine init
podman machine start
minikube start --driver=podman
```

Vollständige Anleitung: [SESSION1.md](SESSION1.md)

### Repo klonen

```powershell
git clone https://github.com/ajung/argo-dojo.git
cd argo-dojo
```

---

## 📁 Repo-Struktur

```
argo-dojo/
├── README.md                    # Diese Übersicht
├── SESSION1.md                  # Session 1: Podman & Minikube Setup
├── SESSION2.md                  # Session 2: ArgoCD & GitOps
├── SESSION3.md                  # Session 3: Helm & Sync-Mechanismen
├── SESSION3-QUICKSTART.md       # Schnelleinstieg Session 3
├── AGENTS.md                    # Guide für AI Coding Agents
│
├── app/                         # Session 2: Demo-App
│   └── deployment.yaml
│
├── grafana/                     # Session 3: Helm Integration
│   ├── values.yaml
│   └── application.yaml
│
├── sync-waves-demo/             # Session 3: Sync Waves Demo
│   ├── wave-0-configmap.yaml
│   ├── wave-1-secret.yaml
│   ├── wave-2-deployment.yaml
│   ├── wave-3-service.yaml
│   └── application.yaml
│
└── hooks-demo/                  # Session 3: Sync Hooks Demo
    ├── presync-backup.yaml
    ├── deployment.yaml
    ├── postsync-test.yaml
    └── application.yaml
```

---

## 🎯 Lernziele pro Session

### Session 1
✅ Lokale Kubernetes-Umgebung funktioniert  
✅ Podman als Container-Runtime verstanden  
✅ kubectl-Grundlagen beherrschen

### Session 2
✅ GitOps-Prinzipien verstanden  
✅ Argo CD Application deployen können  
✅ Automated Sync & Self-Heal erleben  
✅ Drift Detection & Reconciliation verstehen

### Session 3
✅ Helm Charts mit Argo CD nutzen  
✅ Sync Waves für Dependencies einsetzen  
✅ Pre/Post-Deployment Hooks implementieren  
✅ Multi-Source Applications verstehen

---

## 🛠 Voraussetzungen

- **OS:** Windows 10/11 (primär), alternativ Linux/macOS
- **RAM:** Minimum 8 GB (16 GB empfohlen für Session 3)
- **Disk:** ~20 GB freier Speicher
- **Tools:** PowerShell, Git
- **Rechte:** Lokale Admin-Rechte für Installation

---

## 📖 Weiterführende Ressourcen

- [Argo CD Dokumentation](https://argo-cd.readthedocs.io/)
- [GitOps Principles](https://www.gitops.tech/)
- [Minikube Docs](https://minikube.sigs.k8s.io/docs/)
- [Podman Docs](https://docs.podman.io/)
- [Helm Documentation](https://helm.sh/docs/)

---

## 🤝 Beiträge & Feedback

Dieses Dojo lebt von euren Erfahrungen! Wenn ihr:
- Probleme löst, die nicht dokumentiert sind
- Verbesserungsvorschläge habt
- Neue Session-Ideen einbringen möchtet

→ Pull Request oder Issue erstellen!

---

