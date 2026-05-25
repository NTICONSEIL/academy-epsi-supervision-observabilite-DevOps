# 📌 Tutoriel **jq** : Guide Complet pour Manipuler du JSON en CLI

*jq* est un processeur JSON léger et flexible pour la ligne de commande. Il permet de **filtrer, transformer, formater et manipuler** des données JSON de manière efficace.

---

## 📥 Installation

### Linux (Debian/Ubuntu)

```bash
sudo apt update && sudo apt install jq
```

### macOS (avec Homebrew)

```bash
brew install jq
```

### Windows (via Chocolatey ou Scoop)

```powershell
# Avec Chocolatey
choco install jq

# Avec Scoop
scoop install jq
```

### Vérifier l'installation

```bash
jq --version
# Sortie attendue : jq-1.6 (ou version supérieure)
```

---

## 🎯 Concepts de Base

### 1. **Syntaxe de base**

jq utilise une syntaxe inspirée de **Python** et **JavaScript** pour interroger et transformer du JSON.

- **Entrée** : JSON via un fichier, un pipe (`|`), ou une chaîne de caractères.
- **Sortie** : JSON formaté par défaut (ou texte brut avec `-r`).

### 2. **Exécuter jq**

```bash
# Depuis un fichier
jq '.key' data.json

# Depuis un pipe (ex: curl)
curl -s https://api.example.com/data | jq '.users[0].name'

# Depuis une chaîne de caractères
echo '{"name": "Élodie", "age": 30}' | jq '.name'
```

---

## 🔍 Sélection et Filtrage

### 1. **Accéder à une clé simple**

```json
{
  "name": "Élodie",
  "age": 30,
  "city": "Paris"
}
```

```bash
echo '{"name": "Élodie", "age": 30, "city": "Paris"}' | jq '.name'
# Sortie : "Élodie"
```

### 2. **Accéder à un tableau**

```json
{
  "users": [
    {"name": "Élodie", "age": 30},
    {"name": "Théo", "age": 25}
  ]
}
```

```bash
# Premier élément du tableau
jq '.users[0].name' data.json
# Sortie : "Élodie"

# Tous les noms des utilisateurs
jq '.users[].name' data.json
# Sortie : "Élodie" "Théo"

# Avec index négatif (dernier élément)
jq '.users[-1].name' data.json
# Sortie : "Théo"
```

### 3. **Filtrer avec des conditions**

```bash
# Utilisateurs de plus de 25 ans
jq '.users[] | select(.age > 25)' data.json

# Utilisateurs dont le nom commence par "É"
jq '.users[] | select(.name | startswith("É"))' data.json

# Filtrer avec plusieurs conditions
jq '.users[] | select(.age > 25 and .city == "Paris")' data.json
```

### 4. **Opérateurs de comparaison**


| Opérateur  | Description           | Exemple                     |
| ---------- | --------------------- | --------------------------- |
| `==`       | Égal à                | `select(.age == 30)`        |
| `!=`       | Différent de          | `select(.name != "Élodie")` |
| `>`        | Supérieur à           | `select(.age > 25)`         |
| `<`        | Inférieur à           | `select(.age < 30)`         |
| `>=`       | Supérieur ou égal     | `select(.age >= 30)`        |
| `<=`       | Inférieur ou égal     | `select(.age <= 25)`        |
| `contains` | Contient (pour array) | `select(.tags               |


---

## 🔄 Transformation de Données

### 1. **Créer un nouvel objet**

```bash
# Extraire uniquement le nom et l'âge
jq '.users[] | {name, age}' data.json

