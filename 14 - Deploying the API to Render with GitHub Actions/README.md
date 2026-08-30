# PulseVote – Deploying the API to Render with GitHub Actions

In the previous activities, you already:

* Built and secured the PulseVote API
* Added RBAC and automated the existing Postman collection with Newman
* Added ESLint and Jest unit testing
* Dockerised the API
* Created a GitHub Actions CI pipeline
* Added SonarQube Cloud analysis and a Quality Gate

At this point, your pipeline can tell you whether the API is ready to deploy. In this activity, we will add the Continuous Deployment (CD) part of the pipeline and deploy the API to Render only after the complete CI job succeeds.

## Research

Before you code, spend 10–15 minutes researching Render and Continuous Deployment.

Answer briefly:

* What is Continuous Deployment (CD)?
* How is CI different from CD?
* What is a Render Web Service?
* What is a deploy hook?
* Why should a deploy hook URL be treated as a secret?
* What does `needs:` do between GitHub Actions jobs?
* Why should pull requests run CI but not deploy the production application?
* What is the risk of clearing a database before automated tests run?

Write a 4–6 sentence summary in your own words and commit it to your repo.

## Requirements

After this activity, your PulseVote API pipeline will:

* Continue running ESLint, Jest, SonarQube, Docker and Newman on pushes and pull requests to `main`
* Continue using the same MongoDB Atlas database configured already
* Deploy only after the complete `api-ci` job succeeds
* Deploy only when code is pushed to `main`
* Not deploy from a pull request
* Trigger a Render deployment through a secret deploy hook
* Run the production API as a Docker-based Render Web Service
* Use the same MongoDB Atlas database from the existing `MONGO_URI` configuration
* Expose the `/health` endpoint to Render as the service health check

## The same Atlas database is used throughout this learning project

For this learning activity, PulseVote uses the same MongoDB Atlas database for the GitHub Actions pipeline and the deployed Render API. Therefore, Render should use that **same Atlas database**:

```text
Render
MONGO_URI → pulsevote
```

This keeps the infrastructure simple while you learn CI/CD. It also means that every successful CI run clears the database before the Newman tests run. Any data you entered manually into the deployed learning application can therefore be deleted by the next pipeline run.

> In a real production application, CI/testing and production databases should be separated. For PulseVote, we are deliberately using one database for the purpose of learning .

## Part 1 – Create the Render Web Service

Go to Render and create a new Web Service and link it to your PulseVote GitHub repository.

Because PulseVote contains both the backend and frontend in one repository, configure the API service to use:

```text
Root Directory: pulsevote-backend
```

The backend already contains the Dockerfile already created, so choose the Docker runtime.

With the root directory set to `pulsevote-backend`, Render can build using the Dockerfile inside that folder.

Give the service an appropriate name such as:

```text
pulsevote-api
```

Select `main` as the branch.

## Part 2 – Configure the Render environment

Open the service's environment settings and add the environment variables required by the deployed API.

At minimum, configure:

```text
MONGO_URI
JWT_SECRET
USE_HTTPS
PORT
```

Use:

```text
USE_HTTPS=false
PORT=10000
```

`MONGO_URI` must use the same Atlas connection string that you already use for the GitHub Actions pipeline.

`JWT_SECRET` should be a suitable secret for the deployed API. You may also use the same `JWT_SECRET` value that you already use in GitHub Actions.

### Why `USE_HTTPS=false`?

Render provides HTTPS to the public URL and terminates TLS before forwarding traffic to your web service. Your Node application can therefore listen using HTTP inside the service.

### Why `PORT=10000`?

PulseVote already reads the port from:

```javascript
const PORT = process.env.PORT || 5000;
```

Render recommends that web services listen on the port supplied through the `PORT` environment variable. We will explicitly use Render's standard web-service port of `10000`.

The server already listens on:

```javascript
const HOST = '0.0.0.0';
```

which is also required for a Render web service to receive incoming traffic.

## Part 3 – Configure the health check

The API already has `/health`

In the Render service settings, set the health check path to `/health`

A successful deployment should eventually allow you to visit `https://<your-render-service>.onrender.com/health` and receive a successful response.

