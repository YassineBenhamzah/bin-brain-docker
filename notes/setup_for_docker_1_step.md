# Project Journal: A Bin with a Brain

**Start Date:** March 27, 2026

---

## Day 1 — March 27: Project Scaffolding & Docker Setup

### What We Did

#### 1. Created the Project Structure
```
bin-with-a-brain/
├── laravel/
├── ocr/
├── nextjs/
└── docker-compose.yml
```

#### 2. Scaffolded Each Service

**Laravel (Backend API):**
```bash
cd laravel
composer create-project laravel/laravel .
```

**OCR (Node.js/Express):**
```bash
cd ocr
npm init -y
npm install express
```
Created `ocr/index.js` with a basic Express server and a `GET /health` endpoint.

**Next.js (Frontend):**
```bash
cd nextjs
npx create-next-app@latest . --typescript --tailwind --eslint --app --no-src-dir --import-alias "@/*"
```

#### 3. Created Dockerfiles

**`laravel/Dockerfile`:**
```dockerfile
FROM php:8.3-fpm
WORKDIR /var/www
RUN apt-get update && apt-get install -y libpng-dev zip unzip curl \
    && docker-php-ext-install pdo pdo_mysql gd
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer
COPY . .
RUN composer install --no-dev --optimize-autoloader
EXPOSE 8000
CMD ["php", "artisan", "serve", "--host=0.0.0.0", "--port=8000"]
```

**`ocr/Dockerfile`:**
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["node", "index.js"]
```

**`nextjs/Dockerfile`:**
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build
EXPOSE 3000
CMD ["npm", "start"]
```

#### 4. Created `docker-compose.yml`
Defined 4 services: `mysql`, `laravel`, `ocr`, `nextjs` with a shared `documents` volume between Laravel and OCR.

#### 5. Pushed to GitHub
Created 4 repositories:
- `bin-brain-api` — Laravel backend
- `bin-brain-ocr` — Node.js OCR service
- `bin-brain-frontend` — Next.js frontend
- `bin-brain-docker` — Root docker-compose.yml

### Issues Encountered & Solutions

#### Issue 1: MySQL Port 3306 Already in Use
- **Error:** `bind: Only one usage of each socket address is normally permitted.`
- **Cause:** Local MySQL was already running on port 3306.
- **Fix:** Changed host port to `3307:3306` in `docker-compose.yml`.

#### Issue 2: Express Module Not Found in OCR Container
- **Error:** `Cannot find module 'express'`
- **Cause:** Ran `npm init -y` but forgot to run `npm install express` before building Docker.
- **Fix:** Ran `npm install express` in the `ocr/` folder, then rebuilt.

#### Issue 3: Next.js Port 3000 Already in Use
- **Error:** `bind: Only one usage of each socket address is normally permitted.`
- **Cause:** Another process was using port 3000 on the host.
- **Fix:** Changed host port to `3003:3000` in `docker-compose.yml`.

#### Issue 4: Laravel Using SQLite Instead of MySQL
- **Error:** `The SQLite database configured for this application does not exist: binbrain.`
- **Cause:** Laravel 11 defaults to `DB_CONNECTION=sqlite`.
- **Fix:** Updated `laravel/.env`:
  ```env
  DB_CONNECTION=mysql
  DB_HOST=mysql
  DB_PORT=3306
  DB_DATABASE=binbrain
  DB_USERNAME=laravel
  DB_PASSWORD=secret
  QUEUE_CONNECTION=database
  ```
  Then ran:
  ```bash
  docker exec -it lfe-laravel-1 php artisan config:clear
  docker exec -it lfe-laravel-1 php artisan migrate:fresh
  ```

#### Issue 5: `node_modules` Committed to GitHub (OCR repo)
- **Cause:** `.gitignore` was not created before `git add .`
- **Fix:**
  ```bash
  git rm -r --cached node_modules
  # Created .gitignore with: node_modules/
  git add .gitignore
  git commit -m "fix: remove node_modules from repository"
  git push origin main
  ```

---

## Day 2 — March 28: Hot Reload Setup for Development

### What We Did

