# TP — Manipulation de logs

> **Session 1 / Bloc 4** — Durée : 1 h
> **Module** BOTE848 — Supervision & Observabilité DevOps

---

## 🎯 Objectifs

À la fin de ce TP, vous serez capable de :

1. Comparer **logs texte non structurés** vs **logs JSON structurés**, et expliquer pourquoi le second l'emporte.
2. Écrire des requêtes `jq` basiques pour filtrer, compter, agréger des logs.
3. **Diagnostiquer un incident réel** à partir des seuls logs, en moins de 10 minutes.

---

## ✅ Prérequis

Avant de commencer, vérifiez que vous avez :

```bash
docker --version          # ≥ 20.x
docker compose version    # ≥ v2.x  (ou : docker-compose --version)
jq --version              # ≥ 1.6
```

Si l'un manque :

| Outil | macOS | Linux (Debian/Ubuntu) | Windows |
|---|---|---|---|
| Docker | Docker Desktop | `apt-get install docker.io docker-compose-plugin` | Docker Desktop |
| jq | `brew install jq` | `apt-get install jq` | `winget install jqlang.jq` |

---

## 🌍 Le scénario

> **Vendredi 16 mai 2025, 10 h 45.**
> Vous êtes SRE on-call sur la plateforme **Acme-Shop**, un site e-commerce français qui fait 2 M€ de CA par mois.
> AlertManager vient de vous notifier :
>
> > 🚨 `Error rate critical on checkout-service — threshold 1% breached for 5 min`
>
> Le directeur technique est dans 10 minutes en visio. Il veut savoir **ce qui se passe, à quel point c'est grave, et ce que vous faites pour résoudre**. Pas le temps d'aller voir les développeurs : vous n'avez que les logs.

---

## Phase 0 — Mise en route (5 min)

### 0.1 Démarrer l'environnement

Dans un terminal, depuis le dossier `tp-logs/` :

```bash
docker compose up -d
```

Attendre 5-10 secondes, puis vérifier :

```bash
docker compose ps
```

Vous devez voir **2 services** en état `running` ou `Up` :

```
NAME              IMAGE          STATUS
app-sample        busybox:1.36   Up
app-sample-json   busybox:1.36   Up
```

### 0.2 Vérifier qu'on capte bien les logs

```bash
docker logs app-sample | head -3
```

Vous devez voir des lignes de logs apparaître. Si la sortie est vide, attendre 5 s de plus et réessayer.

---

## Phase 1 — Logs texte non structurés (15 min)

**Contexte** : on commence par regarder les logs du service legacy d'Acme-Shop, qui écrit en texte plat.

### 1.1 Premier coup d'œil

```bash
docker logs app-sample
```

Vous voyez **40 lignes** de logs. Prenez 2 minutes pour les parcourir.

### 1.2 Questions auxquelles vous devez répondre

Sans regarder la phase suivante, essayez :

> 1. Combien y a-t-il d'erreurs de paiement au total ?
> 2. Quel utilisateur est le plus affecté ?
> 3. À quelle heure exacte la première erreur s'est-elle produite ?
> 4. Combien d'erreurs y a-t-il eu entre 10 h 35 et 10 h 40 ?

### 1.3 Tentez `grep`

```bash
# Une 1re piste...
docker logs app-sample | grep -i error

# Plus précis ?
docker logs app-sample | grep -iE "payment|error"
```

### 1.4 Le constat

Notez par écrit (en 1-2 phrases) les difficultés que vous avez rencontrées :

- Le format des timestamps est-il cohérent ?
- Le format des messages d'erreur est-il identique d'une ligne à l'autre ?
- Pouvez-vous extraire facilement le `user_id` ou l'`order_id` impacté ?

> ⏸  **Pause de réflexion**. Quand le formateur le demande, on partage les observations à voix haute.

---

## Phase 2 — Logs JSON structurés (15 min)

**Contexte** : Acme-Shop a refondu son système de logs il y a 6 mois. Tout est désormais émis en JSON ligne-par-ligne (format **JSONL**).

### 2.1 Premier coup d'œil

```bash
docker logs app-sample-json | head -5
```

Vous voyez maintenant chaque ligne sous forme **d'objet JSON**.

### 2.2 Lecture commentée d'une ligne

Voici à quoi ressemble une entrée typique :

```json
{
  "timestamp": "2025-05-16T10:34:22Z",
  "level": "ERROR",
  "service": "payment-service",
  "trace_id": "abc123def456",
  "message": "Database connection timeout",
  "context": {
    "user_id": "user_456",
    "order_id": "order_12345",
    "amount_eur": 299.99,
    "duration_ms": 5000
  }
}
```

**Repérez** :

- `timestamp` : ISO 8601, parsable sans effort.
- `level` : valeur fixe (`INFO`, `WARN`, `ERROR`), filtrable.
- `service` : on sait d'où ça vient.
- `trace_id` : on peut relier tous les logs d'une même requête.
- `context` : **les données métier embarquées** — c'est la clé.

### 2.3 Mettre en forme la sortie (si jq installé sur la machine)

```bash
docker logs app-sample-json | head -3 | jq .
```

`jq .` reformate (« pretty-print ») le JSON. C'est la commande la plus utile au monde quand on apprend `jq`.

