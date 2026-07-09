# TP — ELK vs Loki vs Prometheus : Choisir ses outils d'observabilité

**Module BOTE848 — Supervision, Observabilité et Monitoring Avancé DevOps**  
**Séance 2 — Blocs 0, 1 & 3**  
Durée estimée : **3h**

---

## 🎯 Objectifs

À l'issue de ce TP, vous serez capable de :

- Déployer une stack ELK simplifiée et y ingérer des logs
- Déployer une stack Loki + Grafana et y ingérer les mêmes logs
- Déployer Prometheus et exploiter des métriques en temps réel
- Exécuter des requêtes de diagnostic sur chacun des trois outils
- Corréler logs et métriques pour diagnostiquer un incident
- Argumenter un choix technologique selon un contexte donné

---

## 📖 Contexte — L'entreprise ShopFlow

**ShopFlow** est un e-commerce B2C avec 50 microservices en production.  
Depuis 2019, leur stack de logs est basée sur **ELK** (Elasticsearch + Kibana).

En novembre 2024, un incident survient : le **provider de paiement Stripe tombe en panne** pendant 9 minutes (08h34 → 08h43). 17 timeouts de paiement surviennent, pour un montant total bloqué de 4 726 €.

L'équipe SRE doit diagnostiquer l'incident **d'abord avec ELK**, puis **avec Loki**, avant de découvrir comment **Prometheus** aurait permis de le détecter en temps réel.

> Les logs de l'incident sont dans le fichier `logs/shopflow.log` (format JSON Lines).  
> Votre mission : retrouver les mêmes informations dans les trois outils, puis comparer leurs approches.

---

## 🗂️ Structure du dépôt

```
tp-elk-loki/
├── docker-compose.yml          ← Stack complète (ELK + Loki + Prometheus)
├── logs/
│   └── shopflow.log            ← Logs de l'incident (JSON Lines)
├── elk/
│   └── inject_logs.py          ← Script d'injection dans Elasticsearch
├── loki/
│   ├── loki-config.yml         ← Configuration Loki
│   ├── promtail-config.yml     ← Configuration Promtail
│   └── grafana-datasources.yml ← Provisioning Grafana (Loki)
├── grafana/
│   └── provisioning/datasources/
│       └── datasources.yml     ← Provisioning Grafana (Loki + Prometheus)
├── prometheus/
│   ├── prometheus.yml          ← Configuration Prometheus
│   └── alert.rules.yml         ← Règles d'alertes ShopFlow
├── app-sample/
│   ├── app.js                  ← Application ShopFlow simulée (live)
│   ├── traffic.js              ← Générateur de trafic
│   └── package.json
└── trigger-incident.sh         ← Déclenche/arrête l'incident Prometheus
```

---

## ✅ Prérequis

Vérifiez votre environnement avant de démarrer :

```bash
docker --version          # ≥ 24.0
docker compose version    # ≥ 2.20
curl --version
```

> **RAM disponible recommandée : 4 Go minimum.**  
> Elasticsearch seul consomme ~600 Mo. Ne lancez pas ELK et Prometheus simultanément si vous êtes limité en RAM.

---

## 📋 Vue d'ensemble

| Partie | Outil | Durée | Ce que vous produisez |
|--------|-------|-------|-----------------------|
| **A** | ELK (Kibana + KQL) | 45 min | 6 requêtes KQL documentées |
| **B** | Loki (Grafana + LogQL) | 45 min | 7 requêtes LogQL + panel Grafana |
| **C** | Synthèse ELK vs Loki | 15 min | Tableau comparatif + recommandation |
| **D** | Prometheus (PromQL + Grafana) | 60 min | Dashboard 4 panels + corrélation Loki |

**Fil rouge :** le même incident ShopFlow (PAYMENT_TIMEOUT du 14/11/2024) est analysé sous trois angles. À la fin, vous saurez quel outil répond à quelle question.

---

---

# PARTIE A — ELK (Elasticsearch + Kibana)

---

## Étape A.1 — Démarrage de la stack ELK

Démarrez uniquement les services ELK avec le profil dédié :

```bash
docker compose --profile elk up -d
```