# Sortie :
# {"name": "Élodie", "age": 30}
# {"name": "Théo", "age": 25}
```

### 2. **Renommer des clés**

```bash
jq '.users[] | {nom: .name, age: .age}' data.json
# Sortie : {"nom": "Élodie", "age": 30}
```

### 3. **Ajouter une clé**

```bash
jq '.users[] | . + {is_admin: false}' data.json
# Sortie : {"name": "Élodie", "age": 30, "is_admin": false}
```

### 4. **Modifier une valeur**

```bash
jq '.users[] | .age *= 2' data.json
# Sortie : {"name": "Élodie", "age": 60}
```

### 5. **Supprimer une clé**

```bash
jq 'del(.users[].age)' data.json
# Sortie : {"users": [{"name": "Élodie"}, {"name": "Théo"}]}
```

---

## 🧮 Opérations Mathématiques

### 1. **Calculs de base**

```bash
# Addition
jq '.users[] | .age + 5' data.json

# Multiplication
jq '.users[] | .age * 2' data.json

# Moyenne d'âge
jq '[.users[].age] | add / length' data.json
```

### 2. **Fonctions mathématiques**


| Fonction      | Description             | Exemple        |
| ------------- | ----------------------- | -------------- |
| `length`      | Longueur (array/string) | `jq '.users    |
| `add`         | Somme des éléments      | `jq '[1, 2, 3] |
| `min` / `max` | Min/Max d'un array      | `jq '[1, 2, 3] |
| `floor`       | Arrondi vers le bas     | `jq '1.7       |
| `ceil`        | Arrondi vers le haut    | `jq '1.2       |


---

## 🔗 Manipulation de Structures

### 1. **Fusionner des objets**

```bash
jq '{user: .users[0], metadata: {source: "API"}}' data.json
```

### 2. **Aplatir un objet imbriqué**

```json
{
  "user": {
    "name": "Élodie",
    "address": {
      "city": "Paris",
      "country": "France"
    }
  }
}
```

```bash
# Extraire toutes les valeurs
jq '.user | [.name, .address.city, .address.country] | @tsv' data.json
# Sortie : "Élodie" "Paris" "France"
```

### 3. **Grouper des données**

```bash
# Grouper les utilisateurs par ville
jq 'group_by(.city) | map({city: .[0].city, count: length})' data.json
```

---

## 📜 Fonctions Utiles

### 1. **Fonctions sur les strings**


| Fonction              | Description                | Exemple    |
| --------------------- | -------------------------- | ---------- |
| `length`              | Longueur de la chaîne      | `jq '.name |
| `startswith`          | Commence par               | `jq '.name |
| `endswith`            | Finit par                  | `jq '.name |
| `contains`            | Contient                   | `jq '.name |
| `split`               | Diviser en array           | `jq '.name |
| `join`                | Joindre un array en string | `jq '.tags |
| `upcase` / `downcase` | Majuscules/Minuscules      | `jq '.name |


### 2. **Fonctions sur les arrays**


| Fonction | Description            | Exemple           |
| -------- | ---------------------- | ----------------- |
| `length` | Nombre d'éléments      | `jq '.users       |
| `map`    | Appliquer une fonction | `jq '.users       |
| `filter` | Filtrer un array       | `jq '.users       |
| `sort`   | Trier un array         | `jq '.users       |
| `unique` | Supprimer les doublons | `jq '[1, 2, 2, 3] |


---

## 🎨 Formatage de la Sortie

### 1. **Sortie en JSON brut**

```bash
jq '.' data.json  # Formate le JSON avec des indentations
```

### 2. **Sortie en texte brut (`-r`)**

```bash
# Sans les guillemets
jq -r '.users[0].name' data.json
# Sortie : Élodie (au lieu de "Élodie")

# Pour un tableau
jq -r '.users[].name' data.json
# Sortie :
# Élodie
# Théo
```

### 3. **Sortie compacte (`-c`)**

```bash
jq -c '.users' data.json
# Sortie : [{"name":"Élodie","age":30},{"name":"Théo","age":25}]
```

### 4. **Sortie en TSV/CSV**

```bash
# TSV (Tab-Separated Values)
jq -r '.users[] | [.name, .age] | @tsv' data.json
# Sortie : Élodie   30

# CSV
jq -r '.users[] | [.name, .age] | @csv' data.json
# Sortie : "Élodie",30
```

