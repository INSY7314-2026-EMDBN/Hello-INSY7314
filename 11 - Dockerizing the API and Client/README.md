# PulseVote – Docker and docker-compose

## Dockerise your backend

### Research
Before you code, spend 10–15 minutes exploring what docker and docker-compose are and why it matters.

Answer briefly:
- What is docker?
- What is docker-compose?
- Why are they an essential skill for developers?
- Can building and running of docker containers automated in any way?
- How do you dockerize a nodejs api?

Write a 4–6 sentence summary in your own words and commit it to your repo.

### Requirements
After this activity, your PulseVote backend will:
- Be dockerised using a `Dockerfile`
- Be included in the root `docker-compose.yml`
- Incorporate `.dockerignore`

Note: you will run the container locally with a command here, but when we get to pipelines, we will run these automatically.

### Code changes

#### 1. Update .gitignore
Update `.gitignore` to not push ssl. This is likely already there, but it would mean that there will be no certificates in your container. This is ok for now.

```.gitignore
node_modules
.env
ssl
.env.*.local
```
#### 2. Update the server.js and app.js code.
Since we will be running our app in a container, some of the code we learnt locally need to be adjusted to cater for a containerized environment.

Notice that HTTPS is not applied in containerized environments. Research the changes and make sure that you understand why you need to disable HTTPS when Dockerizing.

1. Update `app.js`
    ```js
    const express = require('express');
    const cors = require('cors');
    const helmet = require('helmet');
    const dotenv = require('dotenv');
    const authRoutes = require("./routes/authRoutes");
    const organisationRoutes = require("./routes/organisationRoutes");
    const pollRoutes = require("./routes/pollRoutes");
    const { protect } = require("./middleware/authMiddleware");

    dotenv.config();
    const app = express();

    app.use(helmet());

    const CSP_CONNECT = (process.env.CSP_CONNECT || '').split(',').filter(Boolean);
    const defaultConnect = [
    "'self'",
    "http://localhost:5000", "https://localhost:5000",
    "http://localhost:5173", "https://localhost:5173",
    "ws://localhost:5173", "wss://localhost:5173"
    ];

    app.use(
    helmet.contentSecurityPolicy({
        useDefaults: true,
        directives: {
        defaultSrc: ["'self'"],
        scriptSrc: ["'self'", "https://apis.google.com"],
        styleSrc: ["'self'", "'unsafe-inline'", "https://fonts.googleapis.com"],
        fontSrc: ["'self'", "https://fonts.gstatic.com"],
        imgSrc: ["'self'", "data:"],
        connectSrc: CSP_CONNECT.length ? CSP_CONNECT : defaultConnect,
        },
    })
    );

    const allowed = (process.env.CORS_ORIGINS || "http://localhost:5173,https://localhost:5173")
    .split(',')
    .map(s => s.trim());

    app.use(cors({
    origin: (origin, cb) => {
        if (!origin) return cb(null, true);
        if (allowed.includes(origin)) return cb(null, true);
        cb(new Error(`CORS blocked: ${origin}`));
    },
    credentials: true
    }));

    app.use(express.json());
    app.set('trust proxy', 1);

    app.use("/api/auth", authRoutes);
    app.use("/api/organisations", organisationRoutes);
    app.use("/api/polls", pollRoutes);

    app.get('/health', (req, res) => 
    res.status(200).json({
        ok: true,
        ts: Date.now()
    }));

    app.get('/', (req, res) => 
    res.send('PulseVote API running!'));

    app.get('/test', (req, res) => {
    res.json({
        message: 'This is a test endpoint from PulseVote API!',
        status: 'success',
        timestamp: new Date()
    });
    });

    app.get("/api/protected", protect, (req, res) => {
    res.json({
        message: `Welcome, user ${req.user.id}! You have accessed protected data.`,
        timestamp: new Date()
    });
    });

    module.exports = app;
    ```

