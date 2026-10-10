# CLAUDE.md

Guidance for Claude Code when working in this repository.

This is the documentation site for pyhive-integration, the Python library
(`apyhiveapi` / `pyhiveapi`) behind the Home Assistant Hive integration. It is
a Jekyll site published with GitHub Pages at
<https://pyhive.github.io/pyhiveapi.docs/>. The library itself lives in a
separate repository (`Pyhass/Pyhive`); this repository holds no library code.

## Branches

- `V2-pyhive-docs` — working branch for the v2.0.0 documentation.
- `main` — default branch.

## Commands

```bash
# Install dependencies
bundle install

# Build / preview the site locally
bundle exec jekyll serve

# Run linters (markdownlint and secretlint, via pre-commit)
pre-commit run --all-files
```

markdownlint is configured in `.markdownlint.json` (line length and inline
HTML checks are off). secretlint is configured in `.secretlintrc.json` and
runs through Docker.

## Layout

```text
index.md                 ← site home page
docs/
├── getting-started.md
├── authentication.md
├── session.md
├── exceptions.md
├── migration.md         ← upgrade guide for pre-2.0.0 users
├── home-assistant.md    ← guide: Home Assistant integration
├── standalone-async.md  ← guide: apyhiveapi
├── standalone-sync.md   ← guide: pyhiveapi
├── api/                 ← per-device API references (heating, hotwater,
│                          lights, plugs, sensors, hub, actions)
└── examples/            ← worked examples (api.md, session.md)
_data/sidebar.yml        ← navigation menus
_includes/, _layouts/    ← site templates
_config.yml              ← Jekyll configuration
```

## Writing pages

- Every page starts with front matter: `title`, `description` and `sidebar`
  (the menu from `_data/sidebar.yml` to show, e.g. `main` or `api`).
- A new page is not linked anywhere until it is added to `_data/sidebar.yml`.
  URLs there include the `/pyhive.docs` base path and omit the `.md` extension.
- Per-device API references go in `docs/api/`, one file per `hive.*` module.

## Keeping docs accurate

- Document what the library actually does. Check method names, arguments,
  return values and `device.status` keys against the library source before
  writing them down; do not document from memory.
- Use the snake_case names (`start_session`, `update_data`, `device_list`) as
  the primary API. The camelCase names (`startSession`, `updateData`,
  `deviceList`) are backwards-compatible aliases kept for Home Assistant;
  mention them only as aliases.
- Async examples use `apyhiveapi` and `await`; sync examples use `pyhiveapi`.
- Breaking changes from pre-2.0.0 belong in `docs/migration.md`.
- Never put real credentials, tokens, device keys, email addresses or phone
  numbers in examples. Use obvious placeholders such as `user@example.com`.
