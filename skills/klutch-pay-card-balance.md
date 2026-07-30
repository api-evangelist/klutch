---
name: Klutch pay card balance
description: Read the Klutch revolving balance and initiate an ACH payment toward the card from a linked transfer source.
api: graphql/klutch-graphql-operations.yml
endpoint: https://graphql.klutchcard.com/graphql
operations:
  - createSessionToken
  - account.revolvingLoan
  - transferSources
  - createPayment
generated: '2026-07-19'
method: generated
source: postman/klutch-public-api-postman.json
---

# Klutch pay card balance

**Moves money. Human confirmation is required before every `createPayment` call.** State the amount, the payment type and the funding source back to the user and get an explicit yes.

## 1. Authenticate

```graphql
mutation($clientId: String, $secretKey: String) {
  createSessionToken(clientId: $clientId, secretKey: $secretKey)
}
```

## 2. Read the balance

```graphql
query { account { revolvingLoan { balance limit } } }
```

## 3. Pick a funding source

```graphql
query { transferSources { id name status } }
```

Only a source with `status: ACTIVE` can be used. `PENDING` means the source is not yet valid; `DELETED` sources must be ignored. If no source is `ACTIVE`, stop and tell the user to link one — do not attempt a payment.

## 4. Create the payment

```graphql
mutation($transferSourceId: String, $type: PaymentType, $amount: Float) {
  createPayment(transferSourceId: $transferSourceId, type: $type, amount: $amount) {
    id paymentStatus paymentType amount
  }
}
```

Klutch documents three payment types: the full outstanding balance, the minimum due, and a fixed amount. A fixed amount must be greater than $15.00 or the minimum due. The call initiates an ACH transfer from the source to the card.

## 5. Confirm the outcome

Read `paymentStatus` on the response. Report the returned `id` and `amount` to the user.

## Rules

- **No idempotency contract exists — this is the highest-risk operation in the API.** If `createPayment` times out or errors ambiguously, do NOT retry. Re-read the balance and surface the uncertainty to the user; a blind retry can double-pay.
- Never infer an amount. If the user did not name one, use the outstanding-balance or minimum-due payment type explicitly.
- Check the GraphQL `errors[]` array before treating a payment as successful — errors arrive with HTTP 200.
- Exercise the flow in the sandbox first (`https://sandbox.klutchcard.com/graphql`).
