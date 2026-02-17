# Contact List Application — Project Plan

> **Repository**: `contact-list-next-nest`  
> **Stack**: TypeScript (strict) · NestJS + Fastify · Next.js (App Router) · Tailwind CSS · Docker · GitHub Actions

## Project Structure

```
contact-list-next-nest/
├── backend/                    # NestJS + Fastify API
├── frontend/                   # Next.js App Router
├── .github/workflows/
│   └── ci-cd.yml               # CI/CD pipeline
├── docker-compose.yml          # Multi-service orchestration
├── package.json                # Root: concurrently dev scripts
├── deploy.sh                   # SSH deployment helper
├── PROJECT_PLAN.md             # This file
└── README.md
```

---

## PHASE 1: Backend (NestJS + Fastify)

### 1. Initialize NestJS in `backend/`

```bash
# From repo root
npx -y @nestjs/cli new backend --package-manager npm --skip-git --strict
```

- After scaffolding, delete the default test file (`app.controller.spec.ts`) and the default controller/service (`app.controller.ts`, `app.service.ts`) — keep only `app.module.ts` and `main.ts`.
- Verify `tsconfig.json` has `"strict": true`.

> **⚠ Fix from original plan**: The original plan used `cd backend && npx @nestjs/cli new .` which does not work reliably — the NestJS CLI expects a project name, not `.`. Always provide a folder name and run from the repo root.

### 2. Switch to Fastify adapter

```bash
cd backend
npm uninstall @nestjs/platform-express
npm install @nestjs/platform-fastify
npm install class-validator class-transformer
```

- Update `src/main.ts`:
  - Import `FastifyAdapter` and `NestFastifyApplication` from `@nestjs/platform-fastify`.
  - Create the app with `NestFactory.create<NestFastifyApplication>(AppModule, new FastifyAdapter())`.
  - Enable CORS: `app.enableCors({ origin: 'http://localhost:3000' })`.
  - Add `ValidationPipe` globally: `app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }))`.
  - Listen on port `3001` and bind to `0.0.0.0` (needed for Docker): `await app.listen(3001, '0.0.0.0')`.

> **⚠ Fix from original plan**: The original listed `@nestjs/common` as a dependency to install — it's already included with a NestJS scaffold. Also, `@nestjs/platform-express` must be explicitly **uninstalled** to avoid conflicts.

### 3. Create Contact module, controller, service

```bash
cd backend
npx nest g module contact --no-spec
npx nest g controller contact --no-spec
npx nest g service contact --no-spec
```

- **Contact entity** — create `src/contact/entities/contact.entity.ts`:

```typescript
export interface Contact {
  id: string;        // UUID
  name: string;
  email: string;
  phone: string;
  createdAt: Date;
}
```

- **Create DTO** — `src/contact/dto/create-contact.dto.ts`:

```typescript
import { IsString, IsEmail, IsNotEmpty } from 'class-validator';

export class CreateContactDto {
  @IsString()
  @IsNotEmpty()
  name: string;

  @IsEmail()
  email: string;

  @IsString()
  @IsNotEmpty()
  phone: string;
}
```

- **Update DTO** — `src/contact/dto/update-contact.dto.ts`:

```typescript
import { PartialType } from '@nestjs/mapped-types';
import { CreateContactDto } from './create-contact.dto';

export class UpdateContactDto extends PartialType(CreateContactDto) {}
```

> **📝 Added detail**: Install `@nestjs/mapped-types` for `PartialType`:
> ```bash
> npm install @nestjs/mapped-types
> ```

### 4. Implement CRUD

**`contact.service.ts`** — In-memory array storage:

| Method | Description |
|---|---|
| `findAll()` | Return all contacts |
| `findOne(id)` | Find by ID, throw `NotFoundException` if missing |
| `create(dto)` | Generate UUID (`crypto.randomUUID()`), set `createdAt`, push to array |
| `update(id, dto)` | Find contact, merge fields, return updated |
| `remove(id)` | Find contact, splice from array, return removed |

