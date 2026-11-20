# TP Séance 1 — Lire le Web avant de l'écrire
## Notes d'observation et réponses aux questions (Livrable S1)

**Projet :** Orbitale  
**Module :** M2 Web Services — Fondamentaux REST et styles d'API  
**Date :** 20 novembre 2026  

---

## Étape 0 — Préparation du poste

- **curl** : `curl 8.13.0` (accessible via `curl.exe` sous Windows PowerShell).
- **jq** : `jq-1.8.2` installé via `winget install jqlang.jq`.
- **Dépôt Git** : Initialisé pour le projet Orbitale avec la structure `docs/`.

---

## Étape 1 — Anatomie d’un échange HTTP

Commande exécutée :
```powershell
curl.exe -v https://api.github.com/repos/nodejs/node -o NUL
```

### 1. Ligne de requête et ligne de statut
- **Ligne de requête émise :**
  ```http
  GET /repos/nodejs/node HTTP/1.1
  ```
- **Ligne de statut reçue :**
  ```http
  HTTP/1.1 200 OK
  ```
- **Version HTTP utilisée :** `HTTP/1.1` (négociée via TLS ALPN).

### 2. Quatre en-têtes de réponse remarquables
1. `Content-Type: application/json; charset=utf-8` : Indique au client le type MIME de la représentation ainsi que son encodage de caractères, lui permettant de savoir comment désérialiser le flux d'octets.
2. `Cache-Control: public, max-age=60, s-maxage=60` : Spécifie les règles de mise en cache pour le navigateur/client (`max-age=60` secondes) et pour les caches intermédiaires/CDN (`s-maxage=60` secondes, `public`).
3. `ETag: W/"667e872bb2d616ece1b5f0d00495d591f51303835b4adf323e2ce10b06598103"` : Empreinte de validation de la ressource (ici un ETag faible `W/` garantissant l'équivalence sémantique). Utilisé pour la revalidation conditionnelle.
4. `X-RateLimit-Remaining: 39` : En-tête propriétaire GitHub indiquant le nombre de requêtes restantes autorisées dans la fenêtre horaire actuelle pour l'adresse IP.

### 3. Signification de `x-ratelimit-remaining` et dépassement
- `x-ratelimit-remaining` décompte le quota de requêtes encore disponibles avant restriction (sur une base horaire de 60 requêtes/heure pour les appels non authentifiés).
- **Quand il atteint 0 :** GitHub bloque temporairement les requêtes de ce client en renvoyant une erreur HTTP `403 Forbidden` (avec le message `"API rate limit exceeded"`). En REST moderne (RFC 6585), le code standard recommandé pour cet usage est `429 Too Many Requests`. Le serveur renvoie également `x-ratelimit-reset` indiquant le timestamp UNIX de réinitialisation du quota.

> **Point de contrôle 1 validé :**
> - Méthode : `GET`
> - Cible : `/repos/nodejs/node`
> - Statut : `200 OK`
> - Type de contenu : `application/json; charset=utf-8`
> - ETag : `W/"667e872bb2d616ece1b5f0d00495d591f51303835b4adf323e2ce10b06598103"`

---

## Étape 2 — Le cache conditionnel

Commande exécutée :
```powershell
cmd /c 'curl -s -o NUL -w "%{http_code}\n" -H "If-None-Match: W/\"667e872bb2d616ece1b5f0d00495d591f51303835b4adf323e2ce10b06598103\"" https://api.github.com/repos/nodejs/node'
```

### 1. Code obtenu et absence de corps
- **Code de statut retourné :** `304 Not Modified`
- **Pourquoi le corps est vide ?**  
  Le serveur vérifie l'en-tête conditionnel `If-None-Match`. Constatant que la ressource n'a pas été modifiée depuis la génération de cet ETag, il est inutile de renvoyer à nouveau les données. Le statut 304 ordonne au client de réutiliser la copie déjà présente dans son cache local.

### 2. Intérêts concrets
- **Pour une application mobile :**
  - **Économie de données :** Réduit considérablement la consommation du forfait data de l'utilisateur.
  - **Autonomie de batterie :** L'activation des modules radio 4G/5G est brève et peu énergivore car aucun téléchargement lourd n'a lieu.
  - **Fluidité / Réactivité :** Réponse quasi instantanée pour l'utilisateur.
- **Pour le quota GitHub :**
  - Les requêtes conditionnelles répondant `304 Not Modified` évitent d'impacter le quota classique des requêtes complètes, préservant la bande passante et la charge serveur de l'infrastructure GitHub.

---

## Étape 3 — Les erreurs

### 1. Tableau comparatif des réponses d'erreur

| API | Requête | Code statut | Format du corps | Information utile pour corriger |
| :--- | :--- | :--- | :--- | :--- |
| **GitHub** | `GET /repos/nodejs/ce-depot-n-existe-pas` | `404 Not Found` | JSON (`message`, `documentation_url`, `status`) | Fournit le lien vers la documentation de l'endpoint pour vérifier l'URI et les permissions. |
| **GitHub** | `DELETE /repos/nodejs/node` | `401 Unauthorized` | JSON (`message`, `documentation_url`, `status`) | Indique explicitement que l'action requiert une authentification préalable (`"Requires authentication"`). |
| **geo.api.gouv.fr** | `GET /communes?codePostal=abc` | `200 OK` | JSON (tableau vide `[]`) | **Aucune.** Répond avec un statut de succès, masquant l'erreur de formatage du client. |

### 2. Erreur ne respectant pas la sémantique HTTP
- **L'anomalie :** `geo.api.gouv.fr` renvoyant `200 OK` avec `[]`.
- **Pourquoi c'est un anti-pattern :**  
  Le code postal `abc` est syntaxiquement et sémantiquement invalide (un code postal français est constitué de 5 chiffres). En renvoyant un code `200 OK`, l'API affirme que la requête est valide et qu'il n'existe simplement aucune commune associée, au lieu de signaler une erreur client.  
  L'API aurait dû renvoyer une erreur client :
  - `400 Bad Request` (requête mal formée) ou idéalement `422 Unprocessable Content` (paramètre lisible mais invalide au regard des règles métier).

### 3. Réécriture de l'erreur au format RFC 9457 (*Problem Details for HTTP APIs*)

```http
HTTP/1.1 422 Unprocessable Content
Content-Type: application/problem+json

{
  "type": "https://geo.api.gouv.fr/problems/invalid-parameter",
  "title": "Paramètre de requête invalide",
  "status": 422,
  "detail": "Le paramètre 'codePostal' doit être composé exactement de 5 chiffres décimaux.",
  "instance": "/communes?codePostal=abc",
  "invalid_params": [
    {
      "name": "codePostal",
      "value": "abc",
      "reason": "Format non conforme : 5 chiffres attendus (ex: 75001)."
    }
  ]
}
```

> **Point de contrôle 2 validé :**
> - Les erreurs client (4xx) exigent une correction côté client avant rejeu.
> - Les erreurs serveur (5xx) indiquent un incident serveur temporaire, rejouable plus tard.
> - La réécriture suit scrupuleusement la RFC 9457 (`type`, `title`, `status`, `detail`, `instance`).

---

## Étape 4 — Collections et pagination

### 1. Filtrage et sélection de champs sur geo.api.gouv.fr
- **Filtrer une collection :** Via des paramètres d'URL dans la query string (ex: `?nom=Lyon`).
- **Projeter / restreindre les champs retournés (*sparse fieldsets*) :** Via le paramètre `fields` (ex: `?fields=nom,code,population`).

### 2. Localisation de l'information de pagination chez GitHub
- L'information se trouve dans les **en-têtes HTTP de réponse**, et non dans le corps JSON.
- **En-tête utilisé :** L'en-tête standard `Link` (RFC 8288 / RFC 5988).  
  Exemple observé :  
  `Link: <https://api.github.com/repositories/27193779/issues?per_page=5&after=...&page=2>; rel="next"`

### 3. Avantages de placer la pagination dans un en-tête
1. **Intégrité de la charge utile (*payload*) :** Le corps de réponse reste un tableau JSON propre (`[...]`), sans surcouche d'enveloppe technique (`{ "data": [...], "pagination": {...} }`).
2. **Agnostique au format :** Le mécanisme fonctionne de manière universelle, que le format demandé soit du JSON, du XML, du CSV ou un binaire.
3. **Inspection légère via `HEAD` :** Un client ou un proxy peut inspecter les liens de pagination via une requête `HEAD` sans télécharger tout le contenu de la collection.

---

## Étape 5 — Négociation de contenu

Commandes exécutées :
```powershell
curl.exe -s -H "Accept: application/vnd.github.raw+json" https://api.github.com/repos/nodejs/node/readme
curl.exe -s -H "Accept: application/vnd.github.html+json" https://api.github.com/repos/nodejs/node/readme
```

### 1. Quel en-tête choisit le format ?
- C'est l'en-tête de requête **`Accept`**.  
  Ici, GitHub utilise des types de médias personnalisés (*vendor-specific media types*) pour déterminer si le client désire le markdown brut (`application/vnd.github.raw+json`) ou le HTML pré-rendu (`application/vnd.github.html+json`).

### 2. Statut en cas de format indisponible
- Si le serveur ne peut honorer aucun des formats spécifiés dans l'en-tête `Accept`, il doit renvoyer le code **`406 Not Acceptable`**.

---

## Étape 6 — L’échelle de maturité de Richardson

L'échelle de Richardson mesure le degré d'adhésion d'une API aux principes du Web et du protocole HTTP :
- **Niveau 0 (The Swamp of POX) :** Un endpoint unique (ex: `/api`), transport HTTP comme simple tunnel, verbe unique (POST), logique dans le corps (ex: SOAP, XML-RPC).
- **Niveau 1 (Ressources) :** Des URI distinctes identifient chaque ressource (ex: `/orders/42`), mais les méthodes HTTP restent sous-exploitées (tout en POST ou GET).
- **Niveau 2 (Verbes et statuts HTTP) :** Exploitation rigoureuse de HTTP : GET pour lire, POST pour créer, PATCH pour modifier partiellement, DELETE pour supprimer. Utilisation correcte des codes de statut (200, 201, 204, 404, 422...). C'est le standard de l'industrie et l'objectif d'Orbitale.
- **Niveau 3 (Hypermédia / HATEOAS) :** Les représentations intègrent des liens vers les transitions d'état possibles, guidant dynamiquement le client sans connaissance préalable des URI.

### Classement des API étudiées

1. **GitHub REST API — Niveau 2 (avec éléments précurseurs du niveau 3) :**
   - *Justification :* GitHub utilise des URI de ressources propres (`/repos/{owner}/{repo}`), l'ensemble des verbes HTTP (GET, POST, PATCH, DELETE) et des codes de statut précis (200, 201, 204, 304, 401, 404). Elle utilise l'en-tête `Link` pour la pagination (prémisse de niveau 3), mais n'expose pas un moteur d'hypermédia complet dans l'ensemble de ses ressources pour piloter les transitions d'état métier.
2. **geo.api.gouv.fr — Niveau 1 :**
   - *Justification :* Les entités sont bien adressées par des URI distinctes (`/communes`, `/departements`), mais l'API est en lecture seule (GET uniquement), n'exploite pas la diversité des verbes HTTP, et s'écarte de la sémantique des statuts d'erreur en renvoyant `200 OK` avec un tableau vide lors d'un paramètre erroné.
3. **Stripe API — Niveau 2 (très haute maturité) :**
   - *Justification :* Architecture axée sur des ressources claires (`/v1/customers`, `/v1/charges`), utilisation complète des méthodes (POST pour créer, DELETE pour supprimer une carte), statuts HTTP stricts, mise en place native de la réentrance et de l'idempotence via l'en-tête `Idempotency-Key`.

> **Point de contrôle 3 validé :**  
> Chaque classement repose sur une observation concrète de requêtes réelles.

---

## ROCKET Bonus

### 1. Hypermédia (Niveau 3) sur une commande Orbitale
Exemple de représentation d'une commande selon son état (`pending` vs `paid`) :

**État 1 : Commande en attente de règlement (`pending`)**
```json
{
  "id": "ord_89f1a",
  "status": "pending",
  "totalAmount": 45.00,
  "currency": "EUR",
  "_links": {
    "self": { "href": "/v1/orders/ord_89f1a", "method": "GET" },
    "payment": { "href": "/v1/orders/ord_89f1a/payments", "method": "POST" },
    "cancel": { "href": "/v1/orders/ord_89f1a", "method": "PATCH" }
  }
}
```

**État 2 : Commande réglée (`paid`)**
```json
{
  "id": "ord_89f1a",
  "status": "paid",
  "totalAmount": 45.00,
  "currency": "EUR",
  "_links": {
    "self": { "href": "/v1/orders/ord_89f1a", "method": "GET" },
    "tickets": { "href": "/v1/orders/ord_89f1a/tickets", "method": "GET" },
    "invoice": { "href": "/v1/orders/ord_89f1a/invoice", "method": "GET" }
  }
}
```
*Le lien `payment` a disparu dynamiquement car la commande est déjà payée ; les liens vers les billets et la facture sont devenus disponibles.*

### 2. Comparaison avec les *RESTful API Guidelines* de Zalando
Deux règles notables :
- **Règle #136 : Pluralize Resource Names** : Toutes les collections doivent porter des noms au pluriel (`/v1/events`, `/v1/orders`).
- **Règle #176 : Use RFC 9457 for Error Formats** : Toutes les erreurs d'API doivent être modélisées avec le Content-Type `application/problem+json`.

### 3. Requête GraphQL équivalente et impact sur le cache HTTP
**Requête GraphQL :**
```graphql
query GetEventDetails($id: ID!) {
  event(id: $id) {
    id
    title
    remainingTickets
    nextDates(limit: 3) {
      startAt
      venue
    }
  }
}
```
**Impact sur le cache HTTP :**  
- En REST, un `GET /v1/events/{id}` dispose d'une URL unique mise en cache facilement par les proxys, passerelles et navigateurs via `Cache-Control` et `ETag`.
- En GraphQL, la plupart des requêtes transitent en `POST /graphql`. Par défaut, la méthode `POST` n'est pas mise en cache par les couches HTTP intermédiaires. La mise en cache nécessite alors un mécanisme applicatif complexe ou l'emploi de requêtes persistées (*Persisted Queries*) converties en GET.
