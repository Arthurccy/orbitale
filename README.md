# Orbitale — API RESTful de Billetterie & Contrôle d'Accès

Dépôt officiel du projet fil rouge **Orbitale** (M2 Web Services — Lydia KAHOUADJI).

Orbitale est une plateforme de billetterie d'événements et de spectacles conçue selon une architecture orientée services (SOA / RESTful), desservant trois types de clients :
1. **Front-office Web** (catalogue public, commande de billets)
2. **Application Mobile** (consultation de billets dématérialisés)
3. **Bornes physiques d'entrée** (contrôle d'accès et scan de QR codes)

---

## 📂 Structure du dépôt et livrables

```text
orbitale/
├── .spectral.yaml          # Règles de gouvernance d'API & règle maison "orbitale-pas-de-verbe"
├── openapi.yaml            # Contrat d'API standardisé OpenAPI 3.1.0 (source de vérité)
├── README.md               # Documentation générale du projet
└── docs/
    ├── notes_s1.md         # Livrable S1 : Réponses détaillées au TP guidé (HTTP, Cache, Erreurs, Richardson)
    ├── ressources.md       # Livrable S1 : Modèle de ressources, relations (Mermaid), endpoints & choix de design
    └── mock.png            # Livrable S2 : Capture d'écran des tests d'exécution contre le mock Prism
```

---

## 🎯 Livrables par Séance

### Séance 1 — Fondamentaux REST & Styles d'API
- [📘 `docs/notes_s1.md`](docs/notes_s1.md) :
  - **Anatomie HTTP** : Décorticage des requêtes/réponses réelles (`curl -v`), version HTTP/1.1, rôle des en-têtes (`Content-Type`, `Cache-Control`, `ETag`, `X-RateLimit-Remaining`).
  - **Cache conditionnel** : Validation de `If-None-Match` renvoyant `304 Not Modified` avec corps vide ; intérêt pour la bande passante mobile et les quotas.
  - **Gestion des erreurs & RFC 9457** : Analyse comparative GitHub vs geo.api.gouv.fr, mise en évidence de l'anti-pattern `200 OK []` et réécriture standardisée `application/problem+json`.
  - **Pagination** : En-tête standard `Link` (RFC 8288) et négociation de contenu (`Accept`).
  - **Échelle de maturité de Richardson** : Analyse critique des niveaux 0 à 3 et positionnement d'Orbitale au **Niveau 2**.