**`contact.controller.ts`** — REST endpoints:

| Method | Route | Status Code | Description |
|---|---|---|---|
| `GET` | `/contacts` | 200 | List all contacts |
| `GET` | `/contacts/:id` | 200 | Get single contact |
| `POST` | `/contacts` | 201 | Create new contact |
| `PATCH` | `/contacts/:id` | 200 | Partial update |
| `DELETE` | `/contacts/:id` | 200 | Delete contact |

- Use `@Body()`, `@Param()` decorators.
- `NotFoundException` is thrown by the service for invalid IDs, NestJS auto-maps it to HTTP 404.

### 5. Test backend endpoints

```bash
cd backend && npm run start:dev
```

Test with curl:

```bash
# Create
curl -X POST http://localhost:3001/contacts -H "Content-Type: application/json" -d '{"name":"John","email":"john@example.com","phone":"123456"}'

# List
curl http://localhost:3001/contacts

# Get by ID (use ID from create response)
curl http://localhost:3001/contacts/<id>

# Update
curl -X PATCH http://localhost:3001/contacts/<id> -H "Content-Type: application/json" -d '{"name":"Jane"}'

# Delete
curl -X DELETE http://localhost:3001/contacts/<id>
```

- ✅ Verify all 5 endpoints return correct status codes.
- ✅ Verify validation rejects missing/invalid fields (e.g., bad email).

---

## PHASE 2: Frontend (Next.js App Router)

### 6. Initialize Next.js in `frontend/`

```bash
# From repo root
npx -y create-next-app@latest frontend --typescript --tailwind --eslint --app --src-dir --import-alias "@/*" --no-turbopack --use-npm
```

> **⚠ Fix from original plan**: The original used `.` as the directory (same issue as NestJS). Using `frontend` as the folder name is more reliable. Added `--no-turbopack` and `--use-npm` for consistency. Also added `--use-npm` to match the backend package manager.  
> **📝 Note**: If `create-next-app` prompts for interactive choices despite flags, just accept the defaults. The flags above should work with Next.js 14+.

### 7. Create API utility

Create `frontend/src/lib/api.ts` using **native `fetch`** (no axios):

```typescript
const API_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:3001';

export const api = {
  getContacts: () => fetch(`${API_URL}/contacts`).then(res => res.json()),
  getContact: (id: string) => fetch(`${API_URL}/contacts/${id}`).then(res => res.json()),
  createContact: (data: CreateContactInput) =>
    fetch(`${API_URL}/contacts`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
    }).then(res => res.json()),
  updateContact: (id: string, data: Partial<CreateContactInput>) =>
    fetch(`${API_URL}/contacts/${id}`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
    }).then(res => res.json()),
  deleteContact: (id: string) =>
    fetch(`${API_URL}/contacts/${id}`, { method: 'DELETE' }).then(res => res.json()),
};
```

- Define `CreateContactInput` type in `src/lib/types.ts`.
- All functions should include error handling (check `res.ok`, throw on failure).

> **⚠ Fix from original plan**: The original plan mentioned axios, but the requirements say "no external UI libraries." Using native `fetch` instead.

### 8. Build pages with App Router

| Route | File | Type | Description |
|---|---|---|---|
| `/contacts` | `src/app/contacts/page.tsx` | Server Component | List all contacts, link to view/create |
| `/contacts/new` | `src/app/contacts/new/page.tsx` | Client Component | Create contact form |
| `/contacts/[id]` | `src/app/contacts/[id]/page.tsx` | Server Component | View single contact, edit/delete buttons |
| `/contacts/[id]/edit` | `src/app/contacts/[id]/edit/page.tsx` | Client Component | Edit contact form |

Additional files:

