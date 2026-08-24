# CI/CD Pipeline Demo

A production-style **CI/CD pipeline** built with **GitHub Actions** for a small Node.js/Express application. On every push or pull request, the pipeline automatically **lints**, **tests**, **builds**, and **deploys** the app to a staging environment (with a built-in simulation mode if no live cloud target is configured).

## Tech Stack

- **App:** Node.js + Express
- **Testing:** Jest + Supertest
- **Linting:** ESLint
- **Containerization:** Docker
- **CI/CD:** GitHub Actions
- **Deployment target:** Render (via deploy hook) — or simulated

## Pipeline Stages

The workflow lives at [`.github/workflows/ci-cd.yml`](.github/workflows/ci-cd.yml) and runs as **4 sequential jobs**, each depending on the previous one succeeding:

| Stage | Job name | What it does |
|---|---|---|
| 1. Lint | `lint` | Installs dependencies (`npm ci`) and runs ESLint (`npm run lint`) to catch code quality issues early. |
| 2. Test | `test` | Runs the Jest test suite with coverage (`npm test`) and uploads the coverage report as a build artifact. |
| 3. Build | `build` | Runs a build sanity check and builds a Docker image of the app, uploading it as an artifact for the deploy stage. |
| 4. Deploy | `deploy` | Only runs on a push to `main` (not on PRs). Deploys to staging — real deploy if a deploy-hook secret is present, otherwise a clearly logged simulated deployment so the pipeline always completes successfully. |

**Triggers:**
- `push` to `main` → full pipeline runs, including deploy
- `pull_request` targeting `main` → lint, test, and build run (deploy is skipped) so every PR is validated before merge

## Project Structure

```
cicd-pipeline-demo/
├── .github/
│   └── workflows/
│       └── ci-cd.yml        # GitHub Actions pipeline definition
├── src/
│   ├── index.js             # Express app
│   └── index.test.js        # Jest test suite
├── Dockerfile                # Container build definition
├── .eslintrc.json            # Lint rules
├── .env.example               # Documents expected env vars
├── .gitignore
├── package.json
└── README.md
```

## Environment Variables & Secrets

| Name | Where it's used | Required? |
|---|---|---|
| `PORT` | App runtime (`.env`, local only) | Optional (defaults to 3000) |
| `NODE_ENV` | App runtime | Optional |
| `RENDER_DEPLOY_HOOK_URL` | GitHub Actions secret, used in the `deploy` job | Optional — if unset, deployment is **simulated** instead of failing |

### How to set up the deploy secret (optional)

1. Create a free Web Service on [Render](https://render.com) (or any host that gives you a deploy webhook).
2. Copy its **Deploy Hook URL**.
3. In your GitHub repo: **Settings → Secrets and variables → Actions → New repository secret**.
4. Name it `RENDER_DEPLOY_HOOK_URL` and paste the URL as the value.

Without this secret, the pipeline still runs end-to-end in **simulation mode** — the deploy job logs each step it *would* take (load image → push to registry → roll out → smoke test) and finishes green, but it does **not** produce a live URL, since nothing is actually deployed. This keeps the pipeline fully demonstrable without requiring paid cloud infrastructure, while being explicit that no real environment exists yet.

## Running Locally

```bash
# install dependencies
npm install

# run the app
npm start
# → http://localhost:3000

# lint
npm run lint

# run tests with coverage
npm test

# build check + Docker image
npm run build
docker build -t cicd-pipeline-demo .
docker run -p 3000:3000 cicd-pipeline-demo
```

## Running the Pipeline

The pipeline runs automatically:
- Open a PR against `main` → lint, test, and build jobs execute and must pass before merge.
- Merge / push to `main` → all four jobs run, ending with deployment to staging.

You can watch it live under the **Actions** tab of the repository, and check the **deploy job's summary** for the staging URL and deployment details.

## Design Decisions & Problem-Solving Notes

- **Jobs are split and chained with `needs:`** rather than one monolithic job, so failures are isolated and easy to diagnose (e.g., a lint failure stops before wasting time on a Docker build).
- **Deploy is conditional on branch + event type** (`github.ref == 'refs/heads/main' && github.event_name == 'push'`) so pull requests are validated without accidentally deploying unmerged code.
- **Simulated deploy fallback** ensures the pipeline is fully runnable and demonstrable even without paid cloud infrastructure, while still showing exactly how a real deployment would be wired in.
- **Artifacts (coverage report, Docker image)** are passed between jobs via `actions/upload-artifact` / `download-artifact` instead of rebuilding, saving CI time and mirroring how real pipelines hand off build outputs to deploy stages.
- **`concurrency` group** cancels superseded runs on the same branch, avoiding wasted CI minutes on rapid pushes.

## License

MIT
