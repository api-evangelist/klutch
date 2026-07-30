---
name: Klutch Mini App webhook handler
description: Receive Klutch transaction webhooks, re-read the transaction over GraphQL, enrich or react to it, and render the result back into the Klutch app as a Mini App panel.
api: graphql/klutch-graphql-operations.yml
endpoint: https://graphql.klutchcard.com/graphql
operations:
  - createSessionToken
  - transaction
  - transaction.item.change
  - transactionCategories
  - recipeInstall.createToken
  - recipeInstall.addPanel
  - recipeInstall.panel.changeData
generated: '2026-07-19'
method: generated
source: asyncapi/klutch-webhooks.yml, https://github.com/KlutchCard/api-samples
---

# Klutch Mini App webhook handler

Klutch pushes transaction events to a URL registered in the Klutch account UI under **My Account -> Developers**. There is no subscription API.

## 1. Accept and filter the event

The payload looks like this:

```json
{
  "principal": { "entityID": "<<USERID>>" },
  "event": {
    "_alloyCardType": "com.alloycard.core.entities.transaction.TransactionCreatedEvent",
    "transaction": { "entityID": "txn_..." }
  }
}
```

Switch on `event._alloyCardType` and **ignore anything you do not recognize** — the catalog is not exhaustive. The two documented types are:

- `com.alloycard.core.entities.transaction.TransactionCreatedEvent` — an authorization was created (approved or declined)
- `com.alloycard.core.entities.transaction.TransactionItemCreatedEvent` — a line item was added to a transaction

Note the legacy `alloycard` namespace: it is correct, not a typo.

Respond 2xx quickly and do the work asynchronously. Klutch documents no retry policy and no signature verification, so treat the endpoint as unauthenticated: validate that the referenced transaction actually belongs to the `principal.entityID` you expect before acting.

## 2. Re-read the entity

Events carry IDs only, never full state.

```graphql
mutation($clientId: String, $secretKey: String) {
  createSessionToken(clientId: $clientId, secretKey: $secretKey)
}
```

```graphql
query($id: String!) {
  transaction(id: $id) {
    transactionStatus
    declineReason
    merchantName
    amount
    items { id category { id name } }
  }
}
```

## 3. React

- **Enrich:** resolve a category with `transactionCategories`, then write it back per line item:

```graphql
mutation($transactionId: String!, $itemId: String!, $categoryId: String!) {
  transaction(id: $transactionId) { item(id: $itemId) { change(categoryId: $categoryId) { category { id } } } }
}
```

- **Respond to a decline:** if `transactionStatus == "DECLINED"` and `declineReason` names one of your rules, disable that rule for a short bounded window so the cardholder's retry succeeds — see `skills/klutch-spending-controls.md`.

## 4. Render back into the app

Mint a per-install token, then attach a panel:

```graphql
mutation($id: String) { recipeInstall(id: $id) { createToken } }
```

```graphql
mutation($id: String, $templateFilename: String, $data: JsonString, $entity: Entity, $size: RecipeSize) {
  recipeInstall(id: $id) { addPanel(templateFileName: $templateFilename, data: $data, entity: $entity, size: $size) { id } }
}
```

Use the install token (not the developer session token) as the bearer for panel mutations. Update an existing panel with `panel(id:){ changeData(data:) }` rather than adding a duplicate.

## Rules

- Webhook deliveries are not signed and delivery guarantees are undocumented — assume at-least-once and make your handler tolerant of repeats. Klutch has no idempotency contract, so deduplicate on the transaction entity ID yourself.
- Develop against `https://sandbox.klutchcard.com/graphql` and drive events with `sandbox { createTransaction / settleTransaction / reverseTransaction }` against a CLI-created test user.
- Panels are built from `@klutch-card/klutch-components`; scaffold with `klutch init` and publish with `klutch publish`.
