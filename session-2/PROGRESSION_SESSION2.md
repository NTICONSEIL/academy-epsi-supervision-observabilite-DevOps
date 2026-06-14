# 🟩 SÉANCE 2 - PROGRESSION DÉTAILLÉE
## Les Outils : Loki (Logs) & Prometheus (Métriques) (4h)

**Objectif** : Passer de la théorie aux outils concrets. Configurer et utiliser Loki et Prometheus.

---

## 📋 Vue d'ensemble

| Bloc | Durée | Sujet | Format | Livrable |
|------|-------|-------|--------|----------|
| 1 | 1h | Loki : Architecture & Concepts | 📖 Théorie | Compréhension |
| 2 | 1h | **TP Loki : Config & LogQL** | 💻 Hands-on | Dashboard Loki |
| 3 | 1h | Prometheus : Architecture & Types | 📖 Théorie | Compréhension |
| 4 | 1h | **TP Prometheus : Config & PromQL** | 💻 Hands-on | Alertes configurées |

---

## ⏱️ DÉTAIL TIMING & CONTENUS

### **BLOC 1 : THÉORIE LOGS - LOKI (0:00 - 1:00)**

#### Contenus clés

**1.1 Logs vs Traces vs Métriques (15 min)**

Tableau récapitulatif :

```
LOGS (Pilier 1)
├─ Granularité : TRÈS FINE (événement individual)
├─ Volume : TRÈS ÉLEVÉ (millions/jour)
├─ Structure : Texte (ou JSON structuré)
├─ Storage : Très coûteux si non indexé
├─ Timing : Temps réel
└─ Cas : Débogage, audit, "what happened"

MÉTRIQUES (Pilier 2)
├─ Granularité : AGRÉGÉE (sommes, moyennes)
├─ Volume : BAS (centaines/sec)
├─ Structure : Nombre + labels + timestamp
├─ Storage : Efficace, optimisé séries temporelles
├─ Timing : Régulier (15s, 1min, 5min)
└─ Cas : Tendances, alertes, "how much"

TRACES (Pilier 3)
├─ Granularité : HIÉRARCHIQUE (spans parents-enfants)
├─ Volume : MOYEN (dépend sampling)
├─ Structure : Trace ID, spans, durées
├─ Storage : Coûteux, typically sampled
├─ Timing : Complet requête (ms à sec)
└─ Cas : Causalité, "who called who"
```

Animer :
- Comparer avec analogies : logs = journal détaillé, métriques = graphiques, traces = itinéraire
- Question : "Pour débugging error spécifique, lequel choisiriez-vous ?"
- Réponse : LOGS toujours pour détails

---

**1.2 Architecture Logs & Collecte (20 min)**

Diagram flux logs :

```
Applications (Python, Node, Java)
    ↓ logs (text/JSON)
Collecteurs (Filebeat, Fluent Bit, Logstash, Promtail)
    ↓ envoi batch
Agrégateurs (Fluentd, Logstash)
    ↓ parsing, transformation
Stockage (Elasticsearch, Loki, S3)
    ↓ indexation
Query Engine (Kibana, Loki UI, etc.)
    ↓ affichage
Utilisateurs (Devs, SREs, Ops)
```

Concepts clés :

```
AGENTS COLLECTEURS
├─ Filebeat : léger, logs fichiers
├─ Fluent Bit : léger, flexible, logs+métriques
├─ Logstash : lourd, transformation puissante
└─ Promtail : agent Loki (ce qu'on utilisera)

GARANTIES LIVRAISON
├─ At-least-once : risque duplicates
├─ Exactly-once : difficile à garantir (état)
├─ At-most-once : risque pertes

BUFFERING & RETRY
├─ Queue locale (si backend down)
├─ Retry avec backoff exponentiel
├─ Batch envoi (efficacité)
```

Animer :
- Montrer diagramme en grand
- Décrire chaque composant
- Question : "Que se passe-t-il si collecteur crash ?"
- Réponse : dépend architecture (peut perdre logs en mémoire)

---

**1.3 Loki : Approche Différente (20 min)**

Pourquoi Loki vs ELK Stack ?