## Part 4 – Test the first Render deployment

Allow Render to complete its initial deployment while you create the service.

Check the Render logs. You should see the Node application start and connect to MongoDB Atlas.

Then open:

```text
https://<your-render-service>.onrender.com/health
```

Do not continue until the API is healthy on Render.

If deployment fails, check:

* The Root Directory is `pulsevote-backend`
* The service is using the Docker runtime
* `MONGO_URI` uses the same Atlas database used by the GitHub Actions pipeline
* `JWT_SECRET` is set
* `USE_HTTPS=false`
* `PORT=10000`
* Your Atlas Network Access rules allow the hosted application to connect

## Part 5 – Turn off Render's automatic deploys

We now want GitHub Actions to decide when Render may deploy. If Render deploys automatically on every push and GitHub Actions also triggers a deployment, we would have two independent deployment mechanisms.

In the Render service settings, find Auto-Deploy and set it to:

```text
Off
```

The initial deployment you just used to configure and test the service is complete. From this point onward, GitHub Actions will trigger the deployments.

## Part 6 – Get the Render deploy hook

In your Render service settings, find the Deploy Hook URL.

It will look similar to:`https://api.render.com/deploy/srv-...?... `

Do not place this URL directly in your workflow. Anyone who has the deploy hook can trigger a deployment of your service. Copy it.

## Part 7 – Add the deploy hook as a GitHub Secret

Open your PulseVote GitHub repository and go to: `Settings → Secrets and variables → Actions → New repository secret`

Create `RENDER_DEPLOY_HOOK_URL` and paste the Render deploy hook URL as its value.

## Part 8 – Update the GitHub Actions workflow

Open:

```text
.github/workflows/api-ci.yml
```

Change the workflow name from:

```yaml
name: API CI
```

to:

```yaml
name: API CI/CD
```

Do not change the working `api-ci` job from previous activities.

After that job, add a second job at the same indentation level as `api-ci`:

```yaml
  deploy-api:
    name: Deploy API to Render
    needs: api-ci
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest

    steps:
      - name: Trigger Render deployment
        env:
          RENDER_DEPLOY_HOOK_URL: ${{ secrets.RENDER_DEPLOY_HOOK_URL }}
        run: curl --fail-with-body --show-error --silent --request POST "$RENDER_DEPLOY_HOOK_URL"
```

## What does the new job do?

### `needs: api-ci`

```yaml
needs: api-ci
```

creates a dependency between the jobs.

GitHub Actions will not start `deploy-api` until `api-ci` has completed successfully.


### The deployment condition

```yaml
if: github.event_name == 'push' && github.ref == 'refs/heads/main'
```

means the deployment job only runs for a push to `main`.

A pull request into `main` still runs the complete CI job, but the deployment job is skipped.

This is intentional. A pull request should prove that the proposed code works; it should not deploy that proposed code to production before it has been merged.

### Triggering Render

```yaml
RENDER_DEPLOY_HOOK_URL: ${{ secrets.RENDER_DEPLOY_HOOK_URL }}
```

reads the hook from GitHub Secrets.

The command:

```bash
curl --fail-with-body --show-error --silent --request POST "$RENDER_DEPLOY_HOOK_URL"
```

sends an HTTP request to Render's deploy hook.

If Render rejects the deploy-hook request, `curl` exits unsuccessfully and the deployment job fails.

## Complete workflow

Your completed `.github/workflows/api-ci.yml` should now be:

