# C2C Merchant API Docs

Merchant-facing API documentation built with [Mintlify](https://mintlify.com).

## Local development

**Prerequisites**: Node.js 20+

```bash
npm install -g mint
```

From the `docs/` directory:

```bash
mint dev
```

Preview at <http://localhost:3000>.

## Validate OpenAPI spec

```bash
mint validate openapi.json
```

## Deployment

1. Go to [mintlify.com/start](https://mintlify.com/start)
2. Connect your GitHub repository
3. Set the docs directory to `docs/`
4. Push to main — Mintlify deploys automatically

## Project structure

```
docs/
├── docs.json                  # Mintlify configuration
├── openapi.json               # Merchant API OpenAPI 3.1 spec
├── index.mdx                  # Introduction
├── quickstart.mdx             # Getting started guide
├── authentication.mdx         # API key authentication
├── errors.mdx                 # Error codes reference
├── rate-limits.mdx            # Rate limiting
├── idempotency.mdx            # Idempotency key usage
├── webhooks.mdx               # Webhooks (coming soon)
└── guides/
    └── server-integration.mdx # Full server-side integration guide
```
