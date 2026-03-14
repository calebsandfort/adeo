# Setup Project

This command performs all the steps necessary to get the project up and running after cloning.

## Prerequisites

- **Docker group membership**: The user must be in the `docker` group. If not:
  ```bash
  sudo usermod -aG docker $USER
  ```
  Then log out and back in (or run `newgrp docker`).

## Steps

1. **Install backend dependencies**
   ```bash
   cd backend && uv sync
   ```

2. **Install frontend dependencies**
   ```bash
   cd frontend && pnpm install
   ```

3. **Copy environment files**
   ```bash
   cp frontend/.env.example frontend/.env
   cp backend/.env.example backend/.env
   ```

4. **Change database db name**
   Change the db name from 'boilerplate' to the name of the project.
   Update the database name in all places it is referenced as 'boilerplate':
   - `frontend/.env` (POSTGRES_DB and DATABASE_URL)
   - `backend/.env` (DATABASE_URL)
   - `backend/src/config.py` (default database_url)
   - `docker-compose.yml` (all POSTGRES_DB defaults)
   - `dev.sh` (SESSION name)
   - `frontend/.env.example` and `backend/.env.example`

5. **Remove deprecated middleware.ts**
   Next.js 16+ uses `proxy.ts` instead of `middleware.ts`. If both exist, delete the middleware file:
   ```bash
   rm frontend/src/middleware.ts
   ```

6. **Start Docker services (database)**
   ```bash
   docker compose up -d db
   ```
   Wait for the database to be healthy before proceeding.

7. **Start backend server**
   ```bash
   cd backend && uv run uvicorn src.main:app --host 0.0.0.0 --port 8000 --reload
   ```

8. **Start frontend server**
   ```bash
   cd frontend && pnpm dev
   ```

## Verification

After setup, verify the following endpoints:
- Frontend: http://localhost:3000 (expect 200)
- Backend Health: http://localhost:8000/health (expect 200, `{"status":"ok"}`)
