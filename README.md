# Hikmah Labs Skills

Claude Code plugins and agent skills built and maintained by [Hikmah Labs](https://hikmahlabs.com).

## Claude Code

Add the marketplace:

```
/plugin marketplace add hikmahlabs/hikmah-labs-skills
```

Install a plugin:

```
/plugin install general-translation
```

Once installed, the skill is auto-discoverable — Claude will invoke it when your prompts match its description (e.g. "translate this app", "fix script tag while rendering", "add gt-next").

## Plugins

### `general-translation`

Set up or extend [General Translation](https://generaltranslation.com) (`gt-next`) i18n in a Next.js App Router app. Covers:

- `<GTProvider>` placement (the script-tag-crash bug)
- `<T>` / `<Var>` / `t()` translation patterns
- The `useOptionalGT()` wrapper for components rendered outside the provider tree
- ESLint `no-restricted-imports` + smoke-test enforcement
- Locale switching with cookie-based routing and full-page reloads
- RTL support with Tailwind logical properties
- Backend error codes + DeepL split for user-generated content
- A 5-phase migration workflow for translating existing apps

Built from a fully shipped integration (Next.js 16, 5 locales, ~960 `<T>` blocks, 17 outside-provider files, ESLint + smoke-test enforcement).

[See the plugin README →](./plugins/general-translation/README.md)

### `stripe-convex-webhook`

Set up (or migrate to) a Stripe webhook hosted on a [Convex](https://convex.dev) HTTP endpoint instead of a Next.js API route. Stripe delivers events directly to the dev deployment over the public internet — no `stripe listen --forward-to` tunnel needed. Covers:

- The `httpAction` pattern with `stripe.webhooks.constructEventAsync()` (Web Crypto, not Node `crypto`)
- Mechanical migration from a Next.js API route (`convex.mutation` → `ctx.runMutation`, response shape, env-var move)
- Converting webhook-only `mutation` / `query` to `internalMutation` / `internalQuery` after the public route is gone
- Updating internal Convex callers (reconciliation crons, admin helpers) from `api.foo.bar` → `internal.foo.bar`
- The `*.convex.site` URL (HTTP routes) vs `*.convex.cloud` (WebSocket queries) distinction
- Convex env-var audit (`STRIPE_WEBHOOK_SECRET`, `LOOPS_*_TEMPLATE_ID`) and Stripe Dashboard endpoint setup, dev + prod
- Common pitfalls (sync vs async signature verify, 200-on-error masking failures, leaving mutations public, `"use node"` on `httpAction`)

Distilled from the Sakeenaty migration (Next.js → Convex): 7 event types, 8 webhook-only mutations/queries tightened to internal, no `stripe listen` running in dev.

[See the plugin README →](./plugins/stripe-convex-webhook/README.md)

## Cursor / other agents

These skills are markdown-first, so they work with any agent that supports custom rules. For Cursor, add the GitHub repo as a remote rule. For other agents, copy the relevant `SKILL.md` and `references/*.md` files into your agent's instruction directory.

## Contributing

Issues and PRs welcome. Each plugin lives under `plugins/<name>/` with a self-contained `skills/<name>/SKILL.md` plus `references/`.

## License

MIT
