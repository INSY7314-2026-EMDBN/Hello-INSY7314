# PulseVote – GitHub Actions CI with Docker and Newman

In the previous activities, you already:

- Built and secured the PulseVote API
- Added RBAC and tested the RBAC flow using the PulseVote RBAC Test Postman collection
- Added rate limiting
- Added ESLint and Jest unit testing
- Dockerised the API and frontend

We will now automate the backend checks using GitHub Actions.

The important idea is that we are not creating a new set of API tests. We are taking the Postman collection that you used in Activity 7 – Adding RBAC and running that same collection automatically with Newman.

## Research

Before you code, spend 10–15 minutes researching GitHub Actions and Newman.

Answer briefly:

- What is Continuous Integration (CI)?
- What is GitHub Actions?
- What is a GitHub Actions workflow, job and step?
- What is the difference between `push` and `pull_request` triggers?
- Why should `npm ci` normally be used instead of `npm install` in a CI pipeline?
- What is Newman?
- Why is it useful to run a Postman collection automatically in a pipeline?
- Why should passwords, database connection strings and JWT secrets be stored as GitHub Secrets rather than written into the workflow file?
- Why should integration tests start from a known database state?

Write a 4–6 sentence summary in your own words and commit it to your repo.

## Requirements

After this activity, your PulseVote backend pipeline will:
- Run automatically on pushes and pull requests to `main`
- Install the backend dependencies
- Run ESLint
- Run the Jest unit tests from Activity 10
- Clear a dedicated MongoDB Atlas CI test database before the API tests
- Build the backend Docker image from Activity 11
- Start the containerised API
- Run the existing PulseVote RBAC Test Postman collection with Newman
- Fail if one of the automated Postman requests does not return a successful HTTP response
- Keep all passwords, the MongoDB connection string and the JWT secret out of GitHub

## Before you start – MongoDB Atlas

Some of you have been using MongoDB locally. That was fine while the API and tests were running on your own computer, but a GitHub-hosted Actions runner cannot connect to `mongodb://127.0.0.1:27017/...` on your laptop.

Create a **separate database for CI testing**, for example:

```text
pulsevote_ci
```

Your Atlas connection string stored in GitHub should therefore point to that test database, for example:

```text
mongodb+srv://<username>:<password>@<cluster>/pulsevote_ci?retryWrites=true&w=majority
```
Because GitHub-hosted runners do not have one permanent public IP address, make sure your Atlas Network Access settings allow the runner to connect. For a classroom/test cluster, you may allow access from anywhere (`0.0.0.0/0`). 

## Why clear the database first?

You have already been doing this manually when running the RBAC Postman collection. Activity 11 specifically required you to **clear your test database in Mongo before running the tests**.

The first request in the collection creates the first administrator. If data from a previous run is still present, the API correctly refuses to create another first admin, and the test sequence is no longer starting from a predictable state.

In a CI pipeline, this reset must happen automatically.

We will add a small Node script that connects using `MONGO_URI` and calls MongoDB's `dropDatabase()` before Newman runs. The script contains two safeguards:

1. `ALLOW_DB_CLEAR` must explicitly be set to `true`.
2. The database name must contain `test` or `ci`.

This is why your Atlas URI for this activity should point to something such as `pulsevote_ci`.

## Code changes

### 1. Reuse the Postman collection from Activity 7

You already created the collection earlier. Do not build another collection.

Copy/export **PulseVote RBAC Test.postman_collection.json** into a new `postman` folder inside the backend:

```text
pulsevote-backend/
├── postman/
│   └── PulseVote RBAC Test.postman_collection.json
```

The collection already uses variables such as:

```text
{{PROTOCOL}}
{{HOST}}
{{PORT}}
{{ADMIN_PASSWORD}}
{{MANAGER_PASSWORD}}
{{USER_PASSWORD}}
```

This is useful because Newman can replace those values when the collection runs in GitHub Actions.