Attendez que les services soient prêts (environ 60 secondes) :

```bash
docker compose ps
```

Vous devez voir trois services avec le statut `running` :

```
shopflow-elasticsearch   running   0.0.0.0:9200->9200/tcp
shopflow-kibana          running   0.0.0.0:5601->5601/tcp
shopflow-elk-injector    exited (0)
```

> Le conteneur `elk-injector` se termine normalement après avoir injecté les logs. Le code de sortie `0` signifie succès.

Vérifiez que les logs ont bien été injectés :

```bash
docker logs shopflow-elk-injector
```

Vous devez voir :

```
✅ Elasticsearch prêt (status: yellow)
📁 Index shopflow-logs créé avec mapping
✅ 98 logs injectés dans Elasticsearch (0 erreurs)
🎉 Injection terminée. Index 'shopflow-logs' prêt dans Kibana.
```

---

## Étape A.2 — Découverte de Kibana

Ouvrez votre navigateur : **http://localhost:5601**

Au premier lancement, Kibana propose un assistant de démarrage. Ignorez-le en cliquant sur **"Explore on my own"**.

### Créer une Data View

Pour interroger vos données, Kibana a besoin d'une **Data View** qui pointe vers votre index.

1. Menu latéral → **Management** (icône engrenage) → **Stack Management**
2. Rubrique **Kibana** → **Data Views**
3. Cliquez **"Create data view"**
4. Remplissez :
   - **Name** : `ShopFlow Logs`
   - **Index pattern** : `shopflow-logs`
   - **Timestamp field** : `timestamp`
5. Cliquez **"Save data view to Kibana"**

### Ouvrir Discover

Menu latéral → **Discover** (icône boussole).

Sélectionnez la Data View **ShopFlow Logs** en haut à gauche si elle n'est pas déjà active.

> 💡 Vous voyez 0 résultat ? Vérifiez le sélecteur de plage temporelle en haut à droite — changez-le en **"Last 1 year"** ou saisissez manuellement la plage `Nov 14, 2024 @ 08:00 – 09:00`.

Vous devriez voir **98 documents**.

---

## Étape A.3 — Requêtes de diagnostic avec KQL

Kibana utilise **KQL (Kibana Query Language)** pour filtrer les logs. Saisissez chaque requête dans la barre de recherche en haut de Discover.

### Requête A-1 : Tous les logs d'erreur

```kql
level: "ERROR"
```

**Observez** : combien de logs d'erreur voyez-vous ? Notez le nombre dans votre cahier.

---

### Requête A-2 : Erreurs du service payment uniquement

```kql
level: "ERROR" AND service: "payment-service"
```

**Observez** : tous les timeouts sont-ils sur le même service ? Quel est le champ `error_code` le plus fréquent ?

---

### Requête A-3 : Filtrer sur le code d'erreur PAYMENT_TIMEOUT

```kql
error_code: "PAYMENT_TIMEOUT"
```

**Observez** : à quelle heure s'est produit le premier timeout ? Le dernier ?  
(Triez par timestamp croissant en cliquant sur la colonne `timestamp`.)

---

### Requête A-4 : Combien d'utilisateurs impactés ?

```kql
error_code: "PAYMENT_TIMEOUT"
```

Dans la colonne de gauche, cliquez sur le champ **`user_id`** → **"Visualize"**.

Kibana affiche un graphique en barres des utilisateurs les plus touchés.

**Observez** : chaque utilisateur a-t-il subi exactement 1 timeout, ou certains en ont-ils eu plusieurs ?

---

### Requête A-5 : Montant total bloqué

Nous allons utiliser le tableau **Lens** pour agréger le montant total des commandes bloquées.

1. Menu latéral → **Visualize Library** → **Create visualization** → **Lens**
2. Sélectionnez la Data View **ShopFlow Logs**
3. Filtrez d'abord : en haut, saisissez `error_code: "PAYMENT_TIMEOUT"`
4. Dans le panneau central, choisissez le type **Metric**
5. Faites glisser le champ **`amount`** dans la zone "Value"
6. Changez la fonction d'agrégation en **Sum**

