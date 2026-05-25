# 🎓 BOTE848 — Supervision & Observabilité DevOps

**Module de formation** | Expert en Informatique et Système d'Information (RNCP 35584)
**Durée** : 12 heures — 3 séances de 4h
**Format** : Présentiel + travaux pratiques Docker
**Semestre** : S8
**Version étudiante**

---

# 📖 À propos de ce dépôt

Bienvenue dans le dépôt étudiant du module **BOTE848 — Supervision & Observabilité DevOps**.

Ce dépôt contient :

* les supports de cours des 3 séances,
* les travaux pratiques,
* les jeux de données,
* les stacks Docker nécessaires,
* les ressources techniques utiles,
* les fichiers de configuration pour les TP.

L'objectif du module est de vous faire découvrir les concepts modernes de :

* supervision,
* observabilité,
* monitoring,
* logs,
* métriques,
* traces distribuées,
* diagnostic d'incidents.

---

# 🎯 Objectifs pédagogiques

À l’issue du module, vous serez capable de :

* Comprendre les différences entre supervision et observabilité
* Manipuler des logs structurés JSON
* Utiliser Loki, Prometheus et Jaeger
* Comprendre les 3 piliers de l’observabilité
* Construire des dashboards Grafana
* Diagnostiquer un incident de production
* Comprendre les bases du monitoring DevOps moderne

---

# 📚 Structure du dépôt

```text
BOTE848-Student/
├── README.md
├── QUICK_START.md
│
├── sessions/
│   ├── session-1/
│   │   ├── slides.pdf
│   │   ├── tp-logs-basiques.md
│   │   ├── docker-compose.yml
│   │   └── tp-data/
│   │
│   ├── session-2/
│   │   ├── slides.pdf
│   │   ├── tp-loki.md
│   │   ├── tp-prometheus.md
│   │   └── infrastructure/
│   │
│   └── session-3/
│       ├── slides.pdf
│       ├── tp-jaeger.md
│       ├── cas-incident.md
│       └── grafana-dashboard.json
│
├── resources/
│   ├── documentation/
│   ├── schemas/
│   └── troubleshooting/
│
└── tools/
    └── cheatsheets/
```

---

# 🚀 Démarrage rapide

## 1. Cloner le dépôt

```bash
git clone https://github.com/[organisation]/BOTE848-Student.git
cd BOTE848-Student
```

---

## 2. Vérifier Docker

```bash
docker --version
docker compose version
```

Docker Desktop doit être démarré.

---

## 3. Aller dans une séance

Exemple Session 1 :

```bash
cd sessions/session-1
```

---

## 4. Démarrer le TP

```bash
docker compose up -d
```

---

## 5. Vérifier les containers

```bash
docker compose ps
```

---

# 🐳 Pourquoi Docker ?

Tous les TP utilisent Docker afin de :

* éviter les problèmes d’installation,
* garantir un environnement identique pour tous,
* simplifier les manipulations,
* reproduire un environnement DevOps moderne.

---

# ⚠️ Important — jq et outils Linux

Vous n’avez PAS besoin d’installer :

* jq
* grep
* awk
* outils Linux supplémentaires

Les TP fournissent un container `logs-toolbox` contenant tous les outils nécessaires.

---

## Ouvrir la toolbox

```bash
docker exec -it logs-toolbox sh
```

---

## Exemple

```bash
cat /data/app-logs.jsonl | jq .
```

---

# 📅 Programme des séances

---

# Session 1 — Fondamentaux

## Concepts abordés

* Supervision
* Observabilité
* Logs
* Métriques
* Traces
* JSON logs
* jq

## TP

* Analyse de logs
* Diagnostic d’incident
* Requêtes jq

---

# Session 2 — Loki & Prometheus

## Concepts abordés

* Centralisation des logs
* Loki
* Prometheus
* PromQL
* LogQL
* Collecte de métriques

## TP

* Requêtes Loki
* Dashboards Grafana
* Alertes Prometheus

---

# Session 3 — Traces & Diagnostic

## Concepts abordés

* Jaeger
* OpenTelemetry
* Traces distribuées
* Corrélation logs/métriques/traces

## TP

* Instrumentation applicative
* Diagnostic complet d’incident
* Dashboard d’observabilité

---

# 🛠️ Commandes Docker utiles

## Démarrer

```bash
docker compose up -d
```

---

## Arrêter

```bash
docker compose down
```

---

## Voir les logs

```bash
docker logs app-sample
```

---

## Voir les containers

```bash
docker compose ps
```

---

## Supprimer complètement les données

```bash
docker compose down -v
```

---

# 📚 Ressources utiles

## jq

* [https://jqlang.github.io/jq/manual/](https://jqlang.github.io/jq/manual/)
* [https://jqplay.org/](https://jqplay.org/)

---

## Docker

* [https://docs.docker.com/](https://docs.docker.com/)

---

## Prometheus

* [https://prometheus.io/docs/](https://prometheus.io/docs/)

---

## Grafana

* [https://grafana.com/docs/](https://grafana.com/docs/)

---

## OpenTelemetry

* [https://opentelemetry.io/docs/](https://opentelemetry.io/docs/)

---

# 🧠 Conseils pour réussir le module

* Toujours tester les commandes une par une
* Lire les logs attentivement
* Comprendre les structures JSON
* Ne pas copier/coller sans comprendre
* Utiliser `docker logs`
* Utiliser `jq` progressivement
* Documenter vos observations pendant les TP

---

# ❓ En cas de problème

## Vérifier Docker

```bash
docker ps
```

---

## Redémarrer proprement

```bash
docker compose down -v
docker compose up -d
```

---

## Vérifier les logs d’erreur

```bash
docker compose logs
```

---

# 📄 Livrables attendus

Selon les séances :

* réponses aux questions,
* captures d’écran,
* requêtes jq,
* dashboards,
* analyses d’incidents,
* rapports Markdown.

---

# 🎯 Objectif final du module

À la fin des 12h, vous devrez être capable de :

* comprendre une stack d’observabilité moderne,
* lire et analyser des logs structurés,
* diagnostiquer un incident réel,
* comprendre le fonctionnement des outils DevOps modernes,
* corréler logs, métriques et traces.

---

# 📢 Important

Ce dépôt est un dépôt pédagogique étudiant.

Il contient :

* les supports nécessaires aux TP,
* les ressources de travail,
* les environnements Docker.

Il ne contient PAS :

* les corrigés complets,
* les supports formateurs,
* les grilles pédagogiques internes.

---

# ✅ Bon travail

L’observabilité est aujourd’hui au cœur :

* du DevOps,
* du SRE,
* du Cloud,
* des architectures distribuées,
* de la production moderne.

Ce module vous donne une première expérience pratique concrète de ces problématiques.
