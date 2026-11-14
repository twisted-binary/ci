# twisted-binary/ci

Shared GitHub Actions workflows and Renovate presets for the twisted-binary and tmbs88 repos.
Nothing here is secret: it's generic CI templates.

Callers pin the moving major tag `@v1`. Breaking changes get a new major tag (`v2`).

## Laravel apps — `.github/workflows/laravel.yml`

```yaml
# .github/workflows/ci.yml
name: ci
on:
  push:
    branches: [main]
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
concurrency:
  group: ci-${{ github.head_ref || github.ref }}
  cancel-in-progress: true
permissions:
  contents: read
jobs:
  laravel:
    uses: twisted-binary/ci/.github/workflows/laravel.yml@v1
    with:
      php-versions: '["8.4", "8.5"]' # include the lowest PHP production runs
```

Jobs:

| Job | Runs on drafts | What it does |
|---|---|---|
| `quality` | yes | `composer test:lint`, `composer test:types`, `test:type-coverage` (if defined), `npm run build` + `build:ssr` (if defined) |
| `tests (PHP x)` | no | one job per PHP version: client build, then `composer test:unit` |
| `browser` | no | only when `tests/Browser` exists: Playwright chromium, then `composer test:browser` (or `pest tests/Browser`) |

The app provides these composer scripts. All of them check; none may rewrite files.

| Script | Required | Typical content |
|---|---|---|
| `test:lint` | yes | `pint --parallel --test`, `rector --dry-run`, `npm run format:check`, `npm run lint` |
| `test:types` | yes | `phpstan` |
| `test:unit` | yes | `pest --parallel --coverage --compact` |
| `test:type-coverage` | no | `pest --type-coverage --min=100` |
| `test:browser` | no | `pest --compact tests/Browser` |
| `ci:setup` | no | app-specific prep, run after `.env` + `APP_KEY` + `database/database.sqlite`: passport keys, wayfinder/ziggy generation, data dirs, migrations |

Inputs: `php-versions`, `quality-php` (default `8.5`), `php-extensions`, `node-version` (default `24`), `coverage` (default `pcov`), `runs-on` (default `["self-hosted", "mini23"]`).

Repo-specific extra jobs (for example a MySQL dialect run) stay in the repo's own `ci.yml` next to the `laravel` job.

## Static sites — `.github/workflows/static-site.yml`

```yaml
jobs:
  site:
    uses: twisted-binary/ci/.github/workflows/static-site.yml@v1
    with:
      output-dir: dist          # .output/public for nuxt generate; empty skips the smoke check
      expect-sitemap: true
```

Runs `npm ci`, then each of `format:check`, `lint:check`, `check`, `typecheck`, `test` that the site defines, then `npm run build`. With `output-dir`, it checks the output has an `index.html` with a `<title>` and a `sitemap*.xml`.

Server-rendered sites (Nuxt `nuxt build`, Next) pass `serve-command` instead, e.g. `node .output/server/index.mjs` or `npm start`. The workflow starts it on port 4381 (`PORT`/`HOST` set) and requests each of `smoke-paths` (default `/ /sitemap.xml /robots.txt`): every path must return 200 after redirects, and HTML pages must have a `<title>`.

## Security — `.github/workflows/security.yml`

```yaml
# .github/workflows/security.yml
name: security
on:
  push:
    branches: [main]
    paths: [composer.lock, package-lock.json]
  pull_request:
    paths: [composer.lock, package-lock.json]
  schedule:
    - cron: '0 5 * * 1'
  workflow_dispatch:
permissions:
  contents: read
jobs:
  audit:
    uses: twisted-binary/ci/.github/workflows/security.yml@v1
```

Reads the lockfiles only, with no install: `composer audit --locked --no-dev` and `npm audit --omit=dev --audit-level=high`.

## Runner notes

- Private twisted-binary repos run on the org's self-hosted `mini23` runner (Linux ARM64). Public repos and other owners must pass `runs-on: '["ubuntu-latest"]'`; the runner group refuses public repos.
- `mini23` cannot run Docker *container actions* (the host's Docker refuses the `_temp` bind mount). Use JavaScript/composite actions and plain commands only.
- Third-party actions are pinned by commit SHA; Renovate keeps the pins current.

## Renovate presets

| Preset | Use in | Behavior |
|---|---|---|
| `github>twisted-binary/ci` | Laravel apps, packages | weekly (Monday before 7:00 Zurich), 7-day minimum release age, one grouped PR for minor + patch, separate PRs for majors and 0.x minors, lockfile maintenance on the 1st of each month, `@types/node` held below 25, no automerge |
| `github>twisted-binary/ci:static` | static sites | the above, plus patch updates in their own group, merged by Renovate after all checks pass |

Laravel apps also set the lowest production PHP so Renovate resolves composer updates against it:

```json
{
  "extends": ["github>twisted-binary/ci"],
  "constraints": { "php": "8.4.*" }
}
```

Security updates (GitHub vulnerability alerts) skip the schedule and the minimum release age.

The free plan has no branch protection for private repos, so nothing enforces green CI before a human merge. Renovate's automerge checks the status itself (`platformAutomerge: false`), and only merges when every check passed.
