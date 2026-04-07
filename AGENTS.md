# AGENTS Guide

## Zweck & Scope
- Dieses Repo ist ein **GitOps-Dojo**: Argo CD synchronisiert Kubernetes-Objekte aus Git (`app/`) in den Cluster.
- Fokus ist Session-Setup und Lernfluss, nicht ein produktionsreifes Multi-Service-System.

## Repo-Karte (entscheidend für Agenten)
- `README.md`: Übersicht mit Links zu allen Sessions.
- `SESSION1.md` / `SESSION2.md` / `SESSION3.md`: Vollständige Session-Dokumentationen.
- `app/deployment.yaml`: Session 2 Demo-App (Deployment + Service `demo-app`).
- `grafana/`: Session 3 Helm-Integration (values.yaml + application.yaml für Grafana Chart, kein DB nötig).
- `sync-waves-demo/`: Session 3 Sync-Waves-Beispiele (wave-0 bis wave-3, zeigt Deployment-Reihenfolge).
- `hooks-demo/`: Session 3 Sync-Hooks-Beispiele (PreSync/PostSync Jobs).
- Es gibt aktuell **keine** Build-, Test- oder CI-Dateien im Repo; Änderungen sind primär YAML/GitOps-bezogen.

## Big Picture Architektur
- **Source of Truth:** Git-Stand im Repo.
- **Reconcile-Komponente:** Argo CD (`Application` in Namespace `argocd`) pullt `path: app` aus `targetRevision: HEAD`.
- **Runtime-Ziel:** Kubernetes `default` Namespace; App ist `demo-app` mit NGINX (`nginx:1.25`).
- Erwartetes Verhalten laut README: Drift wird durch `syncPolicy.automated.prune/selfHeal` automatisch korrigiert.

## Kritische Workflows
- Lokale Basis: Minikube + Podman (Windows/PowerShell-first laut `README.md`).
- Argo CD Installationsmuster: bei CRD-Größe immer `kubectl apply ... --server-side` nutzen.
- Sichtprüfung nach Änderungen: `kubectl get pods -w` verwenden, statt manuell Ressourcen zu patchen.
- UI-Zugriff erfolgt per Port-Forward auf `argocd-server` (`8080:443`).

## Projekt-spezifische Konventionen
- `app/deployment.yaml` enthält **Deployment und Service in einer Datei** (mit `---` getrennt).
- Label-Konsistenz ist zentral: `app: demo-app` muss in `metadata.labels`, `selector.matchLabels` und Pod-Template übereinstimmen.
- Änderungsbeispiel aus dem Dojo: `spec.replicas` wird als GitOps-Demo von `1` auf `3` angepasst.
- Keine "kubectl apply"-Direktänderungen an der App im normalen Ablauf; Änderungen gehen via Commit/Push.
- **Helm Values Pattern (Session 3)**: Chart aus externem Helm Repo, Values aus diesem Git Repo (Trennung von Vendor-Logic und Custom-Config).
- **Sync Wave Nummerierung**: 0=Namespace/ConfigMap, 1=Secrets, 2=Deployments, 3=Services (Abstand 10+ lassen für spätere Zwischenschritte).
- **Hook Delete Policies**: `BeforeHookCreation` für wiederholbare Jobs, `HookSucceeded` zum Aufräumen nach Erfolg.

## Integrationen & externe Abhängigkeiten
- Kubernetes lokal über Minikube.
- Container-Laufzeit lokal über Podman (`podman machine` unter Windows).
- GitOps-Controller: Argo CD (CRDs + `argocd` Namespace + UI/Port-Forward).
- Upstream-Manifestquelle in der Doku: `argoproj/argo-cd` Install-Manifest (Raw GitHub URL).

## Agenten-Checkliste vor Änderungen
- Prüfen, ob die Änderung in `app/` GitOps-kompatibel und deklarativ ist.
- Bei Deployment-Änderungen Selektor/Labels/Ports auf Konsistenz mit Service prüfen.
- README-Schritte als operative Wahrheit behandeln, besonders Troubleshooting-Hinweise.
- Wenn Argo-CD-`Application` geändert werden soll: `source.path` bleibt auf `app`, sofern nicht explizit anders gefordert.
