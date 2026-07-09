# Intégration Jaeger — Résumé des modifications

Basé sur ton `bote848-tp-session2.zip`. Tous les fichiers existants sont préservés à l'identique ; seuls les ajouts ci-dessous ont été faits. Testé réellement (npm install + démarrage serveur + requêtes) avant livraison.

## Fichiers modifiés (6)

| Fichier | Modification |
|---|---|
| `docker-compose.yml` | + service `jaeger` (all-in-one, ports 16686/4317/4318) ; + variables `OTEL_*` sur `app-sample` ; + `depends_on: jaeger` |
| `app-sample/tracing.js` | **Nouveau.** Initialisation OpenTelemetry (SDK + auto-instrumentation + export OTLP vers Jaeger) |
| `app-sample/server.js` | `require('./tracing')` en tout premier ; route `/api/checkout` instrumentée avec des spans manuels |
| `app-sample/package.json` | + 4 dépendances OpenTelemetry |
| `app-sample/Dockerfile` | `tracing.js` ajouté à la ligne `COPY` |
| `scripts/check-tp-ready.sh` | + vérification conteneur `jaeger`, UI Jaeger, et présence de traces `api-gateway` |

Aucun autre fichier (TP1-loki.md, TP2-prometheus.md, README.md, configs Loki/Prometheus/Promtail, grafana/) n'a été touché.

## ⚠️ Changement de topologie important

Ton `app-sample` est **une seule application** (`api-gateway`), pas plusieurs microservices. Le TP-Jaeger.md, REFERENCE-JAEGER.md, le PPTX et l'animation que j'avais produits avant supposaient des services séparés (`checkout-service`, `inventory-service`, `payment-service`...). Ce n'est pas ce que tu as réellement.

**Ce que fait le patch à la place** : il instrumente la vraie route `/api/checkout` avec 4 spans internes, tous dans le service `api-gateway` :

```
POST /api/checkout                    (span auto, créé par l'auto-instrumentation Express)
├─ checkout.validate_cart             (~10-25ms, simulé)
├─ checkout.check_inventory           (~15-40ms, simulé)
└─ checkout.process_payment           (le point chaud)
   └─ payment.stripe_api_call         (100-300ms normal, jusqu'à ~500ms+ en incident)
```

C'est un waterfall **réel et honnête** (spans réellement émis par du code réel), juste avec un seul `service.name` (`api-gateway`) au lieu de plusieurs. Pédagogiquement, le point clé (localiser le span fautif dans la hiérarchie) reste intact — seule la terminologie "plusieurs microservices" ne correspond pas à la réalité de la stack.

Le `trace_id` réel est maintenant aussi injecté dans les logs JSON (`context.trace_id`), ce qui rend la corrélation logs↔traces du Bloc 3 réellement fonctionnelle sur ta stack.

## Comment démarrer

```bash
# Dans le dossier du projet, avec les fichiers patchés en place
docker compose up -d --build
sleep 30
./scripts/check-tp-ready.sh

# Générer du trafic checkout pour peupler Jaeger (traffic-gen le fait déjà en continu)
# Vérifier dans Jaeger UI :
open http://localhost:16686   # ou juste ouvrir l'URL dans le navigateur
```

## Ce qu'il reste à faire de mon côté (à confirmer)

~~Les documents suivants...~~ **Fait.** Les 5 documents ont été corrigés pour coller exactement au stack réel (service unique `api-gateway`, spans internes, pas de profils Docker Compose) :

- `TP-Jaeger.md` — v2.0, scénario réécrit autour du vrai piège : le span en erreur n'est pas plus long que la normale, c'est son **statut** qui le trahit
- `REFERENCE-JAEGER.md` — exemples remplacés par le vrai code déployé, docker-compose sans profils, corrélation trace_id avec de vraies valeurs
- `BOTE848_S3_Traces_Jaeger_MSPR.pptx` — slides 3, 4, 5, 6, 9, 11, 12, 14, 17, 18 corrigées (waterfall réel, piège statut/durée, topologie honnête)
- `jaeger-animation.html` — refonte complète : un seul nœud `api-gateway`, spans internes, fin de l'animation sur le statut ERROR plutôt qu'une durée exagérée
- `MSPR-ShopFlow-Incident2.md` — v2.0, **changement de scénario** : au lieu d'inventer un `inventory-service` séparé, le Bloc 4 réutilise `GET /api/slow`, une route **réellement lente** (500-2500ms, déjà présente dans votre code, désormais instrumentée avec les spans `inventory.check_stock` → `inventory.db_lock_wait`). Contraste pédagogique avec le Bloc 2 : Bloc 2 = erreur sans lenteur, Bloc 4 = lenteur sans erreur.

**Un second patch `server.js`** a donc été nécessaire (au-delà de celui déjà livré) : la route `/api/slow` est maintenant instrumentée. Le fichier `server.js` fourni dans ce dossier est la version à jour incluant les deux patches. Testé (route `/api/slow` retourne bien `duration_ms` + `traceId`, aucune erreur JS).

**Fichiers désormais obsolètes** (scénario MSPR précédent, remplacé) : `inventory-incident-logs.jsonl`, `generate_incident.py`, `inject_loki.py`, `NOTE-ANIMATEUR-live-metrics-traces.md` — retirés de la livraison. Le nouveau scénario MSPR est **live** (pas de dataset à injecter), comme `/api/checkout` l'était déjà pour le Bloc 2.