#### 1. Enabled Live Reload for All 3 Services
The problem: every code change required `docker compose down` + `docker compose up --build`. That's not how dev should work.

**Root cause:** Docker copies files into the container at build time (`COPY . .`). Changes on the host machine are invisible to the running container unless you mount the local folder as a volume.

**Additional Windows problem:** Windows file-save events don't trigger Linux file watchers inside Docker containers. Node's `--watch` flag silently does nothing.

#### 2. Final `docker-compose.yml` (Development Version)
```yaml
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: binbrain
      MYSQL_USER: laravel
      MYSQL_PASSWORD: secret
    volumes:
      - mysql_data:/var/lib/mysql
    ports:
      - "3307:3306"

  laravel:
    build: ./laravel
    ports:
      - "8000:8000"
    environment:
      DB_HOST: mysql
      DB_DATABASE: binbrain
      DB_USERNAME: laravel
      DB_PASSWORD: secret
    volumes:
      - documents:/var/documents
      - ./laravel:/var/www
    depends_on:
      - mysql

  ocr:
    build: ./ocr
    ports:
      - "3001:3000"
    volumes:
      - documents:/var/documents
      - ./ocr:/app
      - /app/node_modules
    command: npx -y nodemon -L index.js

  nextjs:
    build: ./nextjs
    ports:
      - "3003:3000"
    volumes:
      - ./nextjs:/app
      - /app/node_modules
      - /app/.next
    command: npm run dev
    depends_on:
      - laravel

volumes:
  mysql_data:
  documents:
```

#### How Each Service Auto-Reloads

| Service | Method | Why it works |
|---------|--------|--------------|
| **Laravel** | PHP re-reads files on every HTTP request | No watcher needed — just save and refresh |
| **OCR** | `npx -y nodemon -L index.js` | `-L` = legacy polling, bypasses Windows file event issue |
| **Next.js** | `npm run dev` | Next.js dev server has built-in hot module replacement |

#### Why `/app/node_modules` is a Separate Volume
When you mount `./ocr:/app`, your Windows `node_modules` (which may not exist or may have wrong binaries) overwrites the Linux `node_modules` inside the container. The anonymous volume `/app/node_modules` keeps the container's own `node_modules` intact.

### Issues Encountered & Solutions

#### Issue 6: OCR Container Not Picking Up Code Changes
- **Error:** New routes added to `index.js` not appearing at `localhost:3001`
- **Cause:** `node --watch` doesn't work on Windows-to-Linux Docker volume mounts (no file events)
- **Fix:** Switched to `npx -y nodemon -L index.js` which uses polling instead

#### Issue 7: `sessions` Table Not Found After Rebuild
- **Error:** `Table 'binbrain.sessions' doesn't exist`
- **Cause:** `docker compose down` + rebuild wipes the container state. Laravel's session/cache tables were never migrated into the new MySQL database.
- **Fix:** Ran `docker exec -it lfe-laravel-1 php artisan migrate:fresh` after every fresh rebuild.

---

## Ports Reference

| Service | Host URL | Internal Docker URL |
|---------|----------|---------------------|
| Next.js | `http://localhost:3003` | `http://nextjs:3000` |
| Laravel | `http://localhost:8000` | `http://laravel:8000` |
| OCR | `http://localhost:3001` | `http://ocr:3000` |
| MySQL | `localhost:3307` | `mysql:3306` |

---

## GitHub Repositories

| Repo | Contents |
|------|----------|
| `bin-brain-api` | Laravel backend API |
| `bin-brain-ocr` | Node.js Express + Tesseract OCR service |
| `bin-brain-frontend` | Next.js + TailwindCSS frontend |
| `bin-brain-docker` | Root `docker-compose.yml` |

---

## Current Status
- ✅ Phase 1: Docker & Service Skeletons — **COMPLETE**
- ⬜ Phase 2: Laravel Upload API & Database — **NEXT**
- ⬜ Phase 3: OCR Service (Tesseract.js)
- ⬜ Phase 4: Laravel Queue Worker
- ⬜ Phase 5: Next.js Frontend UI
- ⬜ Phase 6: Hardening & Polish
