# Analyse de Logs : jq, ELK, Loki et Prometheus

Pour comprendre l'analyse de logs, il faut d'abord lever une petite confusion : ces 4 outils ne jouent pas du tout dans la même catégorie.

- **jq** est un outil local en ligne de commande.
- **ELK** et **Loki** sont de véritables systèmes de gestion de logs centralisés.
- **Prometheus** n'est pas un outil de logs, mais un outil de **métriques** (qui s'associe souvent aux logs).

---

## 1. Les Notions Principales & Présentation des Outils

### jq : Le scalpel du JSON

jq est un utilitaire en ligne de commande (CLI) ultra-léger. Il ne stocke rien, ne centralise rien. Son unique but est de prendre un flux de texte au format JSON (comme un fichier de log d'application moderne), de le filtrer, de le transformer et de le formater de manière lisible.

**Idéal pour :** Analyser un fichier de log local sur sa machine ou directement sur un serveur en SSH.

---

### ELK (Elasticsearch, Logstash, Kibana) : Le poids lourd de la recherche

C'est la suite historique et la plus puissante. Logstash collecte et transforme les logs, Elasticsearch les stocke et les indexe entièrement, et Kibana permet de les visualiser.

**Le secret d'Elasticsearch :** Il fait de l'**indexation complète** (Full-Text). Chaque mot de chaque ligne de log est indexé. C'est un moteur de recherche (comme Google) pour vos logs.

**Inconvénient :** L'indexation lourde consomme énormément de RAM et d'espace disque.

---

### Grafana Loki : Le "Prometheus" des logs (léger et économique)

Créé par Grafana Labs, Loki prend le contre-pied d'ELK. Au lieu d'indexer tout le contenu des logs, Loki **n'indexe que les métadonnées (les labels)** (ex: `app="api"`, `env="prod"`). Le corps du log lui-même est compressé et stocké à bas coût (par exemple sur AWS S3).

**Avantage :** Consomme infiniment moins de ressources et de stockage qu'ELK.

**Inconvénient :** Si vous cherchez un mot-clé précis au milieu de milliards de lignes sans filtrer par labels, la recherche sera plus lente car Loki devra scanner le texte brut à la volée.

---

### Prometheus : Le gardien des métriques (L'intrus)

Prometheus **ne stocke pas de logs**. C'est une base de données temporelle (Time-Series) conçue pour stocker des **chiffres (des métriques)** : l'utilisation CPU (ex: 85%), le nombre de requêtes (ex: 200 req/s), ou le taux d'erreur.

**Le lien avec les logs :** On l'utilise en duo avec les logs. Par exemple, Prometheus vous alerte que le taux d'erreur de l'API a bondi (métrique), et vous basculez ensuite sur Loki pour voir le détail des messages d'erreur (logs) ayant les mêmes labels.

---

## 2. Exemple Comparatif Concret

Imaginons que notre application Web génère la ligne de log JSON suivante (une erreur 500 sur l'URL `/checkout`) :

```json
{
  "timestamp": "2026-06-14T22:00:00Z",
  "service": "payment-api",
  "level": "ERROR",
  "message": "Database connection timeout on /checkout",
  "duration_ms": 1500
}
```

### Le Tableau Comparatif

| Critère | jq | ELK (Elasticsearch) | Grafana Loki | Prometheus |
|---|---|---|---|---|
| **Nature de l'outil** | Commande locale | Système centralisé global | Système centralisé léger | Système de métriques (Pas de logs) |
| **Ce qu'il indexe** | Rien (analyse à la volée) | Tout le contenu JSON | Uniquement les labels (ex: `service="payment-api"`) | Uniquement les métriques numériques |
| **Coût de stockage** | Aucun | Élevé (gros index) | Très faible (fichiers compressés) | Faible (uniquement des chiffres) |
| **Langage de requête** | Syntaxe jq (ex: `.message`) | KQL (Kibana Query) ou Lucene | LogQL (très proche de PromQL) | PromQL |

---

### Comment ils traitent notre exemple ?

#### 1. Avec jq

Vous êtes sur votre terminal et vous voulez extraire uniquement les messages d'erreur du fichier `app.log`. Vous tapez :

```bash
cat app.log | jq 'select(.level == "ERROR") | .message'
```

**Résultat :** `"Database connection timeout on /checkout"`

C'est rapide, parfait pour la machine de dev, mais inutilisable si vous avez 50 serveurs en production.

---

#### 2. Avec ELK

L'agent (Logstash ou FluentBit) envoie le JSON à Elasticsearch. Elasticsearch analyse le JSON et indexe chaque mot.

Dans Kibana, vous tapez simplement dans la barre de recherche : `timeout` ou `checkout`.

**Résultat :** Instantané. Elasticsearch trouve le mot "timeout" n'importe où dans le message car tout est indexé dans son dictionnaire géant.

---

#### 3. Avec Grafana Loki

L'agent de Loki (Promtail ou Grafana Alloy) envoie le log en lui collant des étiquettes (labels) : `service="payment-api"` et `level="ERROR"`. Le message "Database connection timeout..." est compressé dans un bloc de texte brut.

Dans Grafana (en LogQL), pour trouver l'erreur, vous écrivez :

```logql
{service="payment-api", level="ERROR"} |= "timeout"
```

**Résultat :** Loki cible d'abord le flux du service de paiement (très rapide grâce au label indexé), puis il "lit" le texte à toute vitesse à la recherche du mot "timeout".

---

#### 4. Avec Prometheus

Prometheus ignore complètement le message "Database connection timeout on /checkout". En revanche, un exportateur de métriques va lire ce log et va incrémenter un compteur numérique pour Prometheus.

Dans Prometheus (en PromQL), vous analysez la métrique associée :

```promql
rate(http_requests_total{service="payment-api", status="500"}[5m])
```

**Résultat :** Un graphique qui affiche un pic (ex: 12 requêtes en erreur par seconde). Vous ne savez pas *pourquoi* (pas de texte), mais vous savez *quand* et *où* ça brûle.

---

## En résumé : Quel outil choisir ?

- **Utilisez jq** quand vous analysez des fichiers à la main sur votre poste.
- **Choisissez ELK** si vous avez le budget (infrastructure lourde) et que vous avez besoin de faire des recherches de texte ultra-complexes ou du business intelligence sur vos logs.
- **Choisissez Loki** si vous voulez une solution moderne, économique, hautement scalable, et que vous utilisez déjà Grafana et Kubernetes.
- **Utilisez Prometheus** en complément de Loki ou d'ELK pour avoir des tableaux de bord de performance (tableaux de bord graphiques et alertes immédiates).