```yaml
name: API CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  api-ci:
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: pulsevote-backend

    steps:
      - name: Check out repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
          cache-dependency-path: pulsevote-backend/package-lock.json

      - name: Install backend dependencies
        run: npm ci

      - name: Lint backend
        run: npm run lint

      - name: Run unit tests with coverage
        run: npm run test:coverage

      - name: Analyse backend with SonarQube Cloud
        uses: SonarSource/sonarqube-scan-action@v8
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        with:
          projectBaseDir: pulsevote-backend
          args: >
            -Dsonar.organization=${{ secrets.SONAR_ORGANIZATION }}
            -Dsonar.projectKey=${{ secrets.SONAR_PROJECT_KEY }}
            -Dsonar.qualitygate.wait=true

      - name: Clear CI test database
        env:
          MONGO_URI: ${{ secrets.MONGO_URI }}
          ALLOW_DB_CLEAR: true
        run: npm run db:clear

      - name: Build API Docker image
        run: docker build -t pulsevote-api-ci .

      - name: Create Docker test network
        run: docker network create pulsevote-ci

      - name: Start containerised API
        run: |
          docker run -d \
            --name pulsevote-api \
            --network pulsevote-ci \
            -e PORT=5000 \
            -e USE_HTTPS=false \
            -e MONGO_URI="${{ secrets.MONGO_URI }}" \
            -e JWT_SECRET="${{ secrets.JWT_SECRET }}" \
            pulsevote-api-ci

      - name: Wait for API health check
        run: |
          for i in {1..30}; do
            STATUS=$(docker inspect --format='{{.State.Health.Status}}' pulsevote-api 2>/dev/null || echo starting)
            echo "API health status: $STATUS"
            if [ "$STATUS" = "healthy" ]; then
              exit 0
            fi
            if [ "$(docker inspect --format='{{.State.Status}}' pulsevote-api 2>/dev/null)" = "exited" ]; then
              docker logs pulsevote-api
              exit 1
            fi
            sleep 2
          done
          docker logs pulsevote-api
          exit 1

      - name: Run existing Postman collection with Newman
        env:
          POSTMAN_TEST_PASSWORD: ${{ secrets.POSTMAN_TEST_PASSWORD }}
        run: |
          docker run --rm \
            --network pulsevote-ci \
            -v "${{ github.workspace }}/pulsevote-backend/postman:/etc/newman" \
            postman/newman:alpine \
            run "/etc/newman/PulseVote RBAC Test.postman_collection.json" \
            --env-var PROTOCOL=http \
            --env-var HOST=pulsevote-api \
            --env-var PORT=5000 \
            --env-var ADMIN_PASSWORD="$POSTMAN_TEST_PASSWORD" \
            --env-var MANAGER_PASSWORD="$POSTMAN_TEST_PASSWORD" \
            --env-var USER_PASSWORD="$POSTMAN_TEST_PASSWORD" \
            --bail

      - name: Show API logs if the pipeline fails
        if: failure()
        run: docker logs pulsevote-api || true

      - name: Clean up Docker resources
        if: always()
        run: |
          docker rm -f pulsevote-api 2>/dev/null || true
          docker network rm pulsevote-ci 2>/dev/null || true

  deploy-api:
    name: Deploy API to Render
    needs: api-ci
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest

    steps:
      - name: Trigger Render deployment
        env:
          RENDER_DEPLOY_HOOK_URL: ${{ secrets.RENDER_DEPLOY_HOOK_URL }}
        run: curl --fail-with-body --show-error --silent --request POST "$RENDER_DEPLOY_HOOK_URL"
```

## Part 9 – Commit and push

Commit the changes and push it to `main`.

For example:

```bash
git add .
git commit -m "Add Render deployment to API pipeline"
git push
```

Open the Actions tab in GitHub.

You should first see `api-ci` run through the complete pipeline. Only after it succeeds should you see `Deploy API to Render` run. Then open the Render service's Events or Logs page. A new deployment should have been triggered.

## Part 10 – Verify the deployed API

After Render reports a successful deployment, open:

```text
https://<your-render-service>.onrender.com/health
```

Confirm that the API responds successfully.

## Test that CI really blocks deployment

The purpose of CD is not merely to automate deployment. The deployment should happen only after the quality checks pass.

Temporarily introduce a harmless lint or test failure on a branch or controlled test commit and observe the workflow.

You should see `api-ci → failed` and `Deploy API to Render → skipped`

You can undo the deliberate failure afterwards.

This demonstrates the important relationship:
```text
CI passes → deployment is allowed
CI fails  → deployment is blocked
```

The frontend has not been added to this deployment pipeline. We will build a similar CI/CD process for the frontend separately later.

