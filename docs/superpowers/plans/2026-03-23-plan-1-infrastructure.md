# Infrastructure & Foundation — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up a fully running Docker Compose stack on a Hetzner CX41 VPS with Cal.com, n8n, PostgreSQL, Redis, nginx (SSL), and a webhook receiver service — ready to receive verified Meta webhook calls.

**Architecture:** All services run in Docker Compose on a single Hetzner VPS behind an nginx reverse proxy with Let's Encrypt SSL. A lightweight Express service receives and HMAC-verifies incoming Meta webhooks before forwarding to n8n. PostgreSQL is the shared database with UTF-8 encoding and the initial schema applied via migration.

**Tech Stack:** Docker, Docker Compose, Cal.com (self-hosted), n8n, PostgreSQL 15, Redis 7, nginx, Node.js 20 (Express webhook receiver), Certbot (Let's Encrypt)

---

## Project File Structure

```
appointment-chatbot/           ← new repo root
├── docker-compose.yml
├── docker-compose.override.yml   (local dev overrides, git-ignored)
├── .env.example
├── .env                          (git-ignored)
├── nginx/
│   ├── nginx.conf.template       (${DOMAIN} substituted at container startup via envsubst)
│   └── certbot/                  (Let's Encrypt certs, git-ignored)
├── webhook/
│   ├── package.json
│   ├── package-lock.json
│   ├── Dockerfile
│   ├── src/
│   │   ├── index.js              (Express app entry point)
│   │   ├── middleware/
│   │   │   ├── hmac.js           (HMAC-SHA256 verification)
│   │   │   └── rateLimiter.js    (per-IP rate limiting)
│   │   └── routes/
│   │       ├── whatsapp.js       (WhatsApp webhook route)
│   │       └── instagram.js      (Instagram webhook route)
│   └── tests/
│       ├── hmac.test.js
│       ├── whatsapp.test.js
│       └── instagram.test.js
└── db/
    └── migrations/
        └── 001_initial.sql       (full schema from spec Section 4)
```

---

## Task 1: Create the Repository

**Files:**
- Create: `appointment-chatbot/` (new git repo)
- Create: `.gitignore`
- Create: `README.md`

- [ ] **Step 1: Create the project directory and init git**

```bash
mkdir appointment-chatbot
cd appointment-chatbot
git init
```

- [ ] **Step 2: Create .gitignore**

```
.env
docker-compose.override.yml
nginx/certbot/
node_modules/
*.log
.DS_Store
```

- [ ] **Step 3: Create README.md**

```markdown
# Appointment Chatbot SaaS

AI-powered appointment booking chatbot for WhatsApp and Instagram.

See `docs/` for architecture spec and implementation plans.
```

- [ ] **Step 4: Commit**

```bash
git add .
git commit -m "chore: init repository"
```

---

## Task 2: Environment Variables Template

**Files:**
- Create: `.env.example`

- [ ] **Step 1: Create .env.example**

```bash
# PostgreSQL
POSTGRES_USER=chatbot
POSTGRES_PASSWORD=changeme
POSTGRES_DB=chatbot_db

# Cal.com
CALCOM_LICENSE_KEY=
NEXTAUTH_SECRET=changeme_generate_with_openssl
NEXTAUTH_URL=https://yourdomain.com/cal
NEXT_PUBLIC_WEBAPP_URL=https://yourdomain.com/cal
DATABASE_URL=postgresql://chatbot:changeme@postgres:5432/chatbot_db
REDIS_URL=redis://redis:6379

# n8n
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=changeme
N8N_HOST=yourdomain.com
N8N_PORT=5678
N8N_PROTOCOL=https
WEBHOOK_URL=https://yourdomain.com/n8n/

# Webhook receiver
WEBHOOK_PORT=3001
WHATSAPP_APP_SECRET=from_meta_app_dashboard
WHATSAPP_VERIFY_TOKEN=choose_a_random_string
INSTAGRAM_APP_SECRET=from_meta_app_dashboard
INSTAGRAM_VERIFY_TOKEN=choose_a_random_string
N8N_WEBHOOK_URL=http://n8n:5678/webhook

# Domain
DOMAIN=yourdomain.com
CERTBOT_EMAIL=you@youremail.com
```

- [ ] **Step 2: Copy to .env and fill in values for local dev**

```bash
cp .env.example .env
# Edit .env — for local dev, set DOMAIN=localhost and skip certbot
```

- [ ] **Step 3: Commit**

```bash
git add .env.example
git commit -m "chore: add env vars template"
```

---

## Task 3: Database Migration

**Files:**
- Create: `db/migrations/001_initial.sql`

- [ ] **Step 1: Write the migration**

```sql
-- db/migrations/001_initial.sql
-- Run once on first boot via Docker entrypoint

CREATE TABLE IF NOT EXISTS businesses (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  whatsapp_number TEXT UNIQUE,
  instagram_account_id TEXT UNIQUE,
  calcom_org_id TEXT,
  system_prompt TEXT,
  staff_mode TEXT NOT NULL DEFAULT 'auto' CHECK (staff_mode IN ('auto', 'specific')),
  monthly_token_count INTEGER NOT NULL DEFAULT 0,
  token_limit INTEGER NOT NULL DEFAULT 500000,
  active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS staff (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id UUID NOT NULL REFERENCES businesses(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  calcom_user_id TEXT,
  specialties TEXT[],
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS conversations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id UUID NOT NULL REFERENCES businesses(id) ON DELETE CASCADE,
  customer_identifier TEXT NOT NULL,
  channel TEXT NOT NULL CHECK (channel IN ('whatsapp', 'instagram')),
  state TEXT NOT NULL DEFAULT 'IDLE',
  last_activity_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE(business_id, customer_identifier, channel)
);

CREATE TABLE IF NOT EXISTS messages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  conversation_id UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
  role TEXT NOT NULL CHECK (role IN ('user', 'assistant')),
  content TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_messages_conversation_id
  ON messages(conversation_id, created_at DESC);

CREATE TABLE IF NOT EXISTS whatsapp_opt_ins (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id UUID NOT NULL REFERENCES businesses(id) ON DELETE CASCADE,
  customer_phone TEXT NOT NULL,
  opted_in BOOLEAN NOT NULL DEFAULT false,
  consented_at TIMESTAMPTZ,
  expires_at TIMESTAMPTZ,
  UNIQUE(business_id, customer_phone)
);

CREATE TABLE IF NOT EXISTS reminders_sent (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id UUID NOT NULL REFERENCES businesses(id) ON DELETE CASCADE,
  booking_id TEXT NOT NULL,
  customer_identifier TEXT NOT NULL,
  sent_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE(business_id, booking_id)
);

CREATE TABLE IF NOT EXISTS subscriptions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id UUID NOT NULL UNIQUE REFERENCES businesses(id) ON DELETE CASCADE,
  iyzico_subscription_id TEXT UNIQUE,
  iyzico_customer_id TEXT,
  plan TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'active'
    CHECK (status IN ('active', 'paused', 'cancelled', 'grace_period')),
  next_billing_date DATE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

- [ ] **Step 2: Commit**

```bash
git add db/
git commit -m "feat: add initial database schema migration"
```

---

## Task 4: Webhook Receiver — Express App

**Files:**
- Create: `webhook/package.json`
- Create: `webhook/src/index.js`
- Create: `webhook/src/middleware/hmac.js`
- Create: `webhook/src/middleware/rateLimiter.js`
- Create: `webhook/src/routes/whatsapp.js`
- Create: `webhook/src/routes/instagram.js`
- Create: `webhook/Dockerfile`

- [ ] **Step 1: Init Node project**

```bash
mkdir -p webhook/src/middleware webhook/src/routes webhook/tests
cd webhook
npm init -y
npm install express express-rate-limit axios
npm install --save-dev jest supertest
```

- [ ] **Step 2: Add test script to package.json**

In `webhook/package.json`, update the scripts section:
```json
"scripts": {
  "start": "node src/index.js",
  "test": "jest --runInBand"
}
```

- [ ] **Step 3: Write HMAC middleware**

```javascript
// webhook/src/middleware/hmac.js
const crypto = require('crypto');

function verifyHmac(appSecret) {
  return (req, res, next) => {
    const signature = req.headers['x-hub-signature-256'];
    if (!signature) {
      return res.status(403).json({ error: 'Missing signature' });
    }

    const expected = 'sha256=' + crypto
      .createHmac('sha256', appSecret)
      .update(req.rawBody)  // rawBody set by bodyParser below
      .digest('hex');

    // timingSafeEqual requires equal-length buffers — check length first
    const sigBuf = Buffer.from(signature);
    const expBuf = Buffer.from(expected);
    if (sigBuf.length !== expBuf.length) {
      return res.status(403).json({ error: 'Invalid signature' });
    }

    const isValid = crypto.timingSafeEqual(sigBuf, expBuf);

    if (!isValid) {
      return res.status(403).json({ error: 'Invalid signature' });
    }

    next();
  };
}

module.exports = { verifyHmac };
```

- [ ] **Step 4: Write rate limiter middleware**

```javascript
// webhook/src/middleware/rateLimiter.js
const rateLimit = require('express-rate-limit');

const webhookLimiter = rateLimit({
  windowMs: 1 * 60 * 1000,  // 1 minute
  max: 300,                   // max 300 requests per minute per IP
  standardHeaders: true,
  legacyHeaders: false,
  message: { error: 'Too many requests' }
});

module.exports = { webhookLimiter };
```

- [ ] **Step 5: Write WhatsApp route**

```javascript
// webhook/src/routes/whatsapp.js
const express = require('express');
const axios = require('axios');
const { verifyHmac } = require('../middleware/hmac');

const router = express.Router();
const appSecret = process.env.WHATSAPP_APP_SECRET;
const n8nUrl = process.env.N8N_WEBHOOK_URL;

// Meta webhook verification (GET)
router.get('/', (req, res) => {
  const mode = req.query['hub.mode'];
  const token = req.query['hub.verify_token'];
  const challenge = req.query['hub.challenge'];

  if (mode === 'subscribe' && token === process.env.WHATSAPP_VERIFY_TOKEN) {
    return res.status(200).send(challenge);
  }
  res.status(403).json({ error: 'Verification failed' });
});

// Inbound messages (POST)
router.post('/', verifyHmac(appSecret), async (req, res) => {
  // Acknowledge immediately — Meta requires 200 within 5 seconds
  res.status(200).send('OK');

  try {
    await axios.post(`${n8nUrl}/whatsapp`, req.body, { timeout: 5000 });
  } catch (err) {
    console.error('Failed to forward WhatsApp webhook to n8n:', err.message);
  }
});

module.exports = router;
```

- [ ] **Step 6: Write Instagram route**

```javascript
// webhook/src/routes/instagram.js
const express = require('express');
const axios = require('axios');
const { verifyHmac } = require('../middleware/hmac');

const router = express.Router();
const appSecret = process.env.INSTAGRAM_APP_SECRET;
const n8nUrl = process.env.N8N_WEBHOOK_URL;

// Meta webhook verification (GET)
router.get('/', (req, res) => {
  const mode = req.query['hub.mode'];
  const token = req.query['hub.verify_token'];
  const challenge = req.query['hub.challenge'];

  if (mode === 'subscribe' && token === process.env.INSTAGRAM_VERIFY_TOKEN) {
    return res.status(200).send(challenge);
  }
  res.status(403).json({ error: 'Verification failed' });
});

// Inbound messages (POST)
router.post('/', verifyHmac(appSecret), async (req, res) => {
  res.status(200).send('OK');

  try {
    await axios.post(`${n8nUrl}/instagram`, req.body, { timeout: 5000 });
  } catch (err) {
    console.error('Failed to forward Instagram webhook to n8n:', err.message);
  }
});

module.exports = router;
```

- [ ] **Step 7: Write Express entry point**

```javascript
// webhook/src/index.js
const express = require('express');
const { webhookLimiter } = require('./middleware/rateLimiter');
const whatsappRoutes = require('./routes/whatsapp');
const instagramRoutes = require('./routes/instagram');

const app = express();
const PORT = process.env.WEBHOOK_PORT || 3001;

// Parse JSON but also preserve rawBody for HMAC verification
app.use((req, res, next) => {
  express.json({
    verify: (req, res, buf) => { req.rawBody = buf; }
  })(req, res, next);
});

app.use(webhookLimiter);
app.use('/webhook/whatsapp', whatsappRoutes);
app.use('/webhook/instagram', instagramRoutes);

app.get('/health', (req, res) => res.json({ status: 'ok' }));

app.listen(PORT, () => {
  console.log(`Webhook receiver running on port ${PORT}`);
});

module.exports = app;
```

- [ ] **Step 8: Write Dockerfile**

```dockerfile
# webhook/Dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY src/ ./src/
EXPOSE 3001
CMD ["node", "src/index.js"]
```

- [ ] **Step 9: Commit**

```bash
cd ..
git add webhook/
git commit -m "feat: add webhook receiver with HMAC verification"
```

---

## Task 5: Tests for Webhook Receiver

**Files:**
- Create: `webhook/tests/hmac.test.js`
- Create: `webhook/tests/whatsapp.test.js`
- Create: `webhook/tests/instagram.test.js`

- [ ] **Step 1: Write HMAC tests first (TDD — write before running)**

```javascript
// webhook/tests/hmac.test.js
const crypto = require('crypto');
const { verifyHmac } = require('../src/middleware/hmac');

function makeReq(body, secret, tampered = false) {
  const raw = Buffer.from(JSON.stringify(body));
  const sig = 'sha256=' + crypto.createHmac('sha256', secret).update(raw).digest('hex');
  return {
    headers: { 'x-hub-signature-256': tampered ? 'sha256=deadbeef' : sig },
    rawBody: raw,
    body
  };
}

function makeRes() {
  const res = { status: jest.fn().mockReturnThis(), json: jest.fn() };
  return res;
}

test('passes valid HMAC signature', () => {
  const next = jest.fn();
  const req = makeReq({ test: 1 }, 'mysecret');
  verifyHmac('mysecret')(req, makeRes(), next);
  expect(next).toHaveBeenCalled();
});

test('rejects tampered signature with 403', () => {
  const res = makeRes();
  const next = jest.fn();
  const req = makeReq({ test: 1 }, 'mysecret', true);
  verifyHmac('mysecret')(req, res, next);
  expect(res.status).toHaveBeenCalledWith(403);
  expect(next).not.toHaveBeenCalled();
});

test('rejects missing signature header with 403', () => {
  const res = makeRes();
  const next = jest.fn();
  const req = { headers: {}, rawBody: Buffer.from('{}'), body: {} };
  verifyHmac('mysecret')(req, res, next);
  expect(res.status).toHaveBeenCalledWith(403);
});
```

- [ ] **Step 2: Run HMAC tests — expect them to pass (middleware already written)**

```bash
cd webhook
npx jest tests/hmac.test.js --verbose
```

Expected: 3 passing tests.

- [ ] **Step 3: Write WhatsApp route tests**

```javascript
// webhook/tests/whatsapp.test.js
process.env.WHATSAPP_APP_SECRET = 'testsecret';
process.env.WHATSAPP_VERIFY_TOKEN = 'testverifytoken';
process.env.N8N_WEBHOOK_URL = 'http://localhost:9999';

const request = require('supertest');
const crypto = require('crypto');
const app = require('../src/index');

function signedPost(path, body, secret) {
  const raw = JSON.stringify(body);
  const sig = 'sha256=' + crypto.createHmac('sha256', secret).update(raw).digest('hex');
  return request(app)
    .post(path)
    .set('x-hub-signature-256', sig)
    .set('content-type', 'application/json')
    .send(body);
}

test('GET /webhook/whatsapp returns challenge on valid verify token', async () => {
  const res = await request(app)
    .get('/webhook/whatsapp')
    .query({ 'hub.mode': 'subscribe', 'hub.verify_token': 'testverifytoken', 'hub.challenge': 'abc123' });
  expect(res.status).toBe(200);
  expect(res.text).toBe('abc123');
});

test('GET /webhook/whatsapp returns 403 on wrong verify token', async () => {
  const res = await request(app)
    .get('/webhook/whatsapp')
    .query({ 'hub.mode': 'subscribe', 'hub.verify_token': 'wrongtoken', 'hub.challenge': 'abc123' });
  expect(res.status).toBe(403);
});

test('POST /webhook/whatsapp returns 200 with valid HMAC', async () => {
  const res = await signedPost('/webhook/whatsapp', { entry: [] }, 'testsecret');
  expect(res.status).toBe(200);
});

test('POST /webhook/whatsapp returns 403 with invalid HMAC', async () => {
  const res = await request(app)
    .post('/webhook/whatsapp')
    .set('x-hub-signature-256', 'sha256=invalid')
    .send({ entry: [] });
  expect(res.status).toBe(403);
});
```

- [ ] **Step 4: Run WhatsApp tests**

```bash
npx jest tests/whatsapp.test.js --verbose
```

Expected: 4 passing tests.

- [ ] **Step 5: Write Instagram route tests (same pattern)**

```javascript
// webhook/tests/instagram.test.js
process.env.INSTAGRAM_APP_SECRET = 'instasecret';
process.env.INSTAGRAM_VERIFY_TOKEN = 'instaverifytoken';
process.env.N8N_WEBHOOK_URL = 'http://localhost:9999';

const request = require('supertest');
const crypto = require('crypto');
const app = require('../src/index');

function signedPost(path, body, secret) {
  const raw = JSON.stringify(body);
  const sig = 'sha256=' + crypto.createHmac('sha256', secret).update(raw).digest('hex');
  return request(app)
    .post(path)
    .set('x-hub-signature-256', sig)
    .set('content-type', 'application/json')
    .send(body);
}

test('GET /webhook/instagram returns challenge on valid token', async () => {
  const res = await request(app)
    .get('/webhook/instagram')
    .query({ 'hub.mode': 'subscribe', 'hub.verify_token': 'instaverifytoken', 'hub.challenge': 'xyz789' });
  expect(res.status).toBe(200);
  expect(res.text).toBe('xyz789');
});

test('POST /webhook/instagram returns 200 with valid HMAC', async () => {
  const res = await signedPost('/webhook/instagram', { entry: [] }, 'instasecret');
  expect(res.status).toBe(200);
});

test('POST /webhook/instagram returns 403 with invalid HMAC', async () => {
  const res = await request(app)
    .post('/webhook/instagram')
    .set('x-hub-signature-256', 'sha256=invalid')
    .send({ entry: [] });
  expect(res.status).toBe(403);
});
```

- [ ] **Step 6: Run all tests**

```bash
npx jest --verbose
```

Expected: all 10 tests passing.

- [ ] **Step 7: Commit**

```bash
cd ..
git add webhook/tests/
git commit -m "test: add webhook receiver tests (HMAC, WhatsApp, Instagram routes)"
```

---

## Task 6: nginx Configuration

nginx does not natively substitute shell variables in config files. The solution: store `nginx.conf.template` with `${DOMAIN}` placeholders and use `envsubst` (built into `nginx:alpine`) in the container entrypoint to generate the final `nginx.conf` at startup.

**Files:**
- Create: `nginx/nginx.conf.template`

- [ ] **Step 1: Write nginx config template**

```nginx
# nginx/nginx.conf.template
# ${DOMAIN} is substituted at container startup via envsubst

events { worker_connections 1024; }

http {
  upstream calcom    { server calcom:3000; }
  upstream n8n       { server n8n:5678; }
  upstream webhook   { server webhook:3001; }

  # Redirect HTTP → HTTPS
  server {
    listen 80;
    server_name ${DOMAIN};
    location /.well-known/acme-challenge/ { root /var/www/certbot; }
    location / { return 301 https://$host$request_uri; }
  }

  server {
    listen 443 ssl;
    server_name ${DOMAIN};

    ssl_certificate     /etc/letsencrypt/live/${DOMAIN}/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/${DOMAIN}/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;

    # Cal.com dashboard
    location /cal/ {
      proxy_pass http://calcom/;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
    }

    # n8n (IP-restricted — replace YOUR_ADMIN_IP with your actual IP)
    location /n8n/ {
      allow YOUR_ADMIN_IP/32;
      deny all;
      proxy_pass http://n8n/;
      proxy_set_header Host $host;
      proxy_http_version 1.1;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";
    }

    # Meta webhooks
    location /webhook/ {
      proxy_pass http://webhook/webhook/;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
    }

    location /health {
      proxy_pass http://webhook/health;
    }
  }
}
```

- [ ] **Step 2: Update nginx service in docker-compose.yml to use envsubst**

In `docker-compose.yml`, replace the nginx service volumes and add a command:

```yaml
  nginx:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - calcom
      - n8n
      - webhook
    environment:
      DOMAIN: ${DOMAIN}
    volumes:
      - ./nginx/nginx.conf.template:/etc/nginx/nginx.conf.template:ro
      - ./nginx/certbot/conf:/etc/letsencrypt
      - ./nginx/certbot/www:/var/www/certbot
    command: >
      sh -c "envsubst '$$DOMAIN' < /etc/nginx/nginx.conf.template
             > /etc/nginx/nginx.conf && nginx -g 'daemon off;'"
    networks:
      - internal
      - external
```

Note: `$$DOMAIN` (double `$`) prevents Docker Compose from interpolating the variable before passing it to `envsubst`. `envsubst` then substitutes `${DOMAIN}` in the template.

- [ ] **Step 3: Also update the local dev override to skip SSL**

```yaml
# docker-compose.override.yml
services:
  nginx:
    ports:
      - "8080:80"
    command: >
      sh -c "envsubst '$$DOMAIN' < /etc/nginx/nginx.conf.template
             > /etc/nginx/nginx.conf.http && nginx -c /etc/nginx/nginx.conf.http -g 'daemon off;'"
```

> For local dev the 443 block won't work (no certs). The simplest local approach: skip nginx entirely and access services directly via the exposed ports in the override.

- [ ] **Step 4: Commit**

```bash
git add nginx/
git commit -m "feat: add nginx reverse proxy config with envsubst SSL"
```

---

## Task 7: Docker Compose

**Files:**
- Create: `docker-compose.yml`
- Create: `docker-compose.override.yml` (local dev, git-ignored)

- [ ] **Step 1: Write docker-compose.yml**

```yaml
# docker-compose.yml

services:
  postgres:
    image: postgres:15-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_INITDB_ARGS: "--encoding=UTF8 --lc-collate=tr_TR.UTF-8 --lc-ctype=tr_TR.UTF-8"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./db/migrations/001_initial.sql:/docker-entrypoint-initdb.d/001_initial.sql
    networks:
      - internal

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    networks:
      - internal

  calcom:
    image: calcom/cal.com:latest
    restart: unless-stopped
    depends_on:
      - postgres
      - redis
    environment:
      DATABASE_URL: ${DATABASE_URL}
      NEXTAUTH_SECRET: ${NEXTAUTH_SECRET}
      NEXTAUTH_URL: ${NEXTAUTH_URL}
      NEXT_PUBLIC_WEBAPP_URL: ${NEXT_PUBLIC_WEBAPP_URL}
      REDIS_URL: ${REDIS_URL}
      CALCOM_LICENSE_KEY: ${CALCOM_LICENSE_KEY}
    networks:
      - internal

  n8n:
    image: n8nio/n8n:latest
    restart: unless-stopped
    depends_on:
      - postgres
    environment:
      N8N_BASIC_AUTH_ACTIVE: 'true'
      N8N_BASIC_AUTH_USER: ${N8N_BASIC_AUTH_USER}
      N8N_BASIC_AUTH_PASSWORD: ${N8N_BASIC_AUTH_PASSWORD}
      N8N_HOST: ${N8N_HOST}
      N8N_PORT: ${N8N_PORT}
      N8N_PROTOCOL: ${N8N_PROTOCOL}
      WEBHOOK_URL: ${WEBHOOK_URL}
      DB_TYPE: postgresdb
      DB_POSTGRESDB_HOST: postgres
      DB_POSTGRESDB_DATABASE: ${POSTGRES_DB}
      DB_POSTGRESDB_USER: ${POSTGRES_USER}
      DB_POSTGRESDB_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - n8n_data:/home/node/.n8n
    networks:
      - internal

  webhook:
    build: ./webhook
    restart: unless-stopped
    depends_on:
      - n8n
    environment:
      WEBHOOK_PORT: ${WEBHOOK_PORT}
      WHATSAPP_APP_SECRET: ${WHATSAPP_APP_SECRET}
      WHATSAPP_VERIFY_TOKEN: ${WHATSAPP_VERIFY_TOKEN}
      INSTAGRAM_APP_SECRET: ${INSTAGRAM_APP_SECRET}
      INSTAGRAM_VERIFY_TOKEN: ${INSTAGRAM_VERIFY_TOKEN}
      N8N_WEBHOOK_URL: ${N8N_WEBHOOK_URL}
    networks:
      - internal

  nginx:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - calcom
      - n8n
      - webhook
    environment:
      DOMAIN: ${DOMAIN}
    volumes:
      - ./nginx/nginx.conf.template:/etc/nginx/nginx.conf.template:ro
      - ./nginx/certbot/conf:/etc/letsencrypt
      - ./nginx/certbot/www:/var/www/certbot
    command: >
      sh -c "envsubst '$$DOMAIN' < /etc/nginx/nginx.conf.template
             > /etc/nginx/nginx.conf && nginx -g 'daemon off;'"
    networks:
      - internal
      - external

volumes:
  postgres_data:
  n8n_data:

networks:
  internal:
    driver: bridge
  external:
    driver: bridge
```

- [ ] **Step 2: Write local dev override**

```yaml
# docker-compose.override.yml  (git-ignored — for local dev only)
services:
  nginx:
    ports:
      - "8080:80"   # use http on localhost, skip SSL
  n8n:
    ports:
      - "5678:5678"  # expose n8n directly for local access
```

- [ ] **Step 3: Commit docker-compose.yml only (override is git-ignored)**

```bash
git add docker-compose.yml
git commit -m "feat: add docker-compose stack (cal.com, n8n, postgres, redis, nginx, webhook)"
```

---

## Task 8: Local Smoke Test

No new files — verify everything starts up.

- [ ] **Step 1: Start the stack locally**

```bash
docker compose up --build
```

Expected: all services start without errors. Watch for:
- `postgres` → `database system is ready to accept connections`
- `calcom` → `ready on port 3000`
- `n8n` → `Editor is now accessible via: http://localhost:5678`
- `webhook` → `Webhook receiver running on port 3001`

- [ ] **Step 2: Verify webhook health endpoint**

```bash
curl http://localhost:8080/health
```

Expected: `{"status":"ok"}`

- [ ] **Step 3: Verify database schema was applied**

```bash
docker compose exec postgres psql -U chatbot -d chatbot_db -c "\dt"
```

Expected: tables listed — `businesses`, `staff`, `conversations`, `messages`, `whatsapp_opt_ins`, `reminders_sent`, `subscriptions`

- [ ] **Step 4: Verify Cal.com is accessible**

Open `http://localhost:8080/cal` in browser.
Expected: Cal.com setup/login screen.

- [ ] **Step 5: Verify n8n is accessible**

Open `http://localhost:5678` in browser (direct port from override).
Expected: n8n login screen. Log in with credentials from `.env`.

- [ ] **Step 6: Run webhook tests one more time in Docker context to confirm**

```bash
cd webhook && npx jest --verbose
```

Expected: all 10 tests passing.

- [ ] **Step 7: Commit smoke test confirmation (tag the working state)**

```bash
cd ..
git tag v0.1.0-infra
git push origin main --tags
```

---

## Task 9: VPS Deployment (Hetzner)

- [ ] **Step 1: Create Hetzner CX41 server**

In Hetzner Cloud Console:
- Location: Nuremberg or Helsinki (closer to Turkey)
- OS: Ubuntu 24.04
- Type: CX41 (4 vCPU, 16 GB RAM)
- Add your SSH key
- Note the server IP

- [ ] **Step 2: SSH in and install Docker**

```bash
ssh root@YOUR_SERVER_IP

apt update && apt upgrade -y
apt install -y docker.io docker-compose-plugin
systemctl enable docker
systemctl start docker
```

- [ ] **Step 3: Point your domain to the server**

In your DNS provider:
- Add A record: `yourdomain.com` → server IP
- Add A record: `www.yourdomain.com` → server IP
- Wait for DNS propagation (5–30 min)

- [ ] **Step 4: Clone repo to server and set up .env**

```bash
git clone https://github.com/youruser/appointment-chatbot.git
cd appointment-chatbot
cp .env.example .env
nano .env   # fill in all values; set DOMAIN to your actual domain
```

- [ ] **Step 5: Get SSL certificate (Certbot)**

Two containers are needed — one to serve the ACME challenge on port 80, and the certbot container to request the cert. Run them in sequence:

```bash
# 5a: Create cert directories
mkdir -p ./nginx/certbot/conf ./nginx/certbot/www

# 5b: Request the certificate using standalone mode
# (certbot binds port 80 directly — no separate nginx needed)
docker run --rm \
  -v $(pwd)/nginx/certbot/conf:/etc/letsencrypt \
  -p 80:80 \
  certbot/certbot certonly --standalone \
  -d YOUR_DOMAIN \
  --email YOUR_CERTBOT_EMAIL \
  --agree-tos --no-eff-email
```

Expected output from certbot: `Successfully received certificate.`
Certs will be in `./nginx/certbot/conf/live/YOUR_DOMAIN/`

- [ ] **Step 6: Start full stack on VPS**

```bash
docker compose up -d
docker compose ps
```

Expected: all services status `running`.

- [ ] **Step 7: Verify live deployment**

```bash
curl https://yourdomain.com/health
```

Expected: `{"status":"ok"}`

- [ ] **Step 8: Set up daily Hetzner snapshots**

In Hetzner Cloud Console → Server → Backups → Enable automatic backups.

- [ ] **Step 9: Set up nightly pg_dump to Hetzner Object Storage**

Hetzner Object Storage is S3-compatible. Use `s3cmd` to upload backups off-VPS.

```bash
# Install s3cmd
apt install -y s3cmd

# Configure s3cmd for Hetzner Object Storage
# Go to Hetzner Console → Object Storage → Create Bucket (name: chatbot-backups)
# Go to Security → API Tokens → Create S3 credentials (Access Key + Secret Key)
s3cmd --configure
# When prompted:
#   Access Key: your Hetzner S3 access key
#   Secret Key: your Hetzner S3 secret key
#   Default Region: leave blank
#   S3 Endpoint: fsn1.your-objectstorage.com  (or nbg1/hel1 depending on region)
#   DNS-style bucket+hostname: %(bucket)s.fsn1.your-objectstorage.com
#   Use HTTPS: Yes
# Test with: s3cmd ls

# Create the backup directory
mkdir -p /root/backups

# Add to crontab (create directory BEFORE editing crontab)
crontab -e
```

Add these two lines to the crontab:

```
# Nightly pg_dump at 02:00 → upload to Hetzner Object Storage → delete local file
0 2 * * * docker exec $(docker compose -f /root/appointment-chatbot/docker-compose.yml ps -q postgres) pg_dump -U chatbot chatbot_db | gzip > /root/backups/chatbot_$(date +\%Y\%m\%d).sql.gz && s3cmd put /root/backups/chatbot_$(date +\%Y\%m\%d).sql.gz s3://chatbot-backups/ && find /root/backups -mtime +7 -delete >> /root/backups/backup.log 2>&1

# Weekly n8n workflow export at 03:00 Sunday
# Export inside container → docker cp to host → upload to Object Storage
0 3 * * 0 N8N=$(docker compose -f /root/appointment-chatbot/docker-compose.yml ps -q n8n) && docker exec $N8N n8n export:workflow --all --output=/home/node/.n8n/workflows_backup.json && docker cp $N8N:/home/node/.n8n/workflows_backup.json /root/backups/workflows_$(date +\%Y\%m\%d).json && s3cmd put /root/backups/workflows_$(date +\%Y\%m\%d).json s3://chatbot-backups/ >> /root/backups/backup.log 2>&1
```

Verify first backup runs correctly:

```bash
# Trigger a manual test run
docker exec $(docker compose -f /root/appointment-chatbot/docker-compose.yml ps -q postgres) \
  pg_dump -U chatbot chatbot_db | gzip > /root/backups/test.sql.gz \
  && s3cmd put /root/backups/test.sql.gz s3://chatbot-backups/
s3cmd ls s3://chatbot-backups/
```

Expected: `test.sql.gz` listed in the bucket.

---

## Done — Plan 1 Complete

At this point you have:
- A git repository with a clean, deployable Docker Compose stack
- Cal.com, n8n, PostgreSQL (with full schema), Redis, nginx (SSL), and webhook receiver all running
- HMAC-verified webhook endpoints ready for Meta to call
- 10 passing tests covering the webhook receiver
- Daily backups configured

**Next: Plan 2 — Messaging Layer** (Meta Facebook App setup, WhatsApp Cloud API, Instagram Graph API, Embedded Signup)
