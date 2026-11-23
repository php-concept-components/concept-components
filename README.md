# Concept Components

Monorepo for first-party Concept feature scaffolds (`php-concept/component-*`).

| Role | Repo |
|------|------|
| **Monorepo (source of truth)** | [php-concept-components/concept-components](https://github.com/php-concept-components/concept-components) |
| **Split packages** | [php-concept-components/component-*](https://github.com/php-concept-components) |

Composer package names: `php-concept/component-auth-api-token`, …

Work only in this monorepo. Split repos are published automatically from `packages/*` (Symfony-style: **one release tag everywhere**).

Components are **editable scaffolds** (app product code), not sealed plugins — they ship as versioned packages so apps can `require` them and release on a separate cadence from assemblies.

---

## Layout

```text
concept-components/
├── packages/
│   ├── auth-api-token/       → php-concept/component-auth-api-token
│   └── …
├── packages.json             # monorepo path → split repo name
├── scripts/
│   ├── align-package-deps.sh
│   ├── create-split-repos.sh
│   ├── split-packages.sh
│   └── update-package-locks.sh
├── docker-compose.yml
└── Dockerfile
```

Each package:

```text
composer.json
src/                          # PSR-4 Concept\Components\{Name}\
```

Namespace stays `Concept\Components\…`.

Root `composer.json` name `php-concept/components` is **dev-only** (path-requires all packages + sibling extensions/core). It is not a Packagist product.

---

## Local development (Docker only)

Sibling repos must exist next to this one:

```text
/var/www/concept/
├── concept-components/     # this repo
├── concept-core-2/
├── concept-extensions/
└── concept-apps/           # optional mount — run assemblies against path packages
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

### Edit a component → see it in an app

1. Change code under `packages/<name>/src/`.
2. In **concept-apps** (dev compose + `composer-dev.json`), the app already path-requires  
   `/var/www/concept-components/packages/*` (symlink).
3. Restart / hit the app (`apps/api`, …) — no Packagist bump needed for DX.

Prod / CI readers use Packagist after tag → Split.

```bash
# from concept-apps (maintainer)
docker compose exec api composer phpstan
docker compose exec api composer console …   # if the assembly has console
```

---

## Packages

| Path | Composer |
|------|----------|
| `packages/auth-api-token` | `php-concept/component-auth-api-token` |

---

## Release (tag → push → split)

```bash
./scripts/align-package-deps.sh --components ^1.0 --extensions ^3.0   # major / first
docker compose exec app ./scripts/update-package-locks.sh

git add -A && git commit -m "…"
git push origin main

./scripts/create-split-repos.sh   # once per new package

git tag vX.Y.Z && git push origin vX.Y.Z
# CI Split → Packagist on split repo → apps align
```

See [scripts/README.md](scripts/README.md).

---

## CI split

Workflow: `.github/workflows/split.yml` on tags `v*` (+ `workflow_dispatch`).

Needs repo secret **`SPLIT_PUSH_TOKEN`** (PAT with **Contents: Read and write** on org `php-concept-components`):

```bash
gh secret set SPLIT_PUSH_TOKEN --repo php-concept-components/concept-components
```

---

## Checklist per release

- [ ] Changes committed on `main`
- [ ] `composer phpstan` green (in Docker)
- [ ] `git push origin main`
- [ ] `git tag vX.Y.Z` && `git push origin vX.Y.Z`
- [ ] CI Split green (or `./scripts/split-packages.sh --tag vX.Y.Z`)
- [ ] Packagist shows the new version on the split package
- [ ] Apps bump via `align-package-deps.sh --components ^…`