The collection from Activity 7 also stores tokens and IDs as it runs. For example, the Admin login stores `ADMIN_TOKEN`, the organisation request stores `ORG_ID`, and the poll request stores `POLL_ID`. This allows the requests to execute in sequence as one integration test flow.

### 2. Add a collection-level test

The Activity 7 collection used Postman scripts mainly to capture tokens and IDs for later requests. Now that the collection will act as a CI quality gate, add one collection-level Post-response test so that an unsuccessful API response fails Newman:

```javascript
pm.test('Request returns a successful HTTP status', function () {
    pm.expect(pm.response.code).to.be.within(200, 299);
});
```

Add this at collection level, not separately to every request. It will then run after each request in the collection.

This means a `400`, `401`, `403`, `429` or `500` response will cause the Newman stage to fail rather than allowing a broken request sequence to appear successful.

### 3. Add the database clearing script

Create:

```text
pulsevote-backend/scripts/clearDatabase.js
```

Add:

```javascript
const mongoose = require('mongoose');
require('dotenv').config();

const mongo = process.env.MONGO_URI;
const allowClear = String(process.env.ALLOW_DB_CLEAR || '').toLowerCase() === 'true';

async function clearDatabase() {
  if (!mongo) {
    throw new Error('Missing MONGO_URI');
  }

  if (!allowClear) {
    throw new Error('Database clear blocked. Set ALLOW_DB_CLEAR=true only in the CI test job.');
  }

  await mongoose.connect(mongo);
  const dbName = mongoose.connection.name;

  if (!/(test|ci)/i.test(dbName)) {
    throw new Error(`Refusing to drop database '${dbName}'. The CI database name must contain test or ci.`);
  }

  await mongoose.connection.db.dropDatabase();
  console.log(`Cleared MongoDB database: ${dbName}`);
  await mongoose.disconnect();
}

clearDatabase().catch(async (err) => {
  console.error(err.message);
  try {
    await mongoose.disconnect();
  } catch {
  }
  process.exit(1);
});
```

### 4. Update `package.json`

Add a script for the database reset:

```json
"scripts": {
  "start": "node server.js",
  "dev": "npx nodemon server.js",
  "test": "jest --passWithNoTests",
  "lint": "eslint .",
  "db:clear": "node scripts/clearDatabase.js"
}
```

Test the safety check locally from the backend folder without setting `ALLOW_DB_CLEAR`:

```bash
npm run db:clear
```

Never test the actual clear command against a database containing data that you need.

### 5. Update `.dockerignore`

The production API container does not need the Postman collection or the database reset utility.

Add these lines to `pulsevote-backend/.dockerignore`:

```text
postman
scripts/clearDatabase.js
```

### 6. Add the GitHub repository secrets

Open your PulseVote repository on GitHub and go to:

```text
Settings → Secrets and variables → Actions → New repository secret
```

Create these secrets:

```text
MONGO_URI
JWT_SECRET
POSTMAN_TEST_PASSWORD
```

`MONGO_URI` must point to your Mongo Atlas database.

`JWT_SECRET` is used by the running API to sign and verify the JWTs created during the Postman flow. You can use the same as local.

`POSTMAN_TEST_PASSWORD` can be one valid password that is supplied to the admin, manager and user variables when Newman runs. It must satisfy the password rules already implemented in PulseVote: at least eight characters, including a letter and a number. You can use the same as local.

Do not place the real values in your YAML file and do not commit a `.env` file.

### 7. Create the GitHub Actions workflow

At the root of the PulseVote repository, create:

```text
.github/
└── workflows/
    └── api-ci.yml
```

Add:

