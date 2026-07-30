---
name: Klutch virtual card issuing and control
description: Create a Klutch virtual card, activate it, rename it, and lock, unlock or cancel it — the card lifecycle an agent drives on a cardholder's behalf.
api: graphql/klutch-graphql-operations.yml
endpoint: https://graphql.klutchcard.com/graphql
operations:
  - createSessionToken
  - cards
  - createCard
  - card.activateCard
  - card.editCard
  - card.lock
  - card.unlock
  - card.cancelCard
generated: '2026-07-19'
method: generated
source: postman/klutch-public-api-postman.json
---

# Klutch virtual card issuing and control

**Consequential.** Creating and cancelling cards changes the cardholder's real financial instruments. Confirm with the user before `createCard` and always before `cancelCard` — cancellation is not reversible through this API.

## 1. Authenticate

```graphql
mutation($clientId: String, $secretKey: String) {
  createSessionToken(clientId: $clientId, secretKey: $secretKey)
}
```

Send `Authorization: Bearer <token>` on every subsequent call.

## 2. List the existing cards first

Never issue a new card before checking whether a suitable one exists.

```graphql
query { cards { id name status lastFour expirationDate media lockState } }
```

## 3. Create a card

```graphql
mutation($name: String, $media: CardMedia) {
  createCard(name: $name, media: $media) { id name status lastFour expirationDate media }
}
```

`media` is the `CardMedia` enum (virtual vs physical form factor). Give the card a descriptive `name` — it is how the user will recognize it, and rule `displayName`s surface in decline reasons.

## 4. Activate a physical card

Requires the last four digits printed on the card, supplied by the user:

```graphql
mutation($id: String, $lastFour: String) {
  card(id: $id) { activateCard(lastFour: $lastFour) { id status } }
}
```

## 5. Lock, unlock, rename

```graphql
mutation($id: String) { card(id: $id) { lock { id status } } }
mutation($id: String) { card(id: $id) { unlock { id status } } }
mutation($id: String, $name: String) { card(id: $id) { editCard(name: $name) { id status } } }
```

Prefer `lock` over `cancelCard` when the user wants to stop spend temporarily — locking is reversible.

## 6. Cancel (irreversible)

```graphql
mutation($id: String, $terminateReason: TerminateReason) {
  card(id: $id) { cancelCard(terminateReason: $terminateReason) { id status } }
}
```

Ask for explicit confirmation naming the card, then supply a `TerminateReason`.

## Rules

- **No idempotency contract exists.** If a `createCard` call times out, re-run `cards` and check whether the card was created before retrying — a blind retry can issue a duplicate card.
- Never print a full card number, and treat `lastFour` as user-supplied input, not something to look up.
- Test everything against `https://sandbox.klutchcard.com/graphql` with a CLI-created test user first (see sandbox/klutch-sandbox.yml).
- Check the GraphQL `errors[]` array on every response before reading `data`.
