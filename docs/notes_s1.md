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
- **Version HTTP utilisée :** `HTTP/1.1`, négociée via TLS ALPN (`* ALPN: curl offers http/1.1` puis `server accepted http/1.1`).  
  Ce n'est pas un choix de GitHub, qui accepte HTTP/2 : le `curl.exe` livré avec Windows (build Schannel) ne propose que `http/1.1`. Avec un curl compilé avec HTTP/2, la même requête obtient `HTTP/2 200`.

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
  - D'après la documentation GitHub, une réponse `304 Not Modified` n'est **pas décomptée** du quota principal, **à condition que la requête soit authentifiée** (en-tête `Authorization`).
  - **Vérifié en non authentifié :** trois requêtes conditionnelles successives ont toutes répondu `304`, et `X-RateLimit-Remaining` est passé de 55 à 54 puis à 53. Sans jeton, le `304` consomme donc bien le quota.
  - Même sans jeton, le `304` reste utile : il économise la bande passante (aucun corps transféré) et la charge de sérialisation côté serveur.

---

## Étape 3 — Les erreurs

### 1. Tableau comparatif des réponses d'erreur

| API | Requête | Code statut | Classe | Format du corps | Information utile pour corriger |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **GitHub** | `GET /repos/nodejs/ce-depot-n-existe-pas` | `404 Not Found` | 4xx, erreur client | JSON (`message`, `documentation_url`, `status`) | Fournit le lien vers la documentation de l'endpoint pour vérifier l'URI et les permissions. |
| **GitHub** | `DELETE /repos/nodejs/node` | `401 Unauthorized` | 4xx, erreur client | JSON (`message`, `documentation_url`, `status`) | Indique explicitement que l'action requiert une authentification préalable (`"Requires authentication"`). |
| **geo.api.gouv.fr** | `GET /communes?codePostal=abc` | `200 OK` | 2xx (devrait être 4xx) | JSON (tableau vide `[]`) | **Aucune.** Répond avec un statut de succès, masquant l'erreur de formatage du client. |

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

> **Point de contrôle 2 :**
> - Les trois erreurs observées sont des erreurs client (4xx, ou 2xx à tort) : la requête doit être corrigée avant d'être rejouée.
> - Aucune erreur serveur (5xx) n'a été observée. Un `500`, `502` ou `503` signalerait un incident côté serveur, que le client peut rejouer plus tard sans rien changer (en respectant `Retry-After` s'il est présent).
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

1. **GitHub REST API — Niveau 2, avec des éléments de niveau 3 :**
   - *Requêtes observées :* `GET /repos/nodejs/node` → `200` avec `ETag`, puis `304` avec `If-None-Match` ; `DELETE /repos/nodejs/node` sans jeton → `401` (la méthode DELETE est reconnue sur la même URI) ; `GET /repos/nodejs/ce-depot-n-existe-pas` → `404`.
   - *Justification :* une URI par ressource, la méthode porte l'intention et le statut porte le résultat : c'est le niveau 2. Des éléments de niveau 3 existent : le corps de `GET /repos/nodejs/node` contient des liens vers les ressources liées (`issues_url`, `pulls_url`, `contributors_url`…), et la pagination passe par l'en-tête `Link` (`rel="next"`). Il ne s'agit pas d'un vrai HATEOAS : les liens ne dépendent pas de l'état et n'annoncent pas les actions possibles.
2. **geo.api.gouv.fr — Niveau 2 en lecture, avec un écart :**
   - *Requêtes observées :* `GET /communes/00000` → `404 Not Found` ; `GET /communes?codePostal=abc` → `200 OK` avec `[]`.
   - *Justification :* les ressources ont des URI distinctes (`/communes`, `/departements`, `/communes/{code}`), et le `GET` est utilisé à bon escient avec un vrai `404` pour une ressource inexistante. Être en lecture seule n'empêche pas le niveau 2 : il n'y a simplement pas d'autre méthode à exposer. Seul écart, le `200 []` sur un filtre invalide, qui aurait dû être un `400` ou un `422` (étape 3).
3. **Stripe API — Niveau 2 :**
   - *Requêtes observées :* `GET https://api.stripe.com/v1/customers` sans clé → `401` avec un corps JSON `{"error": {"type": "invalid_request_error", "message": "You did not provide an API key..."}}` ; `DELETE /v1/customers/cus_x` sans clé → `401`.
   - *Justification :* URI versionnées par ressource (`/v1/customers`, `/v1/charges`), statuts HTTP significatifs et erreurs structurées. Nuance : Stripe modifie ses ressources en `POST /v1/customers/{id}` et non en `PATCH`, un écart assumé par rapport à la sémantique HTTP. Stripe gère aussi l'idempotence des créations via l'en-tête `Idempotency-Key`, que nous reprenons pour `POST /v1/orders`.

> **Point de contrôle 3 :**  
> Chaque classement s'appuie sur des requêtes réellement exécutées (URI, méthode et statut cités ci-dessus).

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
Deux règles qu'Orbitale **ne respecte pas encore** :
- **Règle #115 — MUST not use URL versioning :** Zalando interdit la version dans l'URI et recommande le versionnage par type de média (#114, ex. `Accept: application/x.orbitale.event+json;version=1`). Orbitale utilise `/v1/...`, comme l'exige le cours. C'est un choix assumé : il rend la version visible et facile à router, au prix d'URI qui changent à chaque version majeure.
- **Règle #118 — MUST property names must be snake_case (and never camelCase) :** Orbitale utilise le camelCase (`startsAt`, `totalAmount`, `buyerEmail`). Pour se conformer, il faudrait écrire `starts_at`, `total_amount`, `buyer_email`.

Règles déjà respectées : #134 (noms de ressources au pluriel) et #176 (erreurs en `application/problem+json`).

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
