# C2C Merchant API Docs

**Live site:** https://c2-c-92f40af6.mintlify.site/api-reference/introduction

Merchant-facing API documentation built with [Mintlify](https://mintlify.com). The structure mirrors the
WayPay docs (Guides + API reference tabs, hand-written MDX endpoint pages) but documents only what the
C2C merchant API actually supports.

## Local development

**Prerequisites**: Node.js 20+

```bash
npm install -g mint   # or: npx mint dev
mint dev              # from the repository root → http://localhost:3000
```

Check links before publishing:

```bash
mint broken-links
```

## Keeping docs and code aligned

- These pages document the C2C API (`harisnaseer191/C2C`); the same content lives in that repo's `docs/`
  folder. Keep both in step.
- Endpoint pages under `api-reference/endpoints/` describe `MerchantOrdersController`
  (`src/C2C.API/Controllers/MerchantOrdersController.cs`). Update them with any change to its request/response
  contracts, validators (`CreateMerchantOrderCommandValidator`) or error messages.
- Request signing follows WayPay's scheme (MD5 `signature` field in the body). The worked examples in
  `api-reference/signature-guide.mdx` are pinned by
  `MerchantRequestSignatureTests.WhenUsingDocumentedWorkedExample_Compute_MatchesPublishedSignature`. If you change
  them, update the test (and vice versa).
- `openapi.json` is the published, hand-maintained merchant spec. The API also generates its own spec at
  `/openapi/v1.json` in Development (the `signature` body field and the 401/413 signature responses are
  documented automatically for `[RequireSignature]` endpoints) — use it to cross-check.
- `mobile-api.md` documents the member mobile API and is excluded from the site via `.mintignore`.

## Project structure

```text
.
├── docs.json                          # Mintlify config (navigation, theme, redirects)
├── openapi.json                       # Merchant API OpenAPI 3.1 spec
├── index.mdx                          # Overview
├── quickstart.mdx                     # First signed deposit, end to end
├── guides/
│   ├── deposit-flow.mdx
│   ├── withdrawal-flow.mdx
│   ├── hosted-payment-page.mdx
│   ├── order-status.mdx
│   └── webhooks.mdx
└── api-reference/
    ├── introduction.mdx
    ├── signature-guide.mdx
    ├── authentication.mdx
    ├── idempotency.mdx
    ├── errors.mdx
    └── endpoints/
        ├── query-balance.mdx
        ├── create-deposit.mdx
        ├── create-withdrawal.mdx
        └── query-order-status.mdx
```

## Deployment

This repository is connected to Mintlify: pushing to `main` deploys the public docs. Only merge changes once
the API behaviour they describe is deployed.