1. Update `server.js`
    ```js
    const mongoose = require('mongoose');
    const app = require('./app');
    const https = require('https');
    const http = require('http');
    const fs = require('fs');
    require('dotenv').config();

    const PORT = process.env.PORT || 5000;
    const HOST = '0.0.0.0';
    const useHttps = String(process.env.USE_HTTPS || '').toLowerCase() === 'true';
    const mongo = process.env.MONGO_URI;

    if (!mongo) {
    console.error('Missing MONGO_URI');
    process.exit(1);
    }

    mongoose.connect(mongo)
    .then(() => {
        if (useHttps) {
        const keyPath = process.env.SSL_KEY_PATH || 'ssl/key.pem';
        const certPath = process.env.SSL_CERT_PATH || 'ssl/cert.pem';
        const haveFiles = fs.existsSync(keyPath) && fs.existsSync(certPath);

        if (!haveFiles) {
            console.warn('SSL files not found, falling back to HTTP');
        } else {
            const options = { key: fs.readFileSync(keyPath), cert: fs.readFileSync(certPath) };
            https.createServer(options, app).listen(PORT, HOST, () => {
            console.log(`HTTPS server running at https://localhost:${PORT}`);
            });
            return;
        }
        }

        http.createServer(app).listen(PORT, HOST, () => {
        console.log(`HTTP server running at http://localhost:${PORT}`);
        });
    })
    .catch((err) => {
        console.error('MongoDB connection error:', err);
        process.exit(1);
    });
    ```

3. Run your app to make sure it all still runs. Clear your test db in Mongo, then run the tests. They should all run successfully.

#### 3. Dockerfile and root `docker-compose.yml`
Research this in detail - do not blindly copy-paste please...

1. Add the `Dockerfile`
    ```Dockerfile
    FROM node:22-alpine3.20 AS deps
    WORKDIR /app
    COPY package*.json ./
    RUN npm ci --omit=dev

    FROM node:22-alpine3.20 AS runner
    WORKDIR /app
    ENV NODE_ENV=production
    ENV PORT=5000
    EXPOSE 5000

    HEALTHCHECK CMD node -e "require('http').get('http://localhost:'+ (process.env.PORT||5000) +'/health',r=>process.exit(r.statusCode===200?0:1)).on('error',()=>process.exit(1))"

    RUN addgroup -S nodegrp && adduser -S nodeuser -G nodegrp
    USER nodeuser

    COPY --from=deps /app/node_modules ./node_modules
    COPY . .
    CMD ["npm","start"]
    ```
2. Add the `.dockerignore` file - what does this do?
    ```
    node_modules
    npm-debug.log
    Dockerfile*
    .dockerignore
    .git
    .gitignore
    coverage
    .vscode
    .circleci
    ssl
    .env
    .env.*.local
    test
    jest.config.js
    eslint.config.cjs
    ```
3. At the **root of the PulseVote project**, add a `docker-compose.yml` - indenting matters!

    ```yaml
    services:
      api:
        container_name: api
        build:
          context: ./pulsevote-backend
          dockerfile: Dockerfile
        ports:
          - "5000:5000"
        env_file:
          - ./pulsevote-backend/.env
        restart: unless-stopped
    ```

    The `docker-compose.yml` file belongs at the project root because we will later add the frontend as a second service to the same Compose application.


4. Run the nodejs app to make sure it runs. 
    ```
    npm run dev
    ```
    Access at https://localhost:5000

5. Stop the API, return to the **root of the PulseVote project**, and then run the container to make sure it runs. 
    ```bash
    docker compose up --build
    ```
    Access at http://localhost:5000 (notice no s)

6. Update your postman collection to use http instead of https, clear your test db in Mongo, then run the tests.    

#### Troubleshooting if using Mongo locally
If you are using MongoDB locally on your machine, this env will work when running locally:
```
MONGO_URI=mongodb://127.0.0.1:27017/pulsevote
```

However, when using the API in a Docker container, you will need to update the `MONGO_URI` in `.env`
```
MONGO_URI=mongodb://host.docker.internal:27017/pulsevote
```

> `host.docker.internal` allows the Docker container to connect back to MongoDB running on your host computer.

Then run this to start up your containers:
```
docker compose up --build
```

## Dockerise your frontend

Now that your backend runs in a container, you will do the same for the React frontend.

### Research
Before you code, spend 10–15 minutes researching how a Vite/React application is normally deployed in Docker.

Answer briefly:
- Why do we build the React application before serving it in a production container?
- What is a multi-stage Docker build?
- Why can Nginx be used to serve a React application?
- Why should `node_modules`, `dist`, `.env` files and local SSL certificates be excluded from the Docker build context?
- What is the difference between an environment variable used by Node at runtime and a `VITE_` environment variable used during a Vite build?

Add your answers to the research summary in your repo in your own words.

### Requirements
After this activity, your PulseVote frontend will:
- Be dockerised using a `Dockerfile`
- Use Nginx to serve the production React build
- Be added to the existing root `docker-compose.yml`
- Incorporate `.dockerignore`
- Connect to the Dockerised backend over HTTP
- Still support the HTTPS certificates you created for normal local development when those certificate files are present

### Code changes

#### 1. Update `api.js`

The backend container is available at `http://localhost:5000`, rather than `https://localhost:5000`.

