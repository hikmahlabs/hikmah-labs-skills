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

## Cursor / other agents

These skills are markdown-first, so they work with any agent that supports custom rules. For Cursor, add the GitHub repo as a remote rule. For other agents, copy the relevant `SKILL.md` and `references/*.md` files into your agent's instruction directory.

## Contributing

Issues and PRs welcome. Each plugin lives under `plugins/<name>/` with a self-contained `skills/<name>/SKILL.md` plus `references/`.

## License

MIT
