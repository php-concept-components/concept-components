# Scripts — concept-components

Утиліти монорепо. Composer-команди краще ганяти в Docker (`docker compose exec app …`).

Карта пакетів → split-репо: [`../packages.json`](../packages.json).  
Extensions sibling: `../concept-extensions` (теги звідти — дефолт для `extension-*`).

| Скрипт | Роль |
|--------|------|
| `align-package-deps.sh` | Вирівняти `require` у `composer.json` (`component-*` + `extension-*`) |
| `update-package-locks.sh` | Оновити **root** `composer.lock` |
| `create-split-repos.sh` | Один раз: створити порожні GitHub-репо в org |
| `split-packages.sh` | Локальний fallback split (звичайний реліз — через CI) |

### Locks

| Файл | У git? |
|------|--------|
| Root `composer.lock` | Так — один граф для Docker/CI монорепо |
| `packages/*/composer.lock` | Ні — libraries; споживачам не потрібні |

`php-concept/core-2` скрипт align **не** чіпає.

---

## Порядок відносно extensions

1. Спочатку extensions: align cross-deps → tag → CI Split.
2. Потім components: align `component-*` + `extension-*` → tag → CI Split.
3. Потім apps bump на `php-concept/component-*`.

---

## Реліз (один прохід)

Packagist/apps читають **`composer.json` з тегу**, не lock. Версія пакета = git-тег (поля `"version"` у json немає).

```
1. align-package-deps.sh   # за потреби з --components / --extensions
2. update-package-locks.sh # root (path-репо — можна до тега)
3. commit + git push origin main
4. git tag vX.Y.Z && git push origin vX.Y.Z
   → CI (.github/workflows/split.yml) сплітить усі component-* з тим самим тегом
```

`split-packages.sh` **не обовʼязковий** (fallback).

---

## `align-package-deps.sh`

**Що робить**

- У `packages/*/composer.json`: `php-concept/component-*` і `php-concept/extension-*` → один constraint на префікс.
- У кореневому `composer.json`: ті самі `require` + pin у path `repositories[].options.versions` (`^X.Y` → `X.Y.0`).
- Composer / locks не чіпає.

**Звідки constraint**

| Префікс | Дефолт |
|---------|--------|
| `component-*` | останній git-тег **цього** репо (`v1.0.0` → `^1.0`) |
| `extension-*` | останній тег `../concept-extensions` |

Стрибок major (останній тег ще старий) — передати явно:

```bash
./scripts/align-package-deps.sh --components ^2.0 --extensions ^3.0
```

**Запуск**

```bash
./scripts/align-package-deps.sh
./scripts/align-package-deps.sh --dry-run
./scripts/align-package-deps.sh --components ^1.0 --extensions ^3.0
./scripts/align-package-deps.sh --set php-concept/component-=^1.0
./scripts/align-package-deps.sh auth-api-token    # один пакет (dirname у packages/)
```

---

## `update-package-locks.sh`

Лише **root** `composer.lock` (`composer update --no-install`).

```bash
docker compose exec app ./scripts/update-package-locks.sh
docker compose exec app ./scripts/update-package-locks.sh --dry-run
```

---

## `create-split-repos.sh`

Одноразово: репо в org з `packages.json`, якщо ще немає.

```bash
./scripts/create-split-repos.sh
```

Потрібні: org `php-concept-components` (створити в GitHub UI) + `gh` з доступом.

---

## `split-packages.sh` (fallback)

Звичайний шлях — **CI** на `push` тега `v*`.

```bash
./scripts/split-packages.sh --tag vX.Y.Z
./scripts/split-packages.sh --tag vX.Y.Z --dry-run
./scripts/split-packages.sh --tag vX.Y.Z component-auth-api-token
```

---

## Новий пакет

1. `packages/<name>/` + запис у `packages.json`
2. `./scripts/create-split-repos.sh`
3. Commit → tag/push → CI Split

---

## Чого не робити

| ❌ | ✅ |
|----|----|
| Комітити `packages/*/composer.lock` | Лише root lock |
| Тегати components, поки extensions split ще зі старими cross-deps | Спочатку extensions release |
| Align major без `--components` / `--extensions`, коли останній тег ще старий major | Явні `--components ^N.0 --extensions ^N.0` |
| Завжди ганяти `split-packages.sh` руками | `git push origin vX.Y.Z` → CI |