- `src/app/layout.tsx` — Root layout with navigation header (links: Home, Contacts, New Contact).
- `src/app/page.tsx` — Landing page that redirects to `/contacts`.
- `src/components/ContactForm.tsx` — Reusable form component (Client Component, `'use client'`), used by both new and edit pages.
- `src/components/Navbar.tsx` — Navigation header component (Client Component for active link highlighting).

> **📝 Added detail**: Server Components can fetch data directly. Client Components (`'use client'`) are needed for forms and interactive elements. The original plan didn't clarify this distinction, which is critical for App Router.

### 9. Implement CRUD operations

**Create (`/contacts/new`)**:
- Controlled form with `useState` for fields.
- On submit, call `api.createContact()`, then `router.push('/contacts')`.
- Show loading spinner during submission.
- Show error message on failure.

**Read list (`/contacts`)**:
- Server component that fetches contacts with `api.getContacts()`.
- Render a table/card list.
- Show "No contacts yet" empty state.
- Each row links to `/contacts/[id]`.

**Read single (`/contacts/[id]`)**:
- Server component that fetches with `api.getContact(id)`.
- Display all fields.
- Buttons: "Edit" (links to edit page), "Delete" (client-side action).
- Delete button should be a small Client Component with confirmation.

**Update (`/contacts/[id]/edit`)**:
- Client component, pre-fills form with existing data from API.
- On submit, call `api.updateContact()`, then `router.push('/contacts/${id}')`.

**Delete**:
- Triggered from the single contact view page.
- Confirm before deleting (`window.confirm()`).
- Call `api.deleteContact()`, then `router.push('/contacts')`.

**Common patterns**:
- Use `router.refresh()` or `revalidatePath` after mutations to refresh server-fetched data.
- Loading states with a simple spinner or "Loading..." text.
- Error handling with try/catch and user-visible error messages.

### 10. Environment variables

Create `frontend/.env.local`:

```
NEXT_PUBLIC_API_URL=http://localhost:3001
```

> **⚠ Important note**: `NEXT_PUBLIC_` variables are **baked into the JS bundle at build time**. This has implications for Docker (see Phase 4 notes).

---

## PHASE 3: Local Development Scripts

### 11. Root `package.json` and scripts

```bash
# From repo root
npm init -y
npm install -D concurrently
```

Update root `package.json` scripts:

```json
{
  "scripts": {
    "dev": "concurrently \"npm run dev:backend\" \"npm run dev:frontend\"",
    "dev:backend": "cd backend && npm run start:dev",
    "dev:frontend": "cd frontend && npm run dev",
    "install:all": "cd backend && npm install && cd ../frontend && npm install"
  }
}
```

> **⚠ Fix from original plan**: The original plan didn't mention creating the root `package.json` first with `npm init`. Without this step, `npm install -D concurrently` would fail.

---

## PHASE 4: Docker Setup

### 12. Backend Dockerfile (`backend/Dockerfile`)

```dockerfile
# -- Build stage --
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# -- Production stage --
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/package*.json ./
RUN npm ci --omit=dev
COPY --from=builder /app/dist ./dist
EXPOSE 3001
CMD ["node", "dist/main"]
```

- Add `backend/.dockerignore`: `node_modules`, `dist`, `.git`.

### 13. Frontend Dockerfile (`frontend/Dockerfile`)

```dockerfile
# -- Build stage --
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
# Build-time env var — for Docker, API calls from the browser go to localhost:3001
ARG NEXT_PUBLIC_API_URL=http://localhost:3001
ENV NEXT_PUBLIC_API_URL=$NEXT_PUBLIC_API_URL
RUN npm run build

# -- Production stage --
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/package*.json ./
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/public ./public
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
ENV NODE_ENV=production
CMD ["npm", "start"]
```

- Add `frontend/.dockerignore`: `node_modules`, `.next`, `.git`.

