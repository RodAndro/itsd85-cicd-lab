# ITSD 85 Lab: Automated CI/CD Pipeline

**Student(s):** Rodan
**Section:** BSIT
**Instructor:** *(to be filled in)*

## What this project does

A small Node.js (Express) web API that is linted and tested on every push and pull request by GitHub Actions, then automatically built into a Docker image and redeployed on a self-hosted runner when the tests pass on `main`.

The `/health` endpoint reports the commit SHA that is running, which proves that each push actually reached the server.

## Pipeline diagram

![Pipeline](docs/pipeline.png)

```
Local dev -> GitHub (CI: lint + test on ubuntu-latest) -> quality gate
          -> self-hosted runner (CD: docker compose up -d --build)
          -> health check that /health reports the new commit SHA
```

## How to run locally

Requires Node.js 24 and npm.

```bash
npm ci
npm run lint
npm test
npm start
```

```bash
curl http://localhost:3000/health
curl 'http://localhost:3000/api/add?a=2&b=3'
```

## How the deployment works (test job -> deploy job on self-hosted runner)

1. `test` runs on GitHub's `ubuntu-latest` runner: `npm ci`, `npm run lint`, `npm test`.
2. `deploy` declares `needs: test`, so it only runs when the tests passed, and only for pushes to `main`.
3. `deploy` targets `runs-on: [self-hosted, linux, deploy]` â€” the student's own Ubuntu server, whose runner polls GitHub outbound, so no public IP is required.
4. The image is tagged with `${APP_VERSION}`, set to `github.sha`, and `docker compose up -d --build --remove-orphans` replaces the running container.
5. The health check retries `http://localhost:8080/health` until the payload contains the new commit SHA.

## Results (push-to-live times from Part 7.5)

| Test | Expected result | Actual result | Push-to-live time | Screenshot |
|---|---|---|---|---|
| A - good change | test pass, deploy pass, new name and commit ID live | test success, deploy success, `ITSD 85 CI/CD Lab - Rodan` live at SHA `6f63295` | 0 min 16 s | docs/screenshots/cp6-deploy-green.png |
| B - broken change | test fail, merge blocked, server unchanged | test failed at `Run tests` (3 of 7 assertions), deploy skipped, merge rejected by ruleset: `Required status check "test" is failing` | n/a | docs/screenshots/cp7-test-red.png |
| C - rollback (Method 1) | old content and a new commit ID live | reverted through the pipeline, good content live at SHA `92ea9be` | 0 min 22 s | docs/screenshots/cp8-rollback.png |
| C - rollback (Method 2) | previous commit ID live within seconds | `APP_VERSION=<old-sha> docker compose up -d --no-build` restored SHA `6f63295` immediately | under 5 s | docs/screenshots/cp9-emergency.png |

Mean push-to-live (n = 2 automated deploys): 19 s. Range: 16-22 s.

## Screenshots (links to docs/screenshots/)

Checkpoint images are stored in `docs/screenshots/`:

- `cp1-versions.png` - git/node/npm versions and `ssh -T git@github.com`
- `cp2-tests-green.png` - `npm test` showing 7 passing tests
- `cp3-repository.png` - repository file list
- `cp4-ci-green.png` - green `test` job in Actions
- `cp5-docker-manual.png` - hello-world and the browser showing the deployed version
- `cp6-deploy-green.png` - green `test -> deploy` run and the browser showing the commit ID

## Paper (link to docs/paper.pdf)

See `docs/paper.pdf`.
