# TP — Traces Distribuées avec Jaeger (version autonome)
## Séance 3 / Bloc 2 — Durée : 1h

> 📌 **Ce document est conçu pour être suivi sans intervention orale d'un animateur.** Tout ce dont vous avez besoin — rappels de concepts, guidage clic par clic, exemples de résultats attendus, indices en cas de blocage — est inclus. Si un point reste flou malgré tout, notez votre question et continuez : mieux vaut avancer avec une incertitude que rester bloqué.

---

## 🎯 Objectifs

À l'issue de ce TP, vous serez capable de :
- Déployer Jaeger dans le stack ShopFlow existant
- Comprendre le principe de l'instrumentation OpenTelemetry (auto + manuelle)
- Rechercher une trace dans Jaeger UI et lire un waterfall
- Identifier **quel span précis** est responsable d'un échec, et lire ses attributs techniques
- Corréler une trace avec les logs via le `trace_id`

## 📋 Prérequis

- Stack Docker de la séance 2 déjà démarré
- Docker et Docker Compose fonctionnels
- Avoir traité le TP1-loki.md et TP2-prometheus.md (séance 2) — on réutilise la **même application** (`app-sample`, service `api-gateway`)

---

## 📖 Rappel express des concepts (lisez avant de commencer)

Si vous avez suivi la théorie juste avant, ceci n'est qu'une piqûre de rappel. Si vous reprenez ce TP plus tard, c'est votre point de départ.

**Trace** : le parcours complet d'une requête, identifié par un `trace_id` unique.