---

## 🔧 Astuces Avancées

### 1. **Utiliser des variables**

```bash
# Stocker une valeur dans une variable
jq --arg name "Élodie" 'select(.name == $name)' data.json

# Avec plusieurs variables
jq --arg min_age 25 --arg city "Paris" \
  'select(.age >= ($min_age | tonumber) and .city == $city)' data.json
```

### 2. **Lire un fichier JSON dans une variable**

```bash
jq --slurpfile config config.json 'include "config"; .users[] | select(.id == $config[0].admin_id)' data.json
```

### 3. **Combiner plusieurs filtres**

```bash
# Utiliser `|` pour enchaîner des opérations
jq '.users | map(select(.age > 25)) | length' data.json
# Sortie : 1 (nombre d'utilisateurs de plus de 25 ans)
```

### 4. **Utiliser `if-then-else**`

```bash
jq '.users[] | if .age > 30 then "Senior" elif .age > 20 then "Junior" else "Child" end' data.json
```

### 5. **Boucles avec `foreach**`

```bash
jq 'foreach .users[] as $user ({}; . + {($user.name): $user.age})' data.json
# Sortie : {"Élodie": 30, "Théo": 25}
```

---

## 📂 Exemples Pratiques

### 1. **Extraire des données d'une API**

```bash
curl -s https://api.github.com/users/mistralai | jq '.name, .login, .bio'
```

### 2. **Filtrer et formater des logs**

```bash
# Supposons un fichier logs.json avec des entrées comme :
# {"timestamp": "2023-01-01", "level": "error", "message": "Failed"}

# Extraire les erreurs
jq '.[] | select(.level == "error")' logs.json

# Compter les erreurs par jour
jq 'group_by(.timestamp) | map({date: .[0].timestamp, errors: length})' logs.json
```

### 3. **Transformer un JSON en structure plate**

```json
# Entrée :
{
  "id": 1,
  "details": {
    "name": "Élodie",
    "address": {
      "city": "Paris"
    }
  }
}
```

```bash
# Sortie : {"id": 1, "details_name": "Élodie", "details_address_city": "Paris"}
jq 'with_entries(if .value | type == "object" then .value | with_entries(.key |= "\(.key)_\(.parent)") else . end)' data.json
```

### 4. **Valider un schéma JSON**

```bash
# Vérifier si un champ existe
jq 'has("name")' data.json

# Vérifier si un champ est de type array
jq '.users | type == "array"' data.json
```

---

## ⚠️ Bonnes Pratiques

1. **Utilise `-r` pour du texte brut** : Évite les guillemets inutiles dans les scripts Bash.
2. **Préfère `@tsv` ou `@csv` pour les exports** : Plus facile à importer dans des outils comme Excel.
3. **Utilise `--arg` pour les variables** : Évite les problèmes d'échappement de caractères.
4. **Teste avec `echo` avant de traiter des gros fichiers** : Valide ton filtre sur un petit exemple.
5. **Utilise `jq '.'` pour formater du JSON** : Rendu lisible avec des indentations.

---

## 📚 Ressources Utiles

- **[Documentation officielle](https://stedolan.github.io/jq/)**
- **[Tutoriel interactif](https://jqplay.org/)** (pour tester en ligne)
- **[Manual complet](https://stedolan.github.io/jq/manual/)**
- **[Cookbook jq](https://github.com/stedolan/jq/wiki/Cookbook)** (recettes courantes)

---

## 🚀 Exercices

1. **Facile** : Extraire tous les noms d'un tableau d'utilisateurs.
2. **Moyen** : Calculer l'âge moyen des utilisateurs.
3. **Difficile** : Regrouper les utilisateurs par ville et compter le nombre par ville.
4. **Expert** : Transformer un JSON imbriqué en une structure plate (ex: `{"user.name": "Élodie"}`).

---

> 💡 **Astuce** : Utilise `jq --help` pour voir toutes les options disponibles !

---

*Dernière mise à jour : Mai 2026*