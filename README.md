# Webhook Delivery Service

**Webhook Delivery Service** est un service backend de distribution fiable de webhooks, conçu pour envoyer des événements métier vers des endpoints tiers de manière asynchrone, sécurisée, observable et rejouable.

Il est développé avec **Symfony 7.4**, **PostgreSQL**, **Redis**, **Symfony Messenger** et **Docker Compose**.

> Envoyez un événement une seule fois. Le service garantit sa distribution fiable vers les bons destinataires, avec retries, signatures HMAC, historique complet et replay.

---

## Pourquoi ce projet ?

Dans une architecture distribuée, une application doit souvent notifier des systèmes tiers lorsqu’un événement métier se produit :

- Un paiement est confirmé.
- Une commande est expédiée.
- Un utilisateur est créé.
- Une facture est payée.
- Un stock est modifié.

Un simple appel HTTP ne suffit pas : le destinataire peut être indisponible, lent, saturé ou retourner une erreur.

Ce projet résout ce problème en fournissant une infrastructure de livraison d’événements avec :

- Traitement asynchrone via une file Redis.
- Retries automatiques avec backoff exponentiel.
- Dead-letter queue pour les échecs définitifs.
- Replay des livraisons échouées.
- Signature HMAC-SHA256 des payloads.
- Idempotence côté producteur et consommateur.
- Journalisation structurée et métriques de livraison.
- Routage d’événements et transformation des payloads.

---

## Fonctionnalités

### Cœur du service

- API REST JSON pour la gestion des endpoints et l’émission d’événements.
- Traitement asynchrone avec Symfony Messenger et Redis.
- Livraison HTTP avec timeout configurable.
- Politique de retry avec backoff exponentiel et jitter.
- Dead-letter queue après épuisement des retries.
- Replay d’une livraison individuelle ou par lot.
- Historique complet des tentatives de livraison.
- Signature HMAC-SHA256 des payloads.
- Idempotence via `Idempotency-Key`.
- Isolation multi-tenant.
- Environnement conteneurisé avec Docker Compose.
- Tests automatisés.

### Fonctionnalités différenciantes

- **Event Router** : routage des événements vers un ou plusieurs endpoints selon le type d’événement et des conditions sur le payload.
- **Payload Transformer** : transformation du payload avant livraison, sans modifier le code du producteur.
- **Circuit Breaker** : mise en quarantaine temporaire d’un endpoint défaillant.
- **Sandbox / Chaos Mode** : simulation d’erreurs HTTP, de latence et de timeouts pour tester la résilience.
- **Observabilité** : logs structurés, métriques de livraison et historique par endpoint.
- **Diagnostic intelligent** : analyse des échecs et recommandation d’action.

---

## Architecture

```text
Producteur
   │
   ▼
API Symfony
   │
   ├── Authentification par clé API
   ├── Validation et idempotence
   ├── Event Router
   ├── Payload Transformer
   ▼
PostgreSQL
   │
   ▼
Symfony Messenger
   │
   ▼
Redis Streams
   │
   ▼
Worker Symfony
   │
   ├── Circuit Breaker
   ├── Signature HMAC-SHA256
   ├── Client HTTP
   ▼
Endpoint consommateur
   │
   ▼
Delivery attempts / Dead-letter queue / Replay
   │
   ▼
Logs structurés / Métriques / Tracing
```

### Composants

| Composant | Rôle |
|---|---|
| Symfony | API REST, validation, sécurité et orchestration |
| PostgreSQL | Source de vérité : tenants, endpoints, événements, livraisons et tentatives |
| Redis | Transport asynchrone Symfony Messenger |
| Symfony Messenger | File de messages, retries et failure transport |
| Docker Compose | Environnement reproductible en local |
| FrankenPHP / Caddy | Runtime PHP et serveur web |

---

## Stack technique

- PHP 8.3
- Symfony 7.4 LTS
- Symfony Messenger
- Symfony Doctrine ORM
- PostgreSQL 16
- Redis 7
- Docker Compose
- FrankenPHP / Caddy
- PHPUnit
- GitHub Actions

---

## Démarrage rapide

### Prérequis

- Docker Desktop ou Docker Engine
- Docker Compose v2.10 ou supérieur
- Make optionnel

### Installation

```bash
git clone [https://github.com/](https://github.com/)<ton-compte>/webhook-delivery.git
cd webhook-delivery

cp .env.example .env

docker compose build --pull --no-cache
docker compose up --wait
```

L’application est disponible sur :

```text
[https://localhost](https://localhost)
```

### Vérifier l’installation

```bash
docker compose exec php php bin/console --version
```

Tu dois obtenir une version Symfony 7.4.x.

### Lancer le worker

```bash
docker compose exec php php bin/console messenger:consume webhook_delivery -vv
```

---

## Variables d’environnement

Exemple de `.env` :