**Résultat attendu** : notez le montant total (en €) bloqué pendant l'incident.

---

### Requête A-6 : Trouver la fin de l'incident

```kql
service: "payment-service" AND message: "rétablie"
```

**Observez** : à quelle heure exacte le service de paiement a-t-il été rétabli ?

---

## Étape A.4 — Bilan ELK

Avant de passer à Loki, notez vos impressions dans le tableau suivant (vous le complèterez après la Partie B) :

| Critère | ELK | Loki |
|---|---|---|
| Démarrage (temps, commandes) | | |
| Création Data View / index | | |
| Syntaxe de requête | | |
| Recherche full-text possible ? | | |
| Facilité de navigation | | |
| Ressources consommées (RAM) | | |

---

## Étape A.5 — Arrêt de la stack ELK

```bash
docker compose --profile elk down
```

> **Important** : arrêtez ELK avant de démarrer Loki si vous disposez de moins de 4 Go de RAM.

---

---

# PARTIE B — Loki + Grafana

---

## Étape B.1 — Démarrage de la stack Loki

```bash
docker compose --profile loki up -d
```

Attendez que les services soient prêts (environ 30 secondes) :

```bash
docker compose ps
```

Vous devez voir trois services `running` :

```
shopflow-loki       running   0.0.0.0:3100->3100/tcp
shopflow-promtail   running
shopflow-grafana    running   0.0.0.0:3000->3000/tcp
```

Vérifiez que Promtail a bien ingéré les logs :

```bash
docker logs shopflow-promtail 2>&1 | grep -i "read\|sent\|level=info"
```

Vérifiez que Loki est prêt :

```bash
curl -s http://localhost:3100/ready
```

Réponse attendue : `ready`

---

## Étape B.2 — Découverte de Grafana + Loki

Ouvrez votre navigateur : **http://localhost:3000**

> Grafana est configuré en accès anonyme pour ce TP — pas de login requis.

### Accéder à l'explorateur Loki

Menu latéral → **Explore** (icône boussole).

En haut à gauche, vérifiez que la source de données est bien **Loki**.

Changez la plage temporelle : en haut à droite, saisissez **"Last 5 years"** ou la plage `2024-11-14 08:00 – 2024-11-14 09:00`.

---

## Étape B.3 — Requêtes de diagnostic avec LogQL

Loki utilise **LogQL** pour interroger les logs. Les requêtes s'écrivent dans le champ **"Log browser"** de l'explorateur Grafana.

### Structure d'une requête LogQL

```
{sélecteur_de_labels} | filtre_de_contenu
```

- `{sélecteur_de_labels}` : sélectionne les **streams** (obligatoire)
- `| filtre_de_contenu` : filtre sur le texte ou les champs JSON (optionnel)

---

### Requête B-1 : Tous les logs ingérés

```logql
{job="shopflow"}
```

**Observez** : les logs apparaissent dans le panneau inférieur. Cliquez sur une ligne pour l'expandre et voir les labels extraits par Promtail.

---

### Requête B-2 : Tous les logs d'erreur

```logql
{job="shopflow", level="ERROR"}
```

> Contrairement à KQL, vous filtrez ici **par label** — Loki a indexé `level` comme label lors de l'ingestion via Promtail.

**Observez** : même résultat qu'avec ELK ? Notez le nombre de lignes.

---

### Requête B-3 : Erreurs du payment-service uniquement

```logql
{job="shopflow", level="ERROR", service="payment-service"}
```

**Observez** : Loki filtre instantanément par combinaison de labels — pas de parsing nécessaire.

---

### Requête B-4 : Filtrer sur le code PAYMENT_TIMEOUT

```logql
{job="shopflow", error_code="PAYMENT_TIMEOUT"}
```

**Observez** : `error_code` est aussi un label extrait par Promtail. La recherche est directe.

---

### Requête B-5 : Recherche textuelle dans le message

```logql
{job="shopflow"} |= "rétablie"
```

