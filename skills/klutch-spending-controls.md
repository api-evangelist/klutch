---
name: Klutch spending controls
description: Author, list, temporarily disable and delete Klutch transaction rules — the server-side spend controls that approve or decline card authorizations in real time.
api: graphql/klutch-graphql-operations.yml
endpoint: https://graphql.klutchcard.com/graphql
operations:
  - createSessionToken
  - cards
  - transactionRules
  - createTransactionRule
  - transactionRule.disableFor
  - transactionRule.delete
generated: '2026-07-19'
method: generated
source: postman/klutch-public-api-postman.json, https://github.com/KlutchCard/api-samples
---

# Klutch spending controls

**Consequential.** A transaction rule can decline the cardholder's real purchases at the point of sale. Confirm scope (which cards, which conditions) with the user before creating one, and warn that an over-broad rule will cause declines in a store.

## 1. Authenticate and scope the rule to cards

```graphql
mutation($clientId: String, $secretKey: String) {
  createSessionToken(clientId: $clientId, secretKey: $secretKey)
}
query { cards { id name status } }
```

Collect the `cardIds` the rule should bind to. A rule with no explicit card scope is riskier than one bound to a single virtual card.

## 2. Read what already exists

```graphql
query {
  transactionRules {
    id name cards { id } createdAt createdBy { entityID type }
    spec {
      specType
      ... on AccumulatingOverPeriodRuleSpec { period amount filters { field value operator invert } }
      ... on DeclineBasedOnFilterRuleSpec { filter { field value operator invert } }
    }
  }
}
```

Do not create a rule that duplicates or contradicts an existing one.

## 3. Create the rule

```graphql
mutation($name: String, $displayName: String, $cardIds: [String], $spec: TransactionRuleSpecInput) {
  createTransactionRule(name: $name, displayName: $displayName, cardIds: $cardIds, spec: $spec) {
    id name cards { id } createdAt spec { specType }
  }
}
```

Spec variants Klutch publishes: `AccumulatingOverPeriodRuleSpec` (a cap per period), an accumulate-between-dates spec, `DeclineBasedOnFilterRuleSpec`, a day-of-week spec, and a time-of-day spec.

Filter predicates take a `field`, `value`, `operator` and `invert`.

- `field`: `ADDRESS`, `AMOUNT`, `CARDHOLDER_PRESENT`, `CARD_PRESENT`, `CITY`, `ENTRY_MODE`, `ID`, `LEGAL_NAME`, `MCC`, `MERCHANT_ID`, `MERCHANT_NAME`, `STATE`, `TERMINAL_ID`, `ZIP_CODE`
- `operator`: `CONTAINS`, `EQUALS`, `GREATER`, `GREATER_EQUALS`, `LESS`, `LESS_EQUALS`, `REGEX_MATCHES`, `STARTS_WITH`

Set a `displayName` the cardholder will understand — it is what surfaces in the decline reason on a blocked transaction.

## 4. Let a legitimate transaction through

To allow a retry that a rule just blocked, disable the rule for a bounded window rather than deleting it:

```graphql
mutation($name: String, $duration: Int) {
  transactionRule(name: $name) { disableFor(durationInSeconds: $duration) { id } }
}
```

Use the shortest workable duration. This is the mechanism Klutch's own "swipe twice" sample uses after detecting a rule-caused decline.

## 5. Delete

```graphql
mutation($id: String) { transactionRule(id: $id) { delete } }
```

## Rules

- Always dry-run new rules in the sandbox (`https://sandbox.klutchcard.com/graphql`) by simulating an authorization with `sandbox { createTransaction(...) }` before applying them to a live card.
- No idempotency contract — re-read `transactionRules` after a timeout instead of blindly retrying `createTransactionRule`.
- Declines surface as a transaction with `transactionStatus: DECLINED` and a free-text `declineReason` containing the rule's display name, not as a GraphQL error.
