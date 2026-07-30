# Klutch

Klutch (Klutch Card) issues an "app-powered" programmable credit card — Klutch Credit (unsecured) and Klutch Spend (collateralized prepaid) — whose behavior is extended by user-installable Mini Apps. Its public GraphQL API exposes enriched card transactions with acquirer metadata, virtual card lifecycle, server-side transaction rules that approve or decline spend in real time, account balances, ACH card payments, and Mini App panel rendering.

- Website: https://klutchcard.com
- Developers: https://www.klutchcard.com/developer
- API reference (Postman): https://api-docs.klutchcard.com/
- GitHub: https://github.com/KlutchCard

## API surface

| | |
|---|---|
| Style | GraphQL (single endpoint, introspection disabled) |
| Production | `https://graphql.klutchcard.com/graphql` |
| Sandbox | `https://sandbox.klutchcard.com/graphql` |
| Auth | Bearer session token (`createSessionToken`) or OAuth 2.1 authorization code + PKCE |
| MCP | `https://mcp.klutchcard.com` (remote, OAuth-protected, Claude + ChatGPT) |
| Events | Transaction webhooks (registered in the account UI) |

## Artifacts in this repo

| Artifact | File |
|---|---|
| Operation catalog | `graphql/klutch-graphql-operations.yml` |
| Postman collection (verbatim) | `postman/klutch-public-api-postman.json` |
| Packages / SDKs | `packages/klutch-packages.yml` |
| CLI | `cli/klutch-cli.yml` |
| Mini App components | `components/klutch-components.yml` |
| MCP server | `mcp/klutch-mcp.yml` |
| Agent Skills | `skills/_index.yml` |
| Authentication | `authentication/klutch-authentication.yml` |
| OAuth scopes | `scopes/klutch-scopes.yml` |
| Well-known | `well-known/klutch-well-known.yml` |
| Conventions | `conventions/klutch-conventions.yml` |
| Errors | `errors/klutch-error-codes.yml` |
| Webhook catalog | `asyncapi/klutch-webhooks.yml` |
| Sandbox | `sandbox/klutch-sandbox.yml` |
| Data model | `data-model/klutch-data-model.yml` |
| Lifecycle | `lifecycle/klutch-lifecycle.yml` |
| Conformance | `conformance/klutch-conformance.yml` |
| Domain security | `security/klutch-domain-security.yml` |
| llms.txt | `llms/klutch-llms.txt` |

## Notable gaps (as of 2026-07-19)

No OpenAPI or AsyncAPI document, no `security.txt`, no status page, no published changelog, no deprecation policy, no documented rate limits, and **no idempotency contract** — which matters, because `createPayment` moves money.

Backed by: 500 Global, Bain Capital Ventures.