```
ELK (Elasticsearch, Logstash, Kibana)
├─ AVANTAGES
│  ├─ Full-text search puissante
│  ├─ Ecosystem riche
│  └─ Large communauté
├─ INCONVÉNIENTS
│  ├─ Coûteux (indexation complète)
│  ├─ Complexe (setup, opérations)
│  ├─ Ressources : 10GB+ heap pour logs importants
│  └─ Apprentissage raide

LOKI (Grafana Loki)
├─ AVANTAGES
│  ├─ Très efficace (index par labels only)
│  ├─ Léger (compatible Kubernetes)
│  ├─ Cheap storage (compressé)
│  ├─ Intégration Grafana native
│  └─ Open source, simpler
├─ INCONVÉNIENTS
│  ├─ Full-text search limitations
│  ├─ Moins mature que ELK
│  ├─ Communauté plus petite
│  └─ Features moins nombreuses
```

Architecture Loki :

```
Promtail (agents)
    ↓ push logs
Distributor (reçoit, valide, labels)
    ↓ routes par tenant
Ingester (RAM) → Chunk storage (disque/S3)
    ↓ flux queries
Querier (récupère chunks)
    ↓ résultats
Loki UI ou Grafana
```

Label strategy (CLÉS pour Loki) :

```
Labels = critères indexation UNIQUE
├─ job : "api", "nginx", "app"
├─ instance : "api-1", "api-2"
├─ env : "prod", "staging"
├─ service : "auth", "payment"
└─ region : "eu-west", "us-east"

⚠️ CARDINALITY WARNING
└─ Trop labels = trop chunks = inefficace
└─ Règle : < 10 labels, < 1000 valeurs uniques par label
└─ BAD : label="user_id" (trop distinct)
└─ GOOD : label="service" (réutilisable)
```

---

**1.4 LogQL : Query Language (5 min)**

Intro rapide syntaxe :

```logql
# Syntax basic
{label="value"} |= "pattern"

# Examples
{service="api"} |= "ERROR"     # All ERRORs from api
{env="prod"} != "INFO"          # Non-info logs prod
{service="api"} |= "timeout"    # Contains "timeout"
{job="nginx"} | json            # Parse JSON
{service="api"} | latency > 1s  # Filter parsed field

# Aggregations
rate({service="api"} |= "ERROR" [5m])  # Error rate 5min
sum(rate({service="api"} [1m]))        # Total rate
```

Animer :
- Montrer 3-4 exemples simples
- Prévoir : "Détail dans TP"
- Question teaser : "Comment chercheriez-vous erreurs API de ce matin ?"

---

### **BLOC 2 : TP LOKI (1:00 - 2:00)**

#### Objectifs TP

Participant peut :
- Déployer Loki + Promtail
- Configurer labels
- Écrire requêtes LogQL
- Visualiser dans Grafana

#### Déroulement

**Phase 1 : Déploiement Loki (15 min)**

```bash
# 1. Vérifier Docker Compose
docker-compose --version

# 2. Déployer stack
cd ~/BOTE848
docker-compose up -d loki promtail grafana

# 3. Attendre 10 secondes
sleep 10

# 4. Vérifier services
docker-compose ps
# Tous devraient être "Up"

# 5. Test Loki accessible
curl -s http://localhost:3100/ready
# Output : "ready" ✓

# 6. Test Promtail logs
docker logs BOTE848_promtail_1 | grep -i "connected"
```

Animer :
- Afficher sur écran ces commandes
- Faire ensemble par 2-3
- Aider les bloqués
- "Si bloqué au démarrage, check docker-compose.yml n'a pas erreurs syntaxe"

---

**Phase 2 : Configuration Promtail (15 min)**

Fichier config fourni :

```yaml
clients:
  - url: http://localhost:3100/loki/api/v1/push

positions:
  filename: /tmp/positions.yaml

scrape_configs:
  - job_name: app-sample
    static_configs:
      - targets: [localhost]
        labels:
          job: app-sample
          env: development
          service: api
          instance: "1"
    pipeline_stages:
      - json:
          expressions:
            timestamp: time
            level: level
            service: service
            message: message
```

Exercice :
> "Modifier config Promtail pour ajouter 2 services :  
> - app-sample (job: app-sample)  
> - nginx (job: nginx)"

Étapes :

```yaml
# Dupliquer scrape_config
scrape_configs:
  - job_name: app-sample
    # ... (keep existing)
  
  - job_name: nginx        # ← AJOUTER
    static_configs:
      - targets: [localhost]
        labels:
          job: nginx
          env: production
          service: web
          instance: "1"
```