> `|=` signifie "contient". Loki effectue une recherche **dans le contenu brut** du log (pas un index full-text comme Elasticsearch — c'est un grep distribué).

**Observez** : retrouvez-vous la ligne "Connexion au provider de paiement rétablie" ?

---

### Requête B-6 : Taux d'erreurs dans le temps (métrique LogQL)

Passez en mode **"Metrics"** dans l'explorateur (bouton en haut à droite de la zone de requête) :

```logql
sum by (service) (count_over_time({job="shopflow", level="ERROR"}[5m]))
```

**Observez** : le graphique montre le nombre d'erreurs par fenêtre de 5 minutes. Le pic d'incident est-il visible ?

---

### Requête B-7 : Parser les champs JSON pour filtrer sur la latence

```logql
{job="shopflow", service="payment-service"}
  | json
  | latency_ms > 5000
```

> `| json` demande à Loki de parser chaque ligne comme JSON pour accéder aux champs non indexés comme `latency_ms`.

**Observez** : toutes les requêtes payment en timeout ont-elles bien `latency_ms > 5000` ?

---

## Étape B.4 — Comparer la syntaxe ELK / Loki

Complétez ce tableau de correspondance :

| Besoin | KQL (Kibana) | LogQL (Loki) |
|---|---|---|
| Tous les logs d'un service | `service: "payment-service"` | `{service="payment-service"}` |
| Filtrer par niveau | `level: "ERROR"` | `{level="ERROR"}` |
| Recherche dans le message | `message: "timeout"` | `{job="shopflow"} \|= "timeout"` |
| Combiner deux filtres | `A AND B` | `{label1="v1", label2="v2"}` |
| Taux dans le temps | Lens → agrégation | `count_over_time(...[5m])` |
| Parser les champs JSON | Automatique (mapping) | `\| json` (à la demande) |

---

## Étape B.5 — Bilan Loki

Complétez maintenant la colonne **Loki** du tableau que vous avez commencé en Étape A.4.

---

## Étape B.6 — Arrêt de la stack Loki

```bash
docker compose --profile loki down
```

---

---

# PARTIE C — Synthèse comparative ELK vs Loki

---

## Exercice C.1 — Reconstruction de l'incident

En utilisant les données que vous avez collectées dans les parties A et B, complétez ce rapport d'incident :

```markdown
## Rapport d'incident — ShopFlow — 14/11/2024

**Heure de début** : ___________
**Heure de fin**   : ___________
**Durée totale**   : ___________

**Service impacté** : ___________
**Provider externe** : ___________
**Code d'erreur**   : ___________

**Nombre de commandes échouées** : ___________
**Montant total bloqué**         : ___________€

**Cause identifiée** :
...

**Preuve (requête utilisée)** :
...
```

---

## Exercice C.2 — Décision de migration

L'équipe SRE de ShopFlow doit maintenant décider : **migrer vers Loki ou rester sur ELK ?**

Voici leur contexte :
- Stack actuelle : Docker Compose sur VM (pas de Kubernetes)
- Volume de logs : 200 Go/jour
- Budget infra actuel pour les logs : 1 800 €/mois
- Besoins : diagnostic d'incidents, pas de compliance ni d'audit full-text
- Équipe : 3 SREs déjà formés sur Grafana (pour Prometheus)

**Question** : rédigez en 5 à 10 lignes une recommandation motivée à destination du CTO de ShopFlow.  
Appuyez-vous sur les observations que vous avez faites pendant le TP.

---

## Exercice C.3 — Questions de compréhension

Répondez brièvement (2-3 lignes chacune) :

**1.** Pourquoi Elasticsearch consomme-t-il plus de RAM que Loki pour le même volume de logs ?

**2.** Dans LogQL, quelle est la différence entre `{level="ERROR"}` et `|= "ERROR"` ?

**3.** Citez un cas d'usage où vous choisiriez ELK plutôt que Loki, et justifiez.

**4.** Qu'est-ce qu'un **stream** dans Loki ? Pourquoi faut-il éviter les labels à haute cardinalité ?

---

---

# PARTIE D — Prometheus : métriques en temps réel

---

## Contexte

ELK et Loki vous ont permis d'analyser *ce qui s'est passé* dans les logs de l'incident du 14/11/2024. Mais ces logs étaient **statiques** — vous travailliez après coup.

Prometheus apporte une dimension différente : les **métriques en temps réel**. L'application `app-sample` simule ShopFlow en live et peut reproduire le même incident (PAYMENT_TIMEOUT) à la demande. Vous allez voir comment Prometheus aurait permis de détecter ce type d'incident **pendant qu'il se produit**, et comment corréler métriques et logs dans Grafana.

> **Transition pédagogique :** ELK et Loki répondent à *"Que s'est-il passé ?"* — Prometheus répond à *"Est-ce qu'il se passe quelque chose en ce moment ?"*

---

## Étape D.0 — Démarrage de l'infrastructure Prometheus

```bash
# Lancer Loki + Prometheus (Grafana redémarre avec les 2 datasources)
docker compose --profile loki --profile prometheus up -d

# Attendre 15 secondes
sleep 15

# Vérifier
docker compose ps
```

Vous devez voir ces services **Up** :

```
shopflow-loki                running   :3100
shopflow-promtail            running
shopflow-app-sample          running   :8080
shopflow-traffic-generator   running
shopflow-prometheus          running   :9090
shopflow-node-exporter       running   :9100
shopflow-grafana             running   :3000
```

Accès :

| Service | URL |
|---------|-----|
| Grafana (Loki + Prometheus) | http://localhost:3000 |
| Prometheus UI | http://localhost:9090 |
| app-sample /metrics | http://localhost:8080/metrics |

> Grafana s'est relancé avec deux datasources provisionnées automatiquement : **Loki** et **Prometheus**. Vous pouvez basculer entre les deux dans Explore.

---

## Étape D.1 — Explorer l'endpoint /metrics (10 min)

Contrairement à ELK (push via injecteur) et Loki (push via Promtail), Prometheus **va chercher** les métriques lui-même toutes les 15 secondes — c'est le **modèle pull**.

```bash
# Voir toutes les métriques exposées par app-sample
curl -s http://localhost:8080/metrics

# Identifier les types de métriques
curl -s http://localhost:8080/metrics | grep "^# TYPE"
```

**📝 Identifiez les types de métriques dans la sortie et notez un exemple de chacun :**

| Type | Nom de la métrique | Ce qu'elle mesure |
|------|--------------------|-------------------|
| counter | | |
| gauge | | |
| histogram | | |

Vérifiez que Prometheus scrape bien toutes les cibles :

Ouvrir **http://localhost:9090/targets** → toutes les cibles doivent être en vert (**UP**).

**Observez** : combien de cibles Prometheus surveille-t-il ? Lesquelles ?

---

## Étape D.2 — PromQL : méthode RED (20 min)

Ouvrir **http://localhost:9090** → onglet **Graph**. Régler la fenêtre temporelle sur **15 minutes** en haut à droite.

### Rate — débit de requêtes

**Requête 1 — Taux total de requêtes par seconde**
```promql
sum(rate(http_requests_total{job="app-sample"}[5m]))
```
> Valeur attendue : 2–5 req/s selon votre machine

**Requête 2 — Débit par endpoint**
```promql
sum by (endpoint) (rate(http_requests_total{job="app-sample"}[5m]))
```
> Quelle route reçoit le plus de trafic ?

**📝 Notez vos baselines (conditions normales) :**

| Métrique | Valeur observée |
|----------|----------------|
| Débit total (req/s) | |
| Débit /api/checkout (req/s) | |

### Errors — taux d'erreur

**Requête 3 — Taux d'erreur 5xx en pourcentage**
```promql
100 * sum(rate(http_requests_total{job="app-sample", status=~"5.."}[5m]))
    / sum(rate(http_requests_total{job="app-sample"}[5m]))
```
> En conditions normales : < 0.5%

**📝 Notez :** taux d'erreur actuel : _____%

### Duration — latence

**Requête 4 — Latence P50 / P95 / P99 sur le checkout**

```promql
histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket{job="app-sample", endpoint="/api/checkout"}[5m])) by (le))
```
```promql
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{job="app-sample", endpoint="/api/checkout"}[5m])) by (le))
```
```promql
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{job="app-sample", endpoint="/api/checkout"}[5m])) by (le))
```

**📝 Notez vos baselines de latence (conditions normales) :**

| Percentile | Valeur normale |
|------------|---------------|
| P50 | |
| P95 | |
| P99 | |

> Ces baselines sont importantes : vous vous en servirez pour mesurer l'écart pendant l'incident.

---

## Étape D.3 — Construire le dashboard Grafana (20 min)

Ouvrir **http://localhost:3000** → **Dashboards → New → New dashboard**

Construire 4 panels, un par un. Pour chaque panel : **Add visualization → sélectionner la datasource Prometheus**.

---

**Panel 1 — Taux d'erreur (type : Stat)**

Titre : `Taux d'erreur 5xx (%)`

```promql
(100 * sum(rate(http_requests_total{job="app-sample", status=~"5.."}[5m]))
    / sum(rate(http_requests_total{job="app-sample"}[5m]))) or vector(0)
```

Configuration :
- Unit : `Percent (0-100)`
- Thresholds : vert=0, jaune à 1%, rouge à 5%
- Color mode : **Background**

> Ce panel doit "sauter aux yeux" quand l'incident se déclenche.

---

**Panel 2 — Latence P50 / P95 / P99 (type : Time series)**

Titre : `Latence checkout — P50 / P95 / P99`

Requête A (Legend : `P50`) :
```promql
histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket{job="app-sample", endpoint="/api/checkout"}[5m])) by (le))
```
Requête B (Legend : `P95`) :
```promql
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{job="app-sample", endpoint="/api/checkout"}[5m])) by (le))
```
Requête C (Legend : `P99`) :
```promql
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{job="app-sample", endpoint="/api/checkout"}[5m])) by (le))
```

Configuration : Unit : `seconds (s)`

---

**Panel 3 — Débit par endpoint (type : Time series)**

Titre : `Débit (req/s) par endpoint`

```promql
sum by (endpoint) (rate(http_requests_total{job="app-sample"}[5m]))
```

Legend : `{{endpoint}}` — Configuration : Unit : `requests/sec`

---

**Panel 4 — Saturation DB (type : Gauge)**

Titre : `DB Pool — saturation`

```promql
db_connections_active{job="app-sample"} / 50
```

Configuration : Unit : `percent (0.0-1.0)`, Min=0, Max=1, Thresholds : vert=0, jaune=0.7, rouge=0.9

---

Sauvegarder : **Save dashboard** → Nom : `ShopFlow — Métriques Prometheus`

---

## Étape D.4 — Incident en direct + corrélation Loki (10 min)

### Déclencher l'incident

```bash
./trigger-incident.sh start
```

**Observez votre dashboard pendant 2 minutes** sans intervenir.

**📝 Remplir pendant l'incident :**

| Métrique | Valeur baseline | Valeur incident | Écart |
|----------|----------------|-----------------|-------|
| Taux d'erreur (%) | | | |
| Latence P95 checkout | | | |
| Latence P99 checkout | | | |
| DB connexions actives | | | |
| Premier panel à réagir | — | — | |

### Corréler avec les logs Loki

Sans quitter Grafana, ouvrir **Explore** → changer la datasource en **Loki** :

```logql
{job="app-sample"} | json | level="ERROR"
```

**📝 Répondez :**

1. Quel message d'erreur et quel `error_code` apparaissent dans les logs Loki ?
2. Ce message correspond-il à ce que vous observez dans les métriques Prometheus ?
3. Avec ELK ou Loki seuls sur des logs statiques (Parties A et B), auriez-vous pu détecter cet incident *pendant qu'il se produisait* ?

### Arrêter l'incident

```bash
./trigger-incident.sh stop
```

> Les métriques Prometheus reviennent à la baseline en ~5 minutes (délai de la fenêtre `rate[5m]`).

---

## Étape D.5 — Bilan Prometheus

**📝 Complétez le tableau comparatif final :**

| Critère | ELK | Loki | Prometheus |
|---------|-----|------|------------|
| Type de données | Logs | Logs | Métriques |
| Données historiques (post-mortem) | ✅ | ✅ | ⚠️ (rétention limitée) |
| Détection temps réel | ❌ | ❌ | ✅ |
| Montant financier bloqué | ✅ | ✅ | ❌ |
| Root cause dans les logs | ✅ | ✅ | ❌ seul |
| Alertes automatiques | ⚠️ | ⚠️ | ✅ natif |
| Consommation RAM | ~1.5 Go | ~200 Mo | ~100 Mo |
| Corrélation avec l'autre outil | — | — | ✅ via Grafana |

---

---

# PARTIE E — Livrable final

---

## Rapport d'incident complet

```
RAPPORT D'INCIDENT SHOPFLOW
────────────────────────────────────────────────────────────
Nom : _______________

┌─ ANALYSE ELK (Partie A) ─────────────────────────────────┐
│ Nombre total d'erreurs        : ____                      │
│ Erreurs PAYMENT_TIMEOUT       : ____                      │
│ Montant financier bloqué      : ______ €                  │
│ Début incident                : 08h____                   │
│ Fin incident                  : 08h____                   │
│ Durée                         : ____ minutes              │
└───────────────────────────────────────────────────────────┘

┌─ ANALYSE LOKI (Partie B) ────────────────────────────────┐
│ Requête LogQL utilisée :                                  │
│   {job="shopflow"} | json | ________________________      │
│ Taux d'erreur au pic (req/min) : ____                     │
│ Latence moyenne des timeouts   : ______ ms                │
└───────────────────────────────────────────────────────────┘

┌─ MÉTRIQUES PROMETHEUS (Partie D) ────────────────────────┐
│ Taux d'erreur max détecté      : _____%                   │
│ Latence P99 max                : ______ ms                │
│ Connexions DB au pic           : ____ / 50                │
│ Délai détection visuelle       : ______ secondes          │
└───────────────────────────────────────────────────────────┘

┌─ CONCLUSION ──────────────────────────────────────────────┐
│ Quel outil aurait détecté l'incident le plus tôt ?        │
│ Quel outil donne le plus d'information sur la cause ?     │
│ Quel outil utiliseriez-vous pour ne plus rater ce type    │
│ d'incident à l'avenir ?                                   │
│                                                           │
│ _________________________________________________________ │
│ _________________________________________________________ │
│ _________________________________________________________ │
└───────────────────────────────────────────────────────────┘
```

---

# 🔧 Troubleshooting

| Problème | Cause probable | Solution |
|---|---|---|
| `shopflow-elk-injector` exited avec code 1 | Elasticsearch pas encore prêt | `docker compose --profile elk restart elk-injector` |
| Kibana affiche 0 résultats | Plage temporelle incorrecte | Changer en "Last 1 year" ou plage manuelle Nov 2024 |
| Loki renvoie "no results" | Promtail n'a pas encore ingéré | Attendre 30s, vérifier `docker logs shopflow-promtail` |
| Grafana ne trouve pas Loki | Loki pas encore prêt | Attendre que `curl localhost:3100/ready` renvoie `ready` |
| Prometheus targets en rouge | Service pas démarré | `docker compose --profile prometheus up -d` |
| `/metrics` renvoie connexion refusée | app-sample pas encore prêt | Attendre le healthcheck : `docker compose ps` |
| Métriques Prometheus à 0 après incident stop | Fenêtre rate[5m] en cours | Attendre ~5 minutes, les valeurs reviennent normales |
| Port 5601 ou 3000 déjà utilisé | Conflit de port | Modifier le port hôte dans `docker-compose.yml` |
| `curl: command not found` | curl absent | `apt-get install curl` ou utiliser le navigateur |

---

# 📚 Ressources

- Documentation Loki : https://grafana.com/docs/loki/latest/
- Référence LogQL : https://grafana.com/docs/loki/latest/query/
- Documentation Elasticsearch : https://www.elastic.co/docs
- Référence KQL : https://www.elastic.co/guide/en/kibana/current/kuery-query.html
- Documentation Prometheus : https://prometheus.io/docs/introduction/overview/
- Référence PromQL : https://prometheus.io/docs/prometheus/latest/querying/basics/

---

*BOTE848 — EPSI Mastère SIN/EISI — 2025-2026*