**Span** : une unité de travail à l'intérieur d'une trace (un appel, une opération). Chaque span a un nom, une durée, peut avoir des spans enfants, et porte deux informations essentielles :
- un **statut** : `OK` ou `ERROR` (rien d'intermédiaire)
- des **attributs** (tags) : des métadonnées libres, par exemple `payment.provider = stripe`

**Waterfall** : la représentation visuelle d'une trace, en cascade. La **largeur** d'une barre = sa durée. L'**indentation** = la relation parent/enfant.

**⚠️ Point d'architecture important** : `app-sample` est **une seule application** (service `api-gateway`), pas plusieurs microservices séparés. Les "étapes" que vous allez voir dans la trace (`checkout.validate_cart`, `checkout.check_inventory`, `checkout.process_payment`, `payment.stripe_api_call`) sont des **spans internes** au même service, pas des appels réseau entre conteneurs différents. Le principe de lecture d'un waterfall reste identique, qu'il traverse un seul service ou dix.

**Le fil conducteur de ce TP** : vous allez comparer une trace qui réussit et une trace qui échoue, span par span, pour découvrir ce qui les différencie réellement. Ne présumez de rien avant d'avoir regardé les données — ce TP contient volontairement un résultat contre-intuitif.

---

## Partie A — Déploiement de Jaeger (10 min)

Le service `jaeger` a été ajouté à votre `docker-compose.yml`, et `app-sample` a été instrumenté avec OpenTelemetry (fichier `tracing.js` + spans manuels dans `server.js`).

### Étape 1 — Démarrer le stack

```bash
cd ~/BOTE848

# Reconstruire l'image app-sample (nouvelles dépendances OpenTelemetry)
# et démarrer le stack complet, y compris jaeger
docker compose up -d --build
```

**Ce que vous devez voir** : une liste de services avec `Built`, `Created`, `Started`. Si la commande se termine avec une ligne `Error response from daemon: ...`, allez directement à la section Troubleshooting en bas de ce document avant de continuer — ne passez pas à l'étape suivante avec une erreur non résolue.

### Étape 2 — Vérifier que tout tourne

```bash
sleep 15
docker compose ps
```

**Ce que vous devez voir** : une liste de conteneurs, tous en statut `Up`, incluant désormais `jaeger` (en plus de ceux déjà connus depuis la séance 2 : `app-sample`, `traffic-gen`, `loki`, `promtail`, `prometheus`, `node-exporter`, `cadvisor`, `grafana`).

Si un conteneur reste en `Created` sans jamais passer à `Up` : relancez simplement `docker compose up -d` (sans `--build` cette fois).

### Étape 3 — Vérifier Jaeger UI

```bash
curl -s http://localhost:16686 -o /dev/null -w "%{http_code}\n"
```

**Résultat attendu** : `200`

Ouvrez ensuite **http://localhost:16686** dans votre navigateur — vous devez voir l'interface Jaeger UI (bandeau noir en haut, "JAEGER UI").

### Étape 4 — Vérification complète automatisée

```bash
./scripts/check-tp-ready.sh
```

**Résultat attendu** : toutes les lignes en ✓ OK, y compris en bas de la sortie :
```
Jaeger UI (16686)                      ✓ OK
Jaeger reçoit des traces               ✓ OK
```

Si "Jaeger reçoit des traces" est en ✗ FAIL : c'est normal si vous venez tout juste de démarrer le stack. `traffic-gen` a besoin d'un peu de temps pour émettre ses premières requêtes. Attendez 30 secondes et relancez le script.

### ✅ Checkpoint Partie A

Ne continuez pas tant que :
- [ ] `docker compose ps` montre tous les conteneurs `Up`, y compris `jaeger`
- [ ] http://localhost:16686 affiche l'interface Jaeger UI
- [ ] `check-tp-ready.sh` est entièrement au vert

---

## Partie B — Comprendre l'instrumentation (15 min)

### Étape 1 — Lire les fichiers

```bash
cat ~/BOTE848/app-sample/tracing.js
cat ~/BOTE848/app-sample/server.js
```

### Étape 2 — Comprendre `tracing.js`

Ce fichier initialise le SDK OpenTelemetry et l'auto-instrumentation (Express, HTTP) :

```javascript
const sdk = new NodeSDK({
  serviceName: SERVICE_NAME,               // "api-gateway"
  traceExporter: new OTLPTraceExporter({ url: OTLP_ENDPOINT }),
  instrumentations: [getNodeAutoInstrumentations({ /* ... */ })],
});
sdk.start();
```

Point à comprendre : ce fichier doit être chargé **avant** tout le reste de l'application (regardez la toute première ligne de `server.js` : `require('./tracing')`). Pourquoi ? Parce que l'auto-instrumentation fonctionne en "patchant" les modules Node.js (Express, HTTP) au moment où ils sont chargés — si `tracing.js` était chargé après, il serait trop tard pour intercepter quoi que ce soit.

### Étape 3 — Comprendre l'instrumentation manuelle dans `server.js`

La route `/api/checkout` est instrumentée avec **4 spans manuels imbriqués** :

```javascript
app.post('/api/checkout', async (req, res) => {
  // ...
  await tracer.startActiveSpan('checkout.validate_cart', async (span) => {
    await new Promise((r) => setTimeout(r, Math.random() * 15 + 10));
    span.end();
  });

  await tracer.startActiveSpan('checkout.check_inventory', async (span) => {
    await new Promise((r) => setTimeout(r, Math.random() * 25 + 15));
    span.end();
  });

  await tracer.startActiveSpan('checkout.process_payment', async (paymentSpan) => {
    await tracer.startActiveSpan('payment.stripe_api_call', async (stripeSpan) => {
      await new Promise((r) => setTimeout(r, latencyMs));   // 100-300ms, TOUJOURS

      if (Math.random() < failureRate) {
        const errorCode = /* PAYMENT_TIMEOUT | PAYMENT_INVALID_CARD | PAYMENT_RATE_LIMIT */;
        stripeSpan.recordException(new Error(errorCode));
        stripeSpan.setStatus({ code: SpanStatusCode.ERROR, message: errorCode });
        paymentSpan.setStatus({ code: SpanStatusCode.ERROR, message: errorCode });
        // ...
      }
      // ...
    });
  });
});
```

Structure de la hiérarchie de spans qui en résulte :
```
POST /api/checkout                  (span racine, automatique)
├─ checkout.validate_cart
├─ checkout.check_inventory
└─ checkout.process_payment
   └─ payment.stripe_api_call
```

### ✍️ Questions (à répondre dans votre rapport)

1. Combien de spans **manuels** (créés explicitement avec `tracer.startActiveSpan`) sont exécutés pour une seule requête `/api/checkout` réussie ? Listez-les dans l'ordre.
2. Le span `payment.stripe_api_call` est-il **enfant** de `checkout.process_payment` ou son **frère** (même niveau) ? Justifiez avec le code (regardez où chaque `startActiveSpan` est imbriqué dans l'autre).
3. Que se passe-t-il exactement dans le code quand le paiement échoue ? Citez les deux méthodes appelées sur `stripeSpan`.
4. Regardez la ligne `await new Promise((r) => setTimeout(r, latencyMs))`. Cette ligne s'exécute-t-elle **différemment** selon que le paiement va réussir ou échouer ? Qu'est-ce que cela implique pour la durée du span en cas d'erreur ?

💡 **La question 4 est un piège volontaire.** Engagez-vous sur une réponse avant de continuer — écrivez-la, ne la gardez pas juste "dans votre tête". Vous allez la confronter à de vraies données dans la Partie C, et l'exercice n'a de valeur que si vous avez pris position avant de regarder.

Si vous êtes bloqué sur la question 4, dépliez l'indice ci-dessous — mais essayez d'abord sans.

<details>
<summary>🔍 Indice question 4 (à n'ouvrir qu'en dernier recours)</summary>

Regardez bien : la ligne `setTimeout(r, latencyMs)` est-elle **à l'intérieur** ou **à l'extérieur** du `if (Math.random() < failureRate)` ? Si elle est avant ce `if`, alors le délai s'applique-t-il de la même façon, que le tirage au sort échoue ou réussisse ensuite ?
</details>

⚠️ **Dans Jaeger UI, une trace affichera plus que ces 4 spans** (souvent une dizaine). L'auto-instrumentation Express crée aussi des spans techniques pour son fonctionnement interne (middleware, routing, parsing JSON...). Ce n'est pas une anomalie : concentrez-vous uniquement sur les spans nommés `checkout.*` et `payment.*` pour répondre aux questions — les autres ne sont pas à analyser dans ce TP.

### ✅ Checkpoint Partie B

- [ ] Vous avez identifié les 4 spans manuels et leur ordre
- [ ] Vous avez une réponse écrite à la question 4 (même incertaine)

---

## Partie C — Déclencher l'incident et rechercher les traces en erreur (20 min)

### Étape 1 — Rechercher une trace normale (référence)

`traffic-gen` tourne déjà en continu. Ouvrez **http://localhost:16686**.

**Guidage clic par clic** :
1. Dans le panneau de gauche, champ **Service** : sélectionnez `api-gateway`
2. Champ **Operation** : sélectionnez `POST /api/checkout`
3. Champ **Tags** : laissez vide
4. Champ **Lookback** : `Last Hour` (ou `Last 5 minutes` si vous venez de démarrer)
5. Cliquez sur le bouton bleu **Find Traces**

Vous obtenez une liste de traces (une vingtaine par défaut), avec un nuage de points au-dessus représentant leur durée dans le temps.

6. Cliquez sur l'une des traces de la liste qui a un statut réussi (pas de pastille rouge dans le nuage de points, pas d'icône d'erreur à côté du nom)

Vous arrivez sur la vue détaillée du waterfall. Dans le panneau gauche, repérez les lignes `checkout.validate_cart`, `checkout.check_inventory`, `checkout.process_payment`, `payment.stripe_api_call` — ignorez les lignes `middleware - ...` et `request handler - ...` qui les entourent.

📊 **Exemple réel obtenu lors d'un test** (vos chiffres seront différents à chaque exécution, l'aléatoire fait partie du scénario) :

| Span | Durée observée |
|---|---|
| `checkout.validate_cart` | 21.49 ms |
| `checkout.check_inventory` | 29.64 ms |
| `checkout.process_payment` | 189.79 ms |
| `payment.stripe_api_call` | 189.51 ms |
| **Total Spans (avec le bruit Express)** | **10** |

Notez que `payment.stripe_api_call` occupe presque toute la durée de `checkout.process_payment` (normal, c'est son unique enfant), et que cette durée domine largement le temps total de la requête — cohérent avec ce qu'on a vu en théorie (Bloc 1, slide 3 et 4).

### ✍️ Question

5. Notez la durée de chacun des 4 spans (`checkout.*` / `payment.*`) sur **votre propre** trace normale. Sont-elles très différentes les unes des autres, ou du même ordre de grandeur ?

### Étape 2 — Déclencher l'incident

```bash
./scripts/trigger-incident.sh start
# → taux d'échec paiement : 5% -> 30%
```

Laissez tourner **1 minute** pour que `traffic-gen` génère plusieurs échecs.

### Étape 3 — Rechercher une trace en erreur

Retournez dans Jaeger UI et relancez une recherche :

**Guidage clic par clic** :
1. Champ **Service** : `api-gateway` (déjà sélectionné normalement)
2. Champ **Operation** : `POST /api/checkout`
3. Champ **Tags** : tapez `error=true`
4. Champ **Lookback** : `Last 5 minutes`
5. **Find Traces**

Vous devriez voir moins de résultats qu'à l'étape 1 (logique : seules les traces en erreur remontent). S'il n'y a **aucun** résultat, patientez encore 30 secondes et relancez la recherche — `trigger-incident.sh` fait monter le taux d'échec, mais il faut que `traffic-gen` ait le temps de générer plusieurs requêtes derrière.

6. Cliquez sur une des traces trouvées

Vous devriez voir des icônes ❗ rouges à côté de `POST /api/checkout`, `checkout.process_payment`, et `payment.stripe_api_call` — c'est la propagation visuelle du statut d'erreur, du span fautif jusqu'à la racine.

7. Cliquez directement sur la ligne `payment.stripe_api_call` (celle avec l'icône rouge) — un panneau de détail s'ouvre en dessous, avec une section **Tags**.

📊 **Exemple réel obtenu lors d'un test** :

```
payment.stripe_api_call
Service: api-gateway    Duration: 129.91ms    Start Time: 42ms

Tags:
  error = true
  otel.status_code = ERROR
  otel.status_description = PAYMENT_RATE_LIMIT
  payment.provider = stripe
  span.kind = internal

Logs (1)
```

Le champ `otel.status_description` porte le code d'erreur exact — ici `PAYMENT_RATE_LIMIT`, mais chez vous ce sera peut-être `PAYMENT_TIMEOUT` ou `PAYMENT_INVALID_CARD` (choisi aléatoirement par le code à chaque échec).

8. Cliquez sur la ligne **Logs (1)** pour dérouler le détail de l'exception enregistrée (`recordException`) — vous devriez y voir le message d'erreur complet.

### ✍️ Questions

6. Quel est le `trace_id` de cette trace ? (Visible en haut de la page à côté du nom, en version courte — récupérez la version **complète** dans l'URL de votre navigateur, après `/trace/`.)
7. Parmi les 4 spans `checkout.*` / `payment.*`, lequel est marqué en erreur ? Les 3 autres le sont-ils aussi ?
8. Relevez l'attribut `payment.provider` et le code d'erreur exact (`otel.status_description`) de ce span.
9. **Comparez la durée de ce span en erreur avec les durées notées à la question 5 (trace normale).** Le span en erreur est-il anormalement long, comme vous l'auriez peut-être supposé ?

💡 Reprenez votre réponse écrite à la question 4 : aviez-vous anticipé ce résultat ?

### Étape 4 — Revenir à la normale

**Ne sautez pas cette étape** — sinon le taux d'échec restera élevé pour tout le monde sur la stack partagée.

```bash
./scripts/trigger-incident.sh stop
# → taux d'échec paiement : 30% -> 5%
```

### ✅ Checkpoint Partie C

- [ ] Vous avez trouvé et noté les durées d'une trace normale
- [ ] Vous avez trouvé une trace en erreur avec `error=true`
- [ ] Vous avez le `trace_id` complet noté quelque part (vous en aurez besoin en Partie D)
- [ ] Vous avez relancé `trigger-incident.sh stop`

---

## Partie D — Synthèse et corrélation (15 min)

### Étape 1 — Corréler avec les logs (bonus fortement recommandé)

Ouvrez Grafana : **http://localhost:3000** (identifiants habituels de la séance 2)

1. Menu de gauche → **Explore**
2. Sélectionnez la datasource **Loki** (généralement nommée `loki-shopflow` ou équivalent) en haut à gauche
3. Dans le champ de requête, collez (adaptez avec **votre** `trace_id` complet noté à la question 6) :

```logql
{service="api-gateway"} |= "VOTRE_TRACE_ID_ICI"
```

4. **Run query**

📊 **Exemple réel obtenu lors d'un test** (une seule ligne trouvée, exactement sur la fenêtre de l'incident) :

```json
{
  "timestamp": "2026-07-07T22:45:02.450Z",
  "level": "error",
  "service": "api-gateway",
  "message": "Payment processing failed",
  "user_id": "user_715",
  "order_id": "order_841",
  "amount": 41,
  "error_code": "PAYMENT_RATE_LIMIT",
  "duration_ms": 129,
  "attempt_number": 1,
  "trace_id": "692ac39d8a00d664e8b49217cb1a0553"
}
```

Vérifiez trois choses en comparant avec ce que vous avez vu dans Jaeger :
- Le `trace_id` du log correspond-il exactement à celui de votre trace Jaeger ?
- Le `error_code` du log correspond-il à `otel.status_description` vu dans Jaeger ?
- Le `duration_ms` du log est-il cohérent avec la durée du span `payment.stripe_api_call` ?

Si les trois correspondent : vous venez de valider concrètement la corrélation logs ↔ traces vue en théorie (Bloc 1).

### Étape 2 — Compléter le rapport

```markdown
## Incident Analysis — Traces

**Trace ID (trace en erreur)** : _______________________
**Span en erreur** : _______________________
**Durée du span en erreur** : _______ ms
**Durée moyenne du même span sur une trace normale** : _______ ms
**Écart de durée notable ?** : Oui / Non

**Attribut technique relevé (error_code / message d'exception)** :
_______________________________________________________________

**Ce que cette trace montre, que les logs et métriques seuls ne montraient pas** :
_______________________________________________________________

**Root cause réelle de l'incident déclenché** :
_______________________________________________________________

**Confirmation par corrélation logs (trace_id retrouvé dans Loki)** : Oui / Non
```

### 🔓 Corrigé indicatif (à consulter seulement après avoir complété votre rapport)

<details>
<summary>Cliquez pour dérouler le corrigé</summary>

**Écart de durée notable ?** Non. Le span en erreur dure sensiblement la même chose (souvent même un peu moins) qu'un span en succès — les deux tournent autour de 100 à 300ms, cohérent avec le code (`latencyMs` est tiré aléatoirement de la même façon, que le paiement réussisse ou échoue ensuite).

**Ce que la trace montre en plus des logs/métriques** : la localisation exacte, dans la hiérarchie d'appels, de l'opération technique qui a échoué — avec un attribut structuré (`error_code`) directement exploitable, sans avoir à parser un message de log en texte libre.

**Root cause réelle** : le taux d'échec du provider de paiement (Stripe, simulé) a été délibérément augmenté de 5% à 30% par `trigger-incident.sh`. Ce n'est pas une panne de latence, c'est une hausse de la probabilité d'échec sur chaque tentative — indépendamment de leur durée.

**Piège pédagogique de ce TP** : l'intuition la plus répandue en tracing est de repérer la barre la plus large du waterfall pour trouver le coupable. Ici, cette intuition ne suffit pas : le span fautif n'est pas le plus long, il est juste **en erreur**. C'est le **statut** (`ERROR`) et l'**attribut** (`error_code`) qui portent l'information, pas la durée. Retenez cette distinction : tous les incidents ne sont pas des histoires de lenteur.

</details>

---

## ✅ Livrables attendus

- [ ] Réponses aux 9 questions du document
- [ ] Capture d'écran d'une trace normale (les 4 spans `checkout.*`/`payment.*` identifiés, tous OK)
- [ ] Capture d'écran d'une trace en erreur (span `payment.stripe_api_call` en rouge, tags visibles)
- [ ] Rapport de synthèse complété (Partie D, étape 2)
- [ ] Capture de la corrélation `trace_id` dans Loki (Partie D, étape 1)

---

## 🛠️ Troubleshooting complet

### Problèmes liés à Jaeger spécifiquement

| Problème | Solution |
|----------|----------|
| Jaeger UI ne charge pas | Vérifier `docker compose ps` ; le port 16686 doit être libre sur votre machine |
| `api-gateway` n'apparaît pas dans la liste des services Jaeger | L'image `app-sample` n'a peut-être pas été reconstruite : relancer `docker compose up -d --build` |
| Aucune trace avec `error=true` | Vérifier que `trigger-incident.sh start` a bien été exécuté ; laisser `traffic-gen` tourner 1 minute de plus |
| Le waterfall semble incomplet (spans manquants) | Vérifier dans les logs du conteneur (`docker compose logs app-sample`) qu'aucune erreur d'export OTLP n'apparaît |
| Toutes les traces semblent avoir la même durée | C'est normal et attendu ici (cf. Partie C, question 9) — ce n'est pas un bug |
| Une trace affiche 10 spans au lieu de 4 | Normal — l'auto-instrumentation Express ajoute des spans techniques. Concentrez-vous sur les 4 spans `checkout.*`/`payment.*` |

### Problèmes de déploiement plus généraux (déjà rencontrés sur ce TP)

| Problème | Cause probable | Solution |
|----------|----------------|----------|
| Erreur de build `"/traffic-generator.js": not found` | Fichier manquant dans `app-sample/` après une copie de fichiers | Vérifiez `ls app-sample/` — le fichier doit être présent à côté de `server.js` |
| Erreur `node-exporter` : `path / is mounted ... not a shared or slave mount` | Propagation de montage `rslave` non supportée par Docker Desktop (Windows/Mac) | Dans `docker-compose.yml`, le volume de `node-exporter` doit être `/:/host:ro` (sans `,rslave`) |
| Erreur `promtail` : *"mounting a directory onto a file"* | Un fichier de config (`.yml`) a été remplacé par un dossier vide par Docker Desktop | Supprimer le dossier vide, recréer le vrai fichier `.yml` à sa place |
| Target `app-sample` DOWN dans Prometheus (`connection refused`) sur un port inattendu | `prometheus.yml` pointe vers le mauvais port | Vérifier que le job `app-sample` cible bien `app-sample:8000` (pas 8080) |
| Après correctif, certains conteneurs restent en `Created` sans démarrer | Une commande `docker compose up -d` précédente a été interrompue par une erreur avant la fin | Relancer simplement `docker compose up -d` (sans argument supplémentaire) pour finir de démarrer ce qui manque |

Si un problème persiste après avoir consulté ce tableau, notez le message d'erreur exact et demandez de l'aide — ne restez pas bloqué plus de 5-10 minutes sur un problème d'infrastructure, ce n'est pas l'objectif pédagogique de ce TP.

---

*Document participant — Version autonome 1.0*
