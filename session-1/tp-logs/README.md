# TP Logs basiques — Pack technique

> Module **BOTE848 — Supervision & Observabilité DevOps**
> Session 1 / Bloc 4 (1 heure de pratique)

Ce dossier contient tout ce qu'il faut pour animer le TP :

| Fichier | Pour qui | Quand l'utiliser |
|---|---|---|
| `docker-compose.yml` | Tout le monde | Démarrage du TP, Phase 0 |
| `tp-data/` | Containers + participants | Sources des logs (3 fichiers) |
| `tp1-instructions.md` | Participants | Distribué en début de TP |

---

## Préparation en amont (à faire la veille)

```bash
# 1. Tester que tout démarre
docker compose up -d
docker compose ps          # les 2 containers doivent être Up
docker logs app-sample | head -5
docker logs app-sample-json | head -3

# 2. Vérifier le fichier incident
cat tp-data/incident-logs.json | head -1 | jq .

# 3. Arrêter
docker compose down
```

**Si la commande `docker compose` (sans tiret) ne marche pas**, utiliser `docker-compose` (avec tiret). Les commandes du guide participant supportent les deux.

---

## Prérequis participants

Le guide les liste, mais à vérifier la veille :

- ✅ **Docker** (Engine + Compose plugin OU docker-compose standalone)
- ✅ **jq** (`apt-get install jq` / `brew install jq` / `winget install jqlang.jq`)
- ✅ **Terminal bash** ou compatible (PowerShell suffit avec quelques adaptations)

> Si un participant ne peut pas installer `jq`, prévoir un fallback : `docker run --rm -i imega/jq` (image Docker `imega/jq`).

---

## Timing recommandé (1 h)

| Phase | Durée | Sujet |
|---|---|---|
| 0 — Préparation | 5 min | Démarrage des containers, vérifs |
| 1 — Logs non structurés | 15 min | Observation, frustration pédagogique |
| 2 — Logs JSON | 15 min | Le déclic |
| 3 — Requêtes `jq` | 15 min | Exercices progressifs |
| 4 — Diagnostic d'incident | 10 min | Mise en situation SRE on-call |

Plus 5 min de marge.

---

## Régénération des datasets

Les 3 fichiers de `tp-data/` sont produits par `scripts/generate-logs.py`. Seed fixée (`random.seed(42)`), donc reproductible à l'identique. Pour régénérer :

```bash
python3 scripts/generate-logs.py
```

Si tu modifies le scénario (plus d'erreurs, autre type d'incident, etc.), édite ce script et relance.

---

## Dépannage rapide

| Problème côté participant | Solution |
|---|---|
| « docker: command not found » | Vérifier l'installation Docker Desktop / Docker Engine |
| « port already in use » | Aucun port n'est exposé dans ce TP, c'est ailleurs : `docker ps` pour voir |
| `docker logs` vide | Containers pas encore démarrés — relancer `docker compose up -d` |
| `jq: command not found` | Installer jq ou utiliser le fallback Docker |
| Erreur de parsing JSON | Vérifier que le fichier n'a pas été modifié — regénérer avec le script |