> **⚠ Fix from original plan**: `NEXT_PUBLIC_` env vars must be set **at build time** (they get inlined into the client JS bundle by Next.js). Setting them in `docker-compose.yml` `environment` block at runtime has **no effect**. The fix is to use `ARG` in the Dockerfile so the value is baked in during `docker build`.  
> For local Docker usage, the browser needs to reach `http://localhost:3001` (not `http://backend:3001`), because the browser runs on the host machine, not inside Docker's network.

### 14. `docker-compose.yml` (root)

```yaml
version: '3.8'

services:
  backend:
    build: ./backend
    ports:
      - "3001:3001"
    environment:
      - NODE_ENV=production

  frontend:
    build:
      context: ./frontend
      args:
        NEXT_PUBLIC_API_URL: http://localhost:3001
    ports:
      - "3000:3000"
    depends_on:
      - backend
    environment:
      - NODE_ENV=production
```

> **⚠ Fix from original plan**: The original set `NEXT_PUBLIC_API_URL=http://backend:3001` in the `environment` block. This is wrong for two reasons:
> 1. `NEXT_PUBLIC_` vars are build-time only — runtime `environment` has no effect.
> 2. The browser (running on host) cannot resolve `backend` — it needs `localhost:3001`.
>
> The fix uses `build.args` to pass the env var at build time with the correct host-accessible URL.

---

## PHASE 5: GitHub Actions CI/CD

### 15. `.github/workflows/ci-cd.yml`

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main]

env:
  DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}

jobs:
  build-push-backend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - name: Build and push backend
        uses: docker/build-push-action@v5
        with:
          context: ./backend
          push: true
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/contact-list-backend:latest

  build-push-frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - name: Build and push frontend
        uses: docker/build-push-action@v5
        with:
          context: ./frontend
          push: true
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/contact-list-frontend:latest
          build-args: |
            NEXT_PUBLIC_API_URL=${{ secrets.PRODUCTION_API_URL }}

  deploy:
    runs-on: ubuntu-latest
    needs: [build-push-backend, build-push-frontend]
    steps:
      - uses: actions/checkout@v4
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.DEPLOY_HOST }}
          username: ${{ secrets.DEPLOY_USER }}
          key: ${{ secrets.DEPLOY_SSH_KEY }}
          script: |
            cd ${{ secrets.DEPLOY_PATH }}
            docker compose pull
            docker compose up -d --remove-orphans
```

**Required GitHub Secrets** (must be configured in repo Settings → Secrets):

| Secret | Description |
|---|---|
| `DOCKERHUB_USERNAME` | Docker Hub username |
| `DOCKERHUB_TOKEN` | Docker Hub access token |
| `DEPLOY_HOST` | Server IP or hostname |
| `DEPLOY_USER` | SSH username on server |
| `DEPLOY_SSH_KEY` | Private SSH key for deployment |
| `DEPLOY_PATH` | Path on server where `docker-compose.yml` lives |
| `PRODUCTION_API_URL` | Production API URL (e.g., `https://api.yourdomain.com`) |

> **⚠ Fix from original plan**: The original didn't specify which GitHub secrets are needed or how to configure them. Also, the frontend build needs `PRODUCTION_API_URL` passed as a build arg, not a runtime env var.

### 16. Deployment script — `deploy.sh` (root)

```bash
#!/bin/bash
set -e

echo "Pulling latest images..."
docker compose pull

echo "Stopping old containers..."
docker compose down

echo "Starting new containers..."
docker compose up -d --remove-orphans

echo "Cleaning up old images..."
docker image prune -f

echo "Deployment complete!"
```

> **📝 Added**: `set -e` for fail-fast behavior, image pruning step, and status messages.

---

## PHASE 6: README.md and Polish

### 17. `README.md` sections

The README should include:

1. **Project Title & Description** — What the app does.
2. **Tech Stack** — NestJS, Fastify, Next.js, Tailwind, Docker.
3. **Prerequisites** — Node.js 20+, npm, Docker (optional).
4. **Local Development Setup**:
   ```bash
   npm run install:all
   npm run dev
   ```
   - Backend runs at `http://localhost:3001`
   - Frontend runs at `http://localhost:3000`