Animer :
- Montrer template
- Demander : "Quels labels pour nginx ?"
- Ensemble modifier fichier
- Redémarrer Promtail : `docker-compose restart promtail`
- Vérifier logs arrivent

---

**Phase 3 : Requêtes LogQL (20 min)**

Exercices progressifs :

```bash
# ACCÈS Loki UI
# URL : http://localhost:3100

# REQUÊTE 1 : Tous logs du service 'api'
{service="api"}
# → Voir tous logs du service API

# REQUÊTE 2 : Errors seulement
{service="api"} | level="ERROR"
# → Voir 3 errors

# REQUÊTE 3 : Logs contenant "timeout"
{service="api"} |= "timeout"
# → Résultats

# REQUÊTE 4 : Rate d'erreurs (par minute)
rate({service="api"} | level="ERROR" [1m])
# → Graphique : ↓ rate errors

# REQUÊTE 5 : Logs nginx (access logs)
{service="web"} | status >= "400"
# → Erreurs HTTP 400+
```

À faire :

```
Participant écrit :
- 1 requête simples labels
- 1 requête avec filtre contenu
- 1 requête avec aggregation rate
- 1 requête personnalisée (sujet choix)

Résultats stockés (screenshots)
```

Animer :
- Live coding : afficher Loki UI
- Exécuter requête ensemble
- Expliquer chaque partie
- Laisser libre explorer après
- "Essayez combiner 2 labels différents"

---

**Phase 4 : Visualiser dans Grafana (10 min)**

```bash
# 1. Ouvrir Grafana
# URL : http://localhost:3000
# Login : admin / admin

# 2. Créer panel Loki
# New → Panel → Loki datasource
# Query : {service="api"} | level="ERROR"
# Legend : {{service}} - {{level}}

# 3. Type visualization
# Logs table (affiche ligne logs)
# Graph (affiche taux)

# 4. Sauvegarder
```

Livrables :
- [ ] Loki déployée et accessible
- [ ] Promtail configurée (2+ services)
- [ ] 5 requêtes LogQL exécutées
- [ ] 1 dashboard Grafana avec Loki panel
- [ ] Screenshots sauvegardées

---

### **BLOC 3 : THÉORIE MÉTRIQUES - PROMETHEUS (2:00 - 3:00)**

#### Contenus clés

**3.1 Types de Métriques (15 min)**

```
COUNTER (ne fait que monter)
├─ Exemple : total_requests_received = 1,000,000
├─ Reset : seulement à restart process
├─ Cas : events count, bytes processed
└─ Query PromQL : rate(http_requests_total[5m])

GAUGE (monte et descend)
├─ Exemple : current_memory_usage = 2048 MB
├─ Peut baisser : connection closed, cache clear
├─ Cas : CPU, memory, temperature
└─ Query PromQL : node_memory_MemFree_bytes

HISTOGRAM (distribution)
├─ Exemple : request_duration_seconds
│  ├─ bucket 0.1s : 150 requêtes
│  ├─ bucket 0.5s : 47 requêtes
│  ├─ bucket 1.0s : 3 requêtes
│  └─ sum : 200 total, mean = 0.15s
├─ Auto buckets pour quantiles
├─ Cas : latencies, sizes
└─ Query PromQL : histogram_quantile(0.95, rate(...))

SUMMARY (quantiles pré-calculés)
├─ Exemple : request_duration_seconds
│  ├─ quantile="0.5" : 0.15s (median)
│  ├─ quantile="0.9" : 0.5s
│  ├─ quantile="0.99" : 1.2s
├─ Pré-calculé par application
├─ Cas : latencies, custom metrics
└─ Query PromQL : rate(request_duration_seconds_sum[5m])
```

Animer :
- Comparer Types avec des exemples du domaine participants
- Question : "Request latency → quel type ?"
- Réponse : HISTOGRAM ou SUMMARY (pour quantiles)

---

**3.2 Prometheus Architecture (15 min)**

Diagram :

```
Applications (exposen /metrics)
    ↓ HTTP scrape
Prometheus (pull model)
    ├─ Time series database local
    ├─ Storage : 2 semaines par default
    └─ Alert rules evaluation
    ↓
AlertManager
    └─ Send notifications
    ↓
Users + Dashboard (Grafana, Prometheus UI)
```

Concepts clés :