Instead of hard-coding the API address, allow Vite to provide it when the frontend is built. Keep the local Docker API as the fallback:

```js
import axios from "axios";

const api = axios.create({
  baseURL: import.meta.env.VITE_API_URL || "http://localhost:5000/api",
  headers: { "Content-Type": "application/json" }
});

api.interceptors.request.use((config) => {
  const token = localStorage.getItem("token");
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

export default api;
```

Research why Vite only exposes environment variables to frontend code when their names begin with `VITE_`.

#### 2. Update `vite.config.js`

Your local frontend previously expected the files in `ssl` to always exist. Those files are deliberately not copied into the Docker image, so the Vite configuration must only enable HTTPS when the certificate files are available.

```js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import fs from 'fs'

const keyPath = 'ssl/key.pem'
const certPath = 'ssl/cert.pem'
const hasHttpsCertificates = fs.existsSync(keyPath) && fs.existsSync(certPath)

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173,
    https: hasHttpsCertificates
      ? {
          key: fs.readFileSync(keyPath),
          cert: fs.readFileSync(certPath),
        }
      : false,
  },
})
```

This means that `npm run dev` can still use HTTPS when your local certificates exist, while `npm run build` can also run inside Docker where the certificates are intentionally absent.

#### 3. Add the frontend `Dockerfile`

We use one stage to install dependencies and build the React application, then a second lightweight Nginx stage to serve the generated static files.

```Dockerfile
FROM node:22-alpine3.20 AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
ARG VITE_API_URL=http://localhost:5000/api
ENV VITE_API_URL=$VITE_API_URL
RUN npm run build

FROM nginx:alpine AS runner
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Notice that the final image does not need Node.js to run the frontend. Node is only used in the build stage.

#### 4. Add `nginx.conf`

React Router handles routes such as `/login` and `/dashboard` in the browser. Nginx therefore needs to return `index.html` when a requested frontend route is not a physical file.

```nginx
server {
    listen 80;
    server_name _;

    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

Research what `try_files` is doing here and why it matters for a single-page application.

#### 5. Add `.dockerignore`

```text
node_modules
npm-debug.log
Dockerfile*
.dockerignore
.git
.gitignore
dist
.vscode
ssl
.env
.env.*.local
```

As with the backend, `.dockerignore` prevents unnecessary or sensitive local files from being sent into the Docker build context.

#### 6. Update the root `docker-compose.yml`

You already created one `docker-compose.yml` at the root of the project for the backend. Do not create a second Compose file inside the frontend folder.

Update the existing root file so that it now defines both the backend and frontend services:

```yaml
services:
  api:
    container_name: api
    build:
      context: ./pulsevote-backend
      dockerfile: Dockerfile
    ports:
      - "5000:5000"
    env_file:
      - ./pulsevote-backend/.env
    restart: unless-stopped

  frontend:
    container_name: frontend
    build:
      context: ./pulsevote-frontend
      dockerfile: Dockerfile
      args:
        VITE_API_URL: http://localhost:5000/api
    ports:
      - "5173:80"
    depends_on:
      - api
    restart: unless-stopped
```

The browser still accesses the frontend at port `5173`, but Docker maps that host port to Nginx on port `80` inside the container. The `depends_on` entry tells Docker Compose to start the API service before the frontend service.

Notice that both services are now managed from one Compose file. This is one of the main benefits of Docker Compose: related containers can be built and started together as one application.

#### 7. Run and test the frontend locally

Before testing Docker, make sure the normal React development server still starts:

```bash
npm run dev
```

If your `ssl/key.pem` and `ssl/cert.pem` files exist, access it at:

```text
https://localhost:5173
```

If those files are not present, Vite will fall back to:

```text
http://localhost:5173
```

#### 8. Build and run the complete application
Stop the normal Vite development server, return to the root of the PulseVote project, and run:

```bash
docker compose up --build
```

Docker Compose will now build and start both the backend and frontend containers.

Access the Dockerised frontend at:

```text
http://localhost:5173
```

The frontend should communicate with the Dockerised backend at:

```text
http://localhost:5000/api
```

Make sure the backend container is running as well before testing login, registration, organisations and polls.

#### 9. Test the complete application

With the root Compose application running, confirm that:
- The frontend opens at `http://localhost:5173`
- The backend health endpoint responds at `http://localhost:5000/health`
- Registration and login still work
- Protected API requests still send the JWT token
- Organisation and poll functionality still works
- Refreshing a frontend route such as `/login` or `/dashboard` does not produce an Nginx 404 error

You should understand each Docker and Nginx configuration line before committing the changes. 

Soon, we will automate container builds later when we work with pipelines.