5. **Docker Setup**:
   ```bash
   docker compose up --build
   ```
6. **API Endpoints**:
   | Method | Endpoint | Description |
   |---|---|---|
   | GET | /contacts | List all |
   | GET | /contacts/:id | Get one |
   | POST | /contacts | Create |
   | PATCH | /contacts/:id | Update |
   | DELETE | /contacts/:id | Delete |
7. **Environment Variables** — Table of all env vars per service.
8. **Deployment** — Docker Hub + SSH deployment via GitHub Actions.
9. **Project Structure** — Folder tree diagram.

### 18. Final testing checklist

- [ ] `npm run dev` starts both backend and frontend concurrently
- [ ] All 5 CRUD endpoints return correct responses (test via curl)
- [ ] Frontend can create, list, view, edit, and delete contacts
- [ ] Form validation works (empty fields, invalid email)
- [ ] Error states display properly (API down, 404)
- [ ] `docker compose up --build` starts both services
- [ ] Frontend in Docker can communicate with backend
- [ ] Push to `main` triggers GitHub Actions workflow
- [ ] Docker images are built and pushed to Docker Hub
- [ ] SSH deploy step runs successfully (requires server setup)

---

## Git Commit Strategy

| After Phase | Commit Message |
|---|---|
| Phase 1 | `feat(backend): add NestJS + Fastify contact CRUD API` |
| Phase 2 | `feat(frontend): add Next.js contact management UI` |
| Phase 3 | `chore: add root dev scripts with concurrently` |
| Phase 4 | `chore(docker): add Dockerfiles and docker-compose` |
| Phase 5 | `ci: add GitHub Actions CI/CD pipeline` |
| Phase 6 | `docs: add README with setup and API documentation` |

---

## Requirements Checklist

- [x] TypeScript everywhere, strict mode
- [x] NestJS + Fastify (no Express)
- [x] Next.js App Router (no React-only components)
- [x] Tailwind CSS + native HTML (no external UI libraries)
- [x] Native `fetch` (no axios)
- [x] Error handling and loading states
- [x] Proper HTTP status codes
- [x] Docker multi-stage builds
- [x] GitHub Actions CI/CD
- [x] Git commits per phase

---

## Summary of Fixes Applied to Original Plan

| # | Issue | Fix |
|---|---|---|
| 1 | `npx @nestjs/cli new .` doesn't work reliably | Use `npx @nestjs/cli new backend` from root |
| 2 | `@nestjs/common` listed as install dependency | Already included in NestJS scaffold — removed |
| 3 | `@nestjs/platform-express` not uninstalled | Added explicit `npm uninstall @nestjs/platform-express` |
| 4 | No mention of `@nestjs/mapped-types` | Added for `PartialType` in UpdateDTO |
| 5 | `create-next-app` with `.` as dir | Use `frontend` folder name from root |
| 6 | Axios usage mentioned | Replaced with native `fetch` per requirements |
| 7 | No client/server component distinction | Specified which pages are Server vs. Client components |
| 8 | Root `package.json` not initialized | Added `npm init -y` step |
| 9 | `NEXT_PUBLIC_` env var set at Docker runtime | Moved to build-time `ARG` — these vars are baked at build |
| 10 | `NEXT_PUBLIC_API_URL=http://backend:3001` | Changed to `http://localhost:3001` — browser can't resolve Docker names |
| 11 | GitHub secrets not specified | Added full secrets table and configuration notes |
| 12 | Missing `ValidationPipe` placement detail | Specified exact setup in `main.ts` |
| 13 | Missing `.dockerignore` files | Added for both services |
| 14 | `deploy.sh` missing `set -e` and cleanup | Added error handling and image pruning |
| 15 | Missing Fastify `listen` bind to `0.0.0.0` | Added — required for Docker networking |
| 16 | Reusable form component not mentioned | Added `ContactForm.tsx` shared component |