```
PULL MODEL
├─ Prometheus scrape cibles à intervalle régulier
├─ Cible expose /metrics endpoint
├─ Format texte Prometheus
├─ Avantage : cible contrôle ce qui expose
├─ Avantage : moins de configuration
└─ Avantage : NAT-friendly (pas de push out)

TARGETS
├─ Statiques : file YAML
├─ Dynamiques : service discovery (Kubernetes, Consul)
└─ Scrape interval : default 15s

TIME SERIES DATABASE
├─ Stockage local (pas cloud)
├─ Compression : ~1-2 bytes per sample
├─ Retention : 15 jours default (configurable)
└─ Queryable immédiatement après scrape

REMOTE STORAGE (optionnel)
├─ Pour retention > 2 semaines
├─ Solutions : S3, GCS, Thanos, etc.
```

Animer :
- Schéma sur grand écran
- Montrer comment /metrics endpoint retourne format texte
- "Prometheus scrape régulièrement → construit séries temporelles"

---

**3.3 Instrumentation Applications (20 min)**

Comment ajouter metrics à votre app :

Exemple Python :

```python
from prometheus_client import Counter, Gauge, Histogram, start_http_server

# Déclarer metrics
requests_total = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint']
)

current_connections = Gauge(
    'db_connections_active',
    'Active database connections'
)

request_duration = Histogram(
    'http_request_duration_seconds',
    'HTTP request latency',
    ['method'],
    buckets=[0.1, 0.5, 1.0, 2.0, 5.0]
)

# Lancer serveur /metrics
start_http_server(8000)

# Dans votre app
requests_total.labels(method='GET', endpoint='/api/users').inc()
current_connections.set(5)

with request_duration.labels(method='GET').time():
    # requête lente
    time.sleep(0.5)
```

Exemple Node.js :

```javascript
const prometheus = require('prom-client');

const httpRequestDuration = new prometheus.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request latency',
  labelNames: ['method', 'status'],
  buckets: [0.1, 0.5, 1.0, 2.0]
});

app.use((req, res, next) => {
  const end = httpRequestDuration.startTimer({
    method: req.method,
    status: res.statusCode
  });
  res.on('finish', () => end());
  next();
});

app.get('/metrics', (req, res) => {
  res.set('Content-Type', prometheus.register.contentType);
  res.end(prometheus.register.metrics());
});
```

Animer :
- Montrer libs disponibles pour différents langages
- "3-4 lignes de code pour ajouter une metric"
- Question : "Comment instrumenteriez-vous votre app ?"

---

**3.4 PromQL : Query Language (10 min)**

Intro syntaxe :

```promql
# Instant vector (valeur instantanée)
node_memory_MemFree_bytes

# Avec labels
http_requests_total{method="GET"}

# Functions
rate(http_requests_total[5m])      # requêtes par second (5min)
increase(sold_total[1h])            # combien vendu dernière heure
avg(cpu_usage)                      # CPU moyen
quantile(0.95, request_duration)   # P95 latency

# Combinations
sum(rate(http_requests_total[5m]))  # Total requests/sec
avg by (method) (request_duration)  # Avg latency par méthode
```

Animer :
- Montrer 3-4 queries simples
- Prévoir "Détail dans TP"
- Teaser : "Dans TP, créerez dashboard avec ces queries"

---

### **BLOC 4 : TP PROMETHEUS (3:00 - 4:00)**

#### Objectifs TP

Participant peut :
- Configurer Prometheus targets
- Écrire requêtes PromQL
- Créer première alerte
- Visualiser dans Grafana

#### Déroulement

**Phase 1 : Configuration Prometheus (15 min)**

Fichier config fourni : `prometheus.yml`

```yaml
global:
  scrape_interval: 15s
  scrape_timeout: 10s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  
  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']
    metrics_path: '/metrics'

  - job_name: 'app-sample'
    static_configs:
      - targets: ['localhost:8000']
```

Exercice :
> "Ajouter 3ème target : 'kafka' sur localhost:9101"

Étapes :

```yaml
scrape_configs:
  # ... existing ...
  
  - job_name: 'kafka'     # ← AJOUTER
    static_configs:
      - targets: ['localhost:9101']
    metrics_path: '/metrics'
    scrape_interval: 30s
```

Animer :
- Montrer format YAML
- Ensemble modifier fichier
- Redémarrer Prometheus : `docker-compose restart prometheus`
- Vérifier targets : http://localhost:9090/targets
- "Cible 'kafka' devrait être green (UP)"

---

**Phase 2 : Requêtes PromQL (20 min)**

Accès UI Prometheus : http://localhost:9090