```yaml
name: API CI

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

      - name: Run unit tests
        run: npm test

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
        run: |
          docker run --rm \
            --network pulsevote-ci \
            -v "${{ github.workspace }}/pulsevote-backend/postman:/etc/newman" \
            postman/newman:alpine \
            run "/etc/newman/PulseVote RBAC Test.postman_collection.json" \
            --env-var PROTOCOL=http \
            --env-var HOST=pulsevote-api \
            --env-var PORT=5000 \
            --env-var ADMIN_PASSWORD="${{ secrets.POSTMAN_TEST_PASSWORD }}" \
            --env-var MANAGER_PASSWORD="${{ secrets.POSTMAN_TEST_PASSWORD }}" \
            --env-var USER_PASSWORD="${{ secrets.POSTMAN_TEST_PASSWORD }}" \
            --bail

      - name: Show API logs if the pipeline fails
        if: failure()
        run: docker logs pulsevote-api || true

      - name: Clean up Docker resources
        if: always()
        run: |
          docker rm -f pulsevote-api 2>/dev/null || true
          docker network rm pulsevote-ci 2>/dev/null || true
```

## What is the workflow doing?

### Checkout and Node setup

```yaml
uses: actions/checkout@v4
```

downloads the repository onto the temporary GitHub-hosted runner.

```yaml
uses: actions/setup-node@v4
```

configures Node 22 and enables npm dependency caching.

### Linting and unit tests

```yaml
npm run lint
npm test
```

Reuse the work from Activity 10. If either fails, GitHub Actions stops the job.

### Clearing the CI database

```yaml
MONGO_URI: ${{ secrets.MONGO_URI }}
ALLOW_DB_CLEAR: true
```

This allows the reset script to connect to the dedicated Atlas test database. The script drops the database so the RBAC sequence always begins without an existing administrator, organisation or poll.

This gives us a repeatable test run instead of depending on whatever data happened to be left behind by the previous pipeline execution.

### Building and running Docker

```yaml
docker build -t pulsevote-api-ci .
```

builds the API using the Dockerfile you created in Activity 11.

We then create a Docker network and run the API container on it. The API connects to MongoDB Atlas using the secret `MONGO_URI`.

The workflow waits for the Docker `HEALTHCHECK` from Activity 11 to report `healthy` before Newman begins.

### Running Newman in Docker

Newman is also run as a container:

```yaml
postman/newman:alpine
```

Both containers join `pulsevote-ci`, so Newman can reach the API using the container name:

```text
http://pulsevote-api:5000
```

We do not edit the collection every time the environment changes. Instead, Newman overrides the collection variables:

```text
PROTOCOL=http
HOST=pulsevote-api
PORT=5000
```

The password values come from the GitHub secret rather than the collection file.

### Cleaning up

The final step uses:

```yaml
if: always()
```

so the temporary API container and Docker network are removed whether the pipeline passes or fails.

## Run the pipeline

Commit and push the changes:

```bash
git add .
git commit -m "Add API CI pipeline with Docker and Newman"
git push
```

Open your repository on GitHub and select the Actions tab.

You should see the API CI workflow run.

A successful run should show green steps for:
- Install backend dependencies
- Lint backend
- Run unit tests
- Clear CI test database
- Build API Docker image
- Start containerised API
- Wait for API health check
- Run existing Postman collection with Newman

Open the Newman step and inspect the request output. You should see the same RBAC flow that you previously ran manually in Postman.

## Test that CI actually protects the project

Do not only prove that the workflow can pass.

Make a small temporary change that causes one unit test, lint check or Postman request to fail, push it, and confirm that the Actions workflow turns red.

Undo the temporary change and push again.

The final workflow should be green.

## Final structure

The relevant additions should now look like:

```text
PulseVote/
├── .github/
│   └── workflows/
│       └── api-ci.yml
├── pulsevote-backend/
│   ├── postman/
│   │   └── PulseVote RBAC Test.postman_collection.json
│   ├── scripts/
│   │   └── clearDatabase.js
│   ├── Dockerfile
│   ├── package.json
│   └── ...
├── pulsevote-frontend/
└── docker-compose.yml
```

## What have we achieved?

You previously ran linting, unit tests, Docker and Postman manually. You have now connected those existing pieces into a Continuous Integration pipeline:

In the next activity, we will extend this pipeline with automated code-quality analysis using SonarQube Cloud.
