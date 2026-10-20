# Deployment (library-backend)

| Part | Where | Pipeline |
|---|---|---|
| Backend image | GHCR: `ghcr.io/<owner>/library-backend/library-api:{sha-xxxxxxx,latest}` | `.github/workflows/ci-cd.yml` → `docker` |
| Backend service | Render (free web service, runs the GHCR image) | `render.yaml` + `.github/workflows/ci-cd.yml` → `deploy-render` |

The frontend is deployed separately from [library-frontend](https://github.com/storyrail-squad/library-frontend).

Deploy jobs run only on `main` (push or manual "Run workflow"), after every quality job before them has passed.

The backend image follows one path: `docker` builds it into the runner's local Docker daemon (no push), Trivy scans
that local image and fails the job on any fixable CRITICAL/HIGH finding (`severity: CRITICAL,HIGH`,
`ignore-unfixed: true`, and `limit-severities-for-sarif: true` so the SARIF output keeps that filter; LOW/MEDIUM
findings never fail the gate), and only then are the same image's tags pushed to GHCR (no rebuild). `deploy-render` passes the pushed image's digest to Render, so Render runs exactly the
bytes Trivy scanned. Nothing is pushed, not even `latest`, when the scan fails.

## One-time setup

### 1. GHCR image visibility

The first `docker` run creates the package `library-backend/library-api` under your account (Profile → Packages). It is
private by default. Render pulls this image, so either open the package → **Package settings → Change visibility →
Public**, or keep it private and give Render a registry credential (step 2).

The job pushes with the built-in `GITHUB_TOKEN` (`packages: write`); no secret is needed.

### 2. Render

1. Run the **Backend CI/CD** workflow on `main` once so the image exists in GHCR (the `deploy-render` job skips itself
   until the hook secret below is set).
2. In `render.yaml`, check `image.url` points at your lower-case `<owner>/library-backend`. For a private package,
   create a registry credential in Render (**Settings → Registry credentials**, GHCR, a classic PAT with
   `read:packages`) named `ghcr` and uncomment the `creds` block.
3. In Render: **New → Blueprint**, connect this repository. Render reads `render.yaml` and creates the `library-api`
   web service (image runtime, free plan, health check `/actuator/health`, `PORT=8080`). Render never builds the
   Dockerfile itself.
4. When prompted, set `APP_CORS_ALLOWED_ORIGINS` (see section 4, CORS).
5. `autoDeploy` is off: deploys happen only when CI on `main` passes. In the service, open **Settings → Deploy Hook**
   and copy the URL.
6. In GitHub, **Settings → Secrets and variables → Actions**:
   - Secret `RENDER_DEPLOY_HOOK_URL` = the deploy hook URL.
   - Variable `BACKEND_URL` = the service URL, e.g. `https://library-api-xxxx.onrender.com` (no trailing slash).

`deploy-render` then calls the hook with `imgURL=ghcr.io/<owner>/library-backend/library-api@sha256:…` (the scanned image's
digest), polls `$BACKEND_URL/actuator/health` until `UP` (up to 15 min, free instances
pull and cold-start slowly), and smoke-tests `GET /api/books` → 200. Then `qa-acceptance` runs the `@qa` Cucumber scenarios from
[library-api-tests](https://github.com/storyrail-squad/library-api-tests) against `$BACKEND_URL`. Also set the variable
`QA_API_URL` (same value) in library-api-tests, so pushes to its `main` re-run the QA scenarios. Without the secret the job prints a notice and
does nothing. Without `BACKEND_URL` it triggers the deploy but skips verification.

Note: the previous instance keeps answering `UP` while Render pulls and starts the new image, so the health poll mainly proves the service is
reachable; check the Render dashboard for the new deploy's status. The free plan's H2 database is in-memory and is
reseeded with 8 books on every restart.

### 3. SonarCloud (optional)

1. Import the repository at sonarcloud.io. The organization key must equal the GitHub owner and the project key
   `<owner>_<repo>` (the defaults SonarCloud proposes); the workflow passes both.
2. Turn off **Automatic Analysis** for the project (CI analysis replaces it).
3. Add the secret `SONAR_TOKEN`. The `sonarcloud` job in `security.yml` then runs `mvn verify` for coverage and
   uploads using `sonar-project.properties`. Without the secret it is skipped.

### 4. CORS

The browser calls the API from the frontend's QA origin (library-frontend on GitHub Pages), which is the scheme + host only, no path:

```
APP_CORS_ALLOWED_ORIGINS=https://<owner>.github.io
```

Use a comma-separated list for several origins, e.g. `https://<owner>.github.io,http://localhost:5173`. Set it in the
Render service's **Environment** tab (it is declared with `sync: false` in `render.yaml`, so Render asks for it and
never overwrites it). The Dockerfile default is `http://localhost:5173`.

## Secrets and variables

| Name | Kind | Needed for | Required |
|---|---|---|---|
| `RENDER_DEPLOY_HOOK_URL` | secret | `deploy-render` | only to deploy to Render |
| `BACKEND_URL` | variable | `deploy-render` health poll + smoke test, and the QA acceptance run | with the hook |
| `SONAR_TOKEN` | secret | `sonarcloud` job | no |
| `GITLEAKS_LICENSE` | secret | `gitleaks` job | only if the repo is owned by an organization |

Also enable **Settings → Code security → Dependency graph** (on by default for public repos) for
`dependency-review`, and keep code scanning on so CodeQL and Trivy SARIF results appear under **Security**.