Exercices progressifs :

```promql
# QUERY 1 : CPU usage
node_cpu_seconds_total{mode="user"}

# QUERY 2 : CPU rate (change per second)
rate(node_cpu_seconds_total{mode="user"}[1m])

# QUERY 3 : Total CPU
sum(rate(node_cpu_seconds_total[1m]))

# QUERY 4 : Memory free
node_memory_MemFree_bytes / 1024 / 1024  # en MB

# QUERY 5 : Custom metric de app
http_requests_total{service="api"}

# QUERY 6 : Request rate per endpoint
rate(http_requests_total[5m]) by (endpoint)

# QUERY 7 : Error rate
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))
```

À faire :

```
Participant :
- Écrit 5 requêtes PromQL différentes
- Affiche graphs pour chacune
- Teste + documenter
- Observe trends/patterns
```

Animer :
- Live coding Prometheus UI
- Exécuter query
- Afficher graph
- "Voyez comme rate() montre trend"
- Encourager expérimentation

---

**Phase 3 : Alert Rules (15 min)**

Fichier `alert.rules.yml` fourni :

```yaml
groups:
  - name: node_alerts
    interval: 30s
    rules:
      - alert: HighCPU
        expr: rate(node_cpu_seconds_total{mode="user"}[5m]) > 0.8
        for: 2m
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          
      - alert: LowMemory
        expr: node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes < 0.2
        for: 5m
        annotations:
          summary: "Low memory on {{ $labels.instance }}"
```

Exercice :
> "Créer nouvelle alerte : 'HighErrorRate'  
> Si error rate > 1% durant 1 minute"

Étapes :

```yaml
- alert: HighErrorRate
  expr: sum(rate(http_requests_total{status=~"5.."}[1m])) / sum(rate(http_requests_total[1m])) > 0.01
  for: 1m
  annotations:
    summary: "High error rate: {{ $value }}"
```

Animer :
- Montrer format alerts
- Ensemble créer nouvelle règle
- Recharger Prometheus : `docker-compose exec prometheus kill -HUP 1`
- Vérifier Alerts tab : http://localhost:9090/alerts

---

**Phase 4 : Dashboard Grafana + Prometheus (10 min)**

```bash
# 1. Ouvrir Grafana
# URL : http://localhost:3000

# 2. Add Prometheus datasource (if not already)
# Configuration → Data Sources → Prometheus
# URL : http://prometheus:9090

# 3. Créer Panel
# New Panel → 
# Metrics : rate(node_cpu_seconds_total[1m])
# Type : Time Series

# 4. Ajouter 2ème query
# B: node_memory_MemFree_bytes

# 5. Legend, axis labels, etc.

# 6. Sauvegarder dashboard
```

Livrables :
- [ ] Prometheus configurée avec 3+ targets
- [ ] 5+ requêtes PromQL exécutées (screenshots)
- [ ] 1-2 alert rules créées
- [ ] 1 dashboard Grafana avec Prometheus panels
- [ ] Compréhension rate(), aggregations

---

## 📚 Matériel session 2 à préparer

### Slides
- [ ] slides-session2.pptx
- [ ] Schémas Loki, Prometheus architecture

### Infrastructure
- [ ] Docker Compose opérationnel (Loki, Prometheus, Grafana)
- [ ] App sample avec Prometheus metrics
- [ ] Config files (promtail, prometheus)

### Ressources
- [ ] LogQL cheat sheet (imprimé)
- [ ] PromQL cheat sheet (imprimé)
- [ ] Exercices LogQL/PromQL

---

## ⏰ Gestion du timing

```
Total 4h :
- Théorie Loki : 1h
- TP Loki : 1h (+5min buffer)
- Théorie Prometheus : 1h
- TP Prometheus : 1h (+5min buffer)
= 4h10 réaliste
```

**Si retard** :
- Sauter débats optionnels
- Montrer solutions TP plutôt que laisser découvrir seul
- Post-session : exercices supplémentaires online

---

## 🎯 Critères succès

✅ À la fin session 2 :
- [ ] Peuvent déployer et configurer Loki
- [ ] Écrivent requêtes LogQL progressivement complexes
- [ ] Comprennent 4 types de métriques Prometheus
- [ ] Déploient et configurent Prometheus
- [ ] Écrivent requêtes PromQL + visualisent dans Grafana
- [ ] Créent première alerte

---

*Version 1.0 - Document d'animation*