```env
APP_ENV=dev
APP_SECRET=change-me

DATABASE_URL=postgresql://app:!ChangeMe!@database:5432/webhook_delivery?serverVersion=16&charset=utf8

REDIS_HOST=redis

MESSENGER_TRANSPORT_DSN=redis://redis:6379/webhook_delivery
MESSENGER_FAILURE_TRANSPORT_DSN=redis://redis:6379/webhook_failed
```

---

## API principale

### Créer un endpoint

```bash
curl -X POST [https://localhost/api/endpoints](https://localhost/api/endpoints) \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Comptabilité",
    "url": "[https://receiver.example.com/webhooks](https://receiver.example.com/webhooks)",
    "event_types": ["payment.succeeded"],
    "timeout_seconds": 5
  }'
```

### Émettre un événement

```bash
curl -X POST [https://localhost/api/events](https://localhost/api/events) \
  -H "Authorization: Bearer $API_KEY" \
  -H "Idempotency-Key: payment_123" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "payment.succeeded",
    "occurred_at": "2026-10-28T12:00:00Z",
    "data": {
      "payment_id": "pay_123",
      "amount": 25000,
      "currency": "XOF"
    }
  }'
```

Réponse :

```json
{
  "event_id": "evt_01J...",
  "status": "accepted",
  "deliveries_created": 1,
  "idempotent": false
}
```

### Consulter une livraison

```bash
curl [https://localhost/api/deliveries/{deliveryId}](https://localhost/api/deliveries/{deliveryId}) \
  -H "Authorization: Bearer $API_KEY"
```

### Rejouer une livraison échouée

```bash
curl -X POST [https://localhost/api/deliveries/{deliveryId}/replay](https://localhost/api/deliveries/{deliveryId}/replay) \
  -H "Authorization: Bearer $API_KEY"
```

---

## Politique de retry

Par défaut, une livraison échouée est retentée selon la stratégie suivante :

| Paramètre | Valeur |
|---|---|
| Tentatives maximales | 6 |
| Délai initial | 30 secondes |
| Multiplicateur | 2 |
| Délai maximum | 1 heure |
| Jitter | 10 % |

Exemple de séquence :

```text
30 s → 1 min → 2 min → 4 min → 8 min → 16 min
```

### Comportement selon l’erreur

| Erreur | Comportement |
|---|---|
| 2xx | Livraison réussie |
| 429 | Retry |
| 500, 502, 503, 504 | Retry |
| Timeout ou erreur réseau | Retry |
| 400, 401, 403, 404 | Échec définitif |
| Échecs après toutes les tentatives | Dead-letter queue |

---

## Signature HMAC

Chaque webhook est signé avec HMAC-SHA256.

En-têtes envoyés :

```http
Content-Type: application/json
X-Webhook-Id: evt_01J...
X-Webhook-Delivery-Id: dlv_01J...
X-Webhook-Event-Type: payment.succeeded
X-Webhook-Timestamp: 1769649600
X-Webhook-Signature: v1=...
X-Webhook-Signature-Version: v1
```

La signature est calculée sur le **corps brut exact** de la requête, en incluant le timestamp.

Côté consommateur, la vérification doit :

- Recalculer la signature avec le secret partagé.
- Comparer les signatures avec une comparaison à temps constant.
- Rejeter les requêtes dont le timestamp est trop ancien.
- Traiter le payload uniquement après validation.

---

## Idempotence

Le producteur peut fournir un en-tête `Idempotency-Key` :

```http
Idempotency-Key: payment_123
```

Si la même clé est envoyée deux fois par le même tenant, le service retourne le même événement sans créer de doublon.

```json
{
  "event_id": "evt_01J...",
  "status": "accepted",
  "deliveries_created": 1,
  "idempotent": true
}
```

---

## Event Router

Une règle de routage permet d’envoyer un événement vers un endpoint selon des conditions métier.

```json
{
  "name": "Paiements importants vers comptabilité",
  "event_types": ["payment.succeeded"],
  "conditions": {
    "data.amount_gte": 10000
  },
  "endpoint_id": "ep_accounting",
  "priority": "high",
  "is_active": true
}
```

Un événement peut donc déclencher :

- Aucune livraison.
- Une seule livraison.
- Plusieurs livraisons vers des endpoints différents.

---

## Payload Transformer

Chaque endpoint peut recevoir un payload transformé.

Template :

```json
{
  "template": {
    "event": "{{ event.type }}",
    "payment_id": "{{ event.data.payment_id }}",
    "amount": "{{ event.data.amount }}",
    "currency": "{{ event.data.currency }}"
  }
}
```

Payload source :

```json
{
  "type": "payment.succeeded",
  "data": {
    "payment_id": "pay_123",
    "amount": 25000,
    "currency": "XOF"
  }
}
```

Payload livré :

```json
{
  "event": "payment.succeeded",
  "payment_id": "pay_123",
  "amount": 25000,
  "currency": "XOF"
}
```

---

## Circuit Breaker

Le service protège les endpoints défaillants grâce à un circuit breaker.

