---
name: Klutch spend analysis
description: Read a Klutch cardholder's transactions and answer spending questions — totals, category breakdowns, top merchants — using server-side filtering and aggregation instead of paging the whole history.
api: graphql/klutch-graphql-operations.yml
endpoint: https://graphql.klutchcard.com/graphql
operations:
  - createSessionToken
  - transactionsPaginated
  - sumTransactions
  - groupTransactions
  - transactionCategories
generated: '2026-07-19'
method: generated
source: postman/klutch-public-api-postman.json, https://github.com/KlutchCard/api-samples
---

# Klutch spend analysis

Read-only. Every operation here only reads the cardholder's own data.

## 1. Authenticate

POST all GraphQL to `https://graphql.klutchcard.com/graphql` (sandbox: `https://sandbox.klutchcard.com/graphql`).

```graphql
mutation($clientId: String, $secretKey: String) {
  createSessionToken(clientId: $clientId, secretKey: $secretKey)
}
```

Send the returned token on every later call as `Authorization: Bearer <token>`. Never echo the secret key or the token back to the user.

## 2. Answer aggregate questions server-side

Do **not** page the whole history to compute a total. Use `sumTransactions` for a single number:

```graphql
query($filter: TransactionFilter) { sumTransactions(filter: $filter) }
```

Use `groupTransactions` for a breakdown. `groupByProperty` is one of `CARD`, `MERCHANT_NAME`, `TRANSACTION_STATUS`, `TRANSACTION_TYPE`, `CATEGORY`; `operation` is one of `SUM`, `COUNT`, `AVERAGE`:

```graphql
query($filter: TransactionFilter, $groupByProperty: TransactionGroupByProperty, $operation: GroupByOperation) {
  groupTransactions(filter: $filter, groupByProperty: $groupByProperty, operation: $operation) { key value }
}
```

`TransactionFilter` accepts `startDate`, `endDate`, `cardIds`, `transactionStatus`, `transactionTypes`.

## 3. Only page when the user wants the actual list

```graphql
query($filter: TransactionFilter, $sortOrder: TransactionSortOrder, $limit: Int, $nextCursor: String) {
  transactionsPaginated(filter: $filter, sortOrder: $sortOrder, limit: $limit, nextCursor: $nextCursor) {
    nextCursor
    list { id transactionDate merchantName amount transactionStatus category { id name } mcc { code description } }
  }
}
```

Loop by passing the returned `nextCursor` back in; stop when it is null. Use `transactionCategories` to resolve category names.

Do not use `transactions(filter:)` — it is deprecated in favour of `transactionsPaginated`.

## Rules

- Introspection is disabled; do not attempt schema discovery, use the operations above.
- Errors come back as HTTP 200 with a top-level `errors[]` array — always check it before reading `data`; the machine-readable class is `errors[].extensions.classification`.
- A `transactionStatus` of `DECLINED` is a normal result, not an API failure. `declineReason` is free text.
- No rate limits are published — be conservative, prefer one aggregate call over many paged calls.