---

## Phase 3 — Requêtes `jq` (15 min)

On va apprendre `jq` par la pratique, du plus simple au plus expressif.

Dans ce TP, jq n'a PAS besoin d'installé sur votre machine.

Nous allons utiliser un container Docker appelé logs-toolbox qui contient :
- jq
- grep
- awk
- sort
- uniq
- wc

### 3.0 Ouvrir le terminal toolbox
```bash
docker exec -it logs-toolbox sh
```

### 3.1 Filtrer par niveau

```bash
# Toutes les entrées de niveau ERROR
cat /data/app-logs.jsonl \
| jq 'select(.level=="ERROR")'
```

**Q** : combien y a-t-il d'erreurs dans la journée nominale ?

```bash
cat /data/app-logs.jsonl \
| jq 'select(.level=="ERROR")' \
| jq -s 'length'
```

### 3.2 Filtrer par service

```bash
# Toutes les entrées du checkout-service
cat /data/app-logs.jsonl \
| jq 'select(.service=="payment-service")'
```

### 3.3 Filtrer sur plusieurs conditions

```bash
# Les erreurs SUR le payment-service uniquement
cat /data/app-logs.jsonl \
| jq 'select(.level=="ERROR" and .service=="payment-service")'
```

### 3.4 Extraire des champs précis

```bash
# Pour chaque erreur, ne sortir que user_id et order_id
cat /data/app-logs.jsonl \
| jq 'select(.level=="ERROR")
      | {
          user: .context.user_id,
          order: .context.order_id,
          msg: .message
        }'
```

### 3.5 Compter par catégorie

```bash
# Combien d'événements par service ?
cat /data/app-logs.jsonl \
| jq -r '.service' \
| sort \
| uniq -c \
| sort -rn
```

```bash
# Combien d'événements par niveau ?
cat /data/app-logs.jsonl \
| jq -r '.level' \
| sort \
| uniq -c
```

### 3.6 Grouper par heure

```bash
# Distribution horaire des événements
cat /data/app-logs.jsonl \
| jq -r '.timestamp[0:13]' \
| sort \
| uniq -c
```

`.timestamp[0:13]` extrait les 13 premiers caractères, soit `2025-05-16T09`, `2025-05-16T10`, etc.

> 💡 **Mini-défi** : trouver le service qui a généré le plus d'événements ce jour-là.

---

## Phase 4 — Diagnostic d'un incident (10 min)

🚨 **Retour au scénario** : AlertManager vient de notifier, vous avez 10 min avant la visio avec le directeur technique.

### 4.1 Les données

Le snapshot des logs des 10 dernières minutes a été sauvegardé dans :

```
tp-data/incident-logs.json
```

Vérifier qu'il est bien là :

```bash
ls -lh tp-data/incident-logs.json
```

### 4.2 Votre mission

Répondez aux **5 questions du débrief** :

1. **Combien d'erreurs** sur la fenêtre observée ?
2. **Quel type d'erreur** domine (error_code) ?
3. **À quelle heure** la première erreur a-t-elle eu lieu ?
4. **Quelle est la cause racine probable** ? (regarder `duration_ms`, `gateway`...)
5. **Combien d'utilisateurs uniques** sont impactés ?

### 4.3 Points de départ (si vous bloquez)

```bash
# Question 1 : compter les erreurs
cat tp-data/incident-logs.json | jq -c 'select(.level=="ERROR")' | wc -l

# Question 2 : distribution par error_code
cat tp-data/incident-logs.json \
  | jq -r 'select(.level=="ERROR") | .context.error_code' \
  | sort | uniq -c | sort -rn

# Question 3 : la 1re erreur (chronologiquement)
cat tp-data/incident-logs.json \
  | jq -r 'select(.level=="ERROR") | .timestamp' \
  | sort | head -1
```

À vous d'écrire les requêtes pour les questions 4 et 5.

### 4.4 Livrable

Sur une feuille (ou un fichier `mon-rapport.md`), produisez le **rapport d'incident** suivant :

```markdown
## Rapport d'incident — [Heure]

**Période observée** :
**Nombre d'erreurs**  :
**Code d'erreur dominant** :
**Première occurrence** :
**Utilisateurs uniques impactés** :

**Hypothèse de cause racine** :
(2-3 phrases)

**Preuves** (requêtes jq qui appuient l'hypothèse) :
- ...
- ...

**Action immédiate recommandée** :
(1-2 phrases)
```

---

## 🏁 Pour conclure

À la fin du TP, **arrêter les containers** proprement :

```bash
docker compose down
```

### Ce qu'on a fait, en une phrase

> On a transformé un mur de texte illisible en données **interrogeables**, et utilisé ces données pour passer d'une alerte vague (« error rate high ») à un diagnostic précis en moins de 10 minutes.

C'est exactement le saut entre **supervision** et **observabilité**.

### Pour aller plus loin (après la session)

- 📖 Le « manuel » de `jq` : https://jqlang.github.io/jq/manual/
- 📖 Tutoriel `jq` interactif : https://jqplay.org/
- 📖 Les **structured logs** chez les grands : ELK / Loki / Datadog — concept identique.