| État | Comportement |
|---|---|
| `closed` | Livraisons normales |
| `open` | Livraisons suspendues temporairement |
| `half_open` | Quelques requêtes tests sont autorisées |

Lorsqu’un endpoint échoue trop souvent :

1. Le circuit passe en `open`.
2. Les nouvelles livraisons sont mises en attente.
3. Après la période d’ouverture, le circuit passe en `half_open`.
4. Si les requêtes tests réussissent, le circuit revient en `closed`.
5. Sinon, il repasse en `open`.

---

## Sandbox et Chaos Mode

Le projet inclut un récepteur de test permettant de simuler des comportements défaillants.

```http
POST /sandbox/endpoints/{id}/failure-rules
```

```json
{
  "fail_first_requests": 3,
  "http_status": 500,
  "latency_ms": 3000,
  "simulate_timeout": false
}
```

Cela permet de démontrer en direct :

1. L’émission d’un événement.
2. L’échec volontaire du receiver.
3. Les retries exponentiels.
4. L’ouverture du circuit breaker.
5. La correction du receiver.
6. Le replay réussi de la livraison.

---

## Observabilité

### Logs structurés

Chaque tentative est journalisée au format JSON :

```json
{
  "timestamp": "2026-10-28T12:00:05Z",
  "level": "error",
  "message": "webhook_delivery_failed",
  "delivery_id": "dlv_01J...",
  "event_id": "evt_01J...",
  "endpoint_id": "ep_01J...",
  "tenant_id": "ten_01J...",
  "attempt": 2,
  "http_status": 503,
  "duration_ms": 1820,
  "error": "HTTP 503 Service Unavailable"
}
```

### Métriques

Le service expose notamment :

- Nombre d’événements reçus.
- Nombre de livraisons créées.
- Nombre de livraisons réussies.
- Nombre de retries.
- Nombre de dead letters.
- Taux de succès par endpoint.
- Latence p50, p95 et p99.
- Nombre d’endpoints en circuit breaker ouvert.

---

## Tests

Lancer la suite de tests :

```bash
docker compose exec php php bin/phpunit
```

### Couverture testée

- Authentification par clé API.
- Création et validation des endpoints.
- Émission d’événements.
- Idempotence.
- Routage des événements.
- Transformation des payloads.
- Signature HMAC.
- Retries et backoff exponentiel.
- Dead-letter queue.
- Replay.
- Circuit breaker.

---

## Structure du projet

```text
.
├── assets/
├── config/
│   ├── packages/
│   ├── routes/
│   └── services.yaml
├── migrations/
├── public/
├── src/
│   ├── Controller/
│   ├── Entity/
│   ├── Message/
│   ├── MessageHandler/
│   ├── Repository/
│   ├── Router/
│   ├── Security/
│   ├── Service/
│   ├── Signature/
│   └── Transformer/
├── tests/
├── compose.yaml
├── compose.override.yaml
├── Dockerfile
└── README.md
```

---

## Roadmap

- [x] Initialisation Symfony, Docker, PostgreSQL et Redis.
- [ ] Gestion des tenants et endpoints.
- [ ] Émission d’événements avec idempotence.
- [ ] Livraison asynchrone avec Symfony Messenger.
- [ ] Retries exponentiels et dead-letter queue.
- [ ] Signature HMAC-SHA256.
- [ ] Replay des livraisons.
- [ ] Event Router.
- [ ] Payload Transformer.
- [ ] Circuit Breaker.
- [ ] Sandbox et Chaos Mode.
- [ ] Métriques Prometheus et dashboard Grafana.
- [ ] Portail développeur.
- [ ] Diagnostic intelligent des échecs.

---

## Décisions techniques

### Pourquoi Symfony Messenger ?

Symfony Messenger fournit une abstraction propre pour les messages asynchrones, les retries, les failure transports et les workers. Il permet de construire une architecture événementielle sans réinventer la gestion des files.

### Pourquoi Redis ?

Redis offre un transport rapide et léger pour Symfony Messenger, adapté au traitement des livraisons de webhooks. PostgreSQL reste la source de vérité pour l’historique, les livraisons et les tentatives.

### Pourquoi PostgreSQL ?

PostgreSQL garantit la persistance fiable des événements, des endpoints, des livraisons et de l’historique des tentatives. Il permet également des requêtes d’audit et d’observabilité efficaces.

### Pourquoi HMAC-SHA256 ?

HMAC-SHA256 permet au consommateur de vérifier que le webhook provient bien du service et que le payload n’a pas été modifié. La signature est calculée sur le corps brut et associée à un timestamp pour limiter les risques de rejeu.

---

## Auteur

Développé par **[Ton nom]**  
Développeur backend PHP/Symfony, orienté architecture distribuée, cloud, DevOps et IA.

- GitHub : [ton-compte](https://github.com/ton-compte)
- LinkedIn : [ton-profil](https://www.linkedin.com/in/ton-profil)
- Portfolio : [ton-site](https://ton-site.com)

---

## Licence

Ce projet est distribué sous licence MIT.
