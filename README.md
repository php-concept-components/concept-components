# Concept Components

Monorepo for first-party Concept feature scaffolds (`php-concept/component-*`).

| Role | Repo |
|------|------|
| **Monorepo (source of truth)** | [php-concept-components/concept-components](https://github.com/php-concept-components/concept-components) |
| **Split packages** | [php-concept-components/component-*](https://github.com/php-concept-components) |

Composer package names: `php-concept/component-auth-api-token`, …

Work only in this monorepo. Split repos are published automatically from `packages/*` (Symfony-style: **one release tag everywhere**).

Components are **editable scaffolds** (app product code), not sealed plugins — but they ship as versioned packages so apps can `require` them and release on a separate cadence from assemblies.

---

## Layout

```text
concept-components/
├── packages/
│   ├── auth-api-token/       → php-concept/component-auth-api-token
│   └── …
├── packages.json             # monorepo path → split repo name
├── scripts/
│   ├── create-split-repos.sh # once: create empty GitHub repos
│   └── split-packages.sh     # push subtree (+ optional tag) to splits
└── docker-compose.yml
```

Each package:

```text
composer.json
src/                          # PSR-4 Concept\Components\{Name}\
```

Namespace stays `Concept\Components\…` (same as when scaffolds lived under `concept-apps/components/`).

---

## Local development (Docker only)

Sibling repos must exist next to this one:

```text
../concept-core-2
../concept-extensions
```

```bash
cd /var/www/concept/concept-components

docker compose up -d --build
docker compose exec app bash

# inside container:
composer install
composer phpstan
```

Do not run Composer/PHPStan on the host.

---

## Packages

| Path | Composer |
|------|----------|
| `packages/auth-api-token` | `php-concept/component-auth-api-token` |

---

## Release (tag → push → split)

```bash
./scripts/align-package-deps.sh --components ^1.0 --extensions ^3.0   # first release / major
docker compose exec app ./scripts/update-package-locks.sh

git add -A && git commit -m "…"
git push origin main

./scripts/create-split-repos.sh   # once per new package
# set SPLIT_PUSH_TOKEN on the monorepo (PAT write to component-* repos)

git tag v1.0.0 && git push origin v1.0.0
# CI Split → then apps bump
```

See [scripts/README.md](scripts/README.md).