- [🏛️ `docs/ressources.md`](docs/ressources.md) :
  - Modélisation des 5 ressources cœur : **Événement**, **Utilisateur**, **Commande**, **Billet**, **Scan**.
  - Diagramme Entité-Relation Mermaid avec cardinalités.
  - Tableau des endpoints `/v1/...` (méthodes, succès `200`/`201`/`204`, erreurs pertinentes `400`/`404`/`409`/`422`).
  - Défense argumentée des 3 choix clés de conception pour la soutenance (Pourquoi `PATCH` vs `PUT`, annulation sans verbe d'action, et création de la ressource `Scan` pour les bornes).

### Séance 2 — Design-First avec OpenAPI
- [📜 `openapi.yaml`](openapi.yaml) :
  - Contrat complet en **OpenAPI 3.1.0** couvrant l'ensemble des opérations (Événements, Commandes, Billets, Scans).
  - Principe *« Lire n'est pas écrire »* : séparation stricte de la représentation lue (`Event`) et des schémas d'écriture (`EventCreate`, `EventPatch`) avec `additionalProperties: false`.
  - Champs calculés et horodatages en `readOnly: true`.
  - Montants financiers exprimés en centimes entiers (`3900` pour 39,00 €).
  - Schémas d'erreurs partagés via `$ref` vers `Problem` et `ValidationProblem` en `application/problem+json`.
  - Code de billet sécurisé validé par regex (`pattern: '^ORB-[0-9A-F]{12}$'`).
- [⚙️ `.spectral.yaml`](.spectral.yaml) :
  - Règles de style et de gouvernance d'API.
  - **Règle maison `orbitale-pas-de-verbe`** : Interdit tout verbe d'action dans les chemins d'URI (`/createEvent`, `/getOrders`, etc.).
  - **Règle bonus `orbitale-param-id`** : Impose que tout paramètre de chemin se termine par `Id` (`eventId`, `orderId`, `ticketId`).
- [🖼️ `docs/mock.png`](docs/mock.png) :
  - Capture attestant de la simulation du serveur mock Prism sur les cas `200 OK`, `201 Created` (avec `Location`), `422 Unprocessable Entity` (rejet strict) et `404 Not Found`.

---

## 🚀 Guide d'Exécution & Vérification

### 1. Valider le contrat avec le linter Spectral
```bash
# Vérification complète des règles de style (0 erreur attendue)
npx --yes @stoplight/spectral-cli@6 lint openapi.yaml
```

### 2. Démarrer le serveur Mock (Prism)
Le mock Prism permet aux équipes clientes (front web et mobile) de développer immédiatement sans attendre l'implémentation finale du backend :
```bash
# Lancement du mock sur le port 4010
npx --yes @stoplight/prism-cli@5 mock openapi.yaml -p 4010
```
*Alternative avec Docker :*
```bash
docker run --rm -p 4010:4010 -v "${PWD}/openapi.yaml:/tmp/openapi.yaml" stoplight/prism:5 mock -h 0.0.0.0 -p 4010 -m false /tmp/openapi.yaml
```

### 3. Tester le Mock avec cURL

```bash
# 1. Lecture d'un événement (200 OK)
curl -i http://localhost:4010/v1/events/6f1c2b8e-4f3a-4e8b-9a51-2d7c0f9e1a23

# 2. Création valide d'un événement (201 Created + en-tête Location)
curl -i -X POST http://localhost:4010/v1/events \
  -H "Content-Type: application/json" \
  -d '{"title":"Nuit électro","venue":"Le Zénith, Paris","startsAt":"2027-03-19T20:00:00.000Z","capacity":6000,"price":3900,"currency":"EUR","status":"published"}'

# 3. Création invalide interceptée par le contrat (422 Unprocessable Entity)
curl -i -X POST http://localhost:4010/v1/events \
  -H "Content-Type: application/json" \
  -d '{"title":"x"}'

# 4. Forcer un cas d'erreur 404 (404 Not Found au format RFC 9457)
curl -i -H "Prefer: code=404" http://localhost:4010/v1/events/6f1c2b8e-4f3a-4e8b-9a51-2d7c0f9e1a23
```

---

## 🎓 Principes d'Architecture à Défendre en Soutenance

1. **L'approche Design-First :** Le contrat est rédigé et validé collectivement avant d'écrire une seule ligne de code serveur. Le mock Prism débloque immédiatement les équipes clientes.
2. **Interface uniforme & orientation ressource :** L'URI nomme la ressource (`/v1/events`), le verbe HTTP porte l'intention. Aucune action procédurale n'apparaît dans l'URL.
3. **Le contrôle d'accès matérialisé en ressource :** Les bornes effectuent `POST /v1/scans` (création d'un événement de pointage) au lieu d'un verbe `/validateTicket`, garantissant l'audit trail, la traçabilité temporelle et la détection des fraudes.
4. **Mutation d'état via `PATCH` :** L'annulation d'une commande s'opère par `PATCH /v1/orders/{id}` avec `{"status":"cancelled"}`, évitant la surcharge et les écrasements concurrents de `PUT`.
5. **Erreurs standardisées (RFC 9457) :** Toutes les erreurs 4xx/5xx répondent en `application/problem+json` avec des propriétés exploitables par les machines (`type`, `status`) et claires pour les humains (`title`, `detail`).
