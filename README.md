# The Ultimate PERN Stack Project Setup Guide

**Last Updated:** March 2026  
**Architecture:** Monorepo · TypeScript · Express · Vite · Docker · Supabase

A clean, production-ready PERN skeleton. Every file is provided. Every decision is explained. Follow it top to bottom and you'll have a running app in under 20 minutes.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Phase 1 — Monorepo Foundation](#phase-1--monorepo-foundation)
3. [Phase 2 — Backend (Express + TypeScript API)](#phase-2--backend-express--typescript-api)
4. [Phase 3 — Frontend (Vite + React + TypeScript)](#phase-3--frontend-vite--react--typescript)
5. [Phase 4 — Orchestration](#phase-4--orchestration)
6. [Phase 5 — Database (Docker + PostgreSQL)](#phase-5--database-docker--postgresql)
7. [Phase 6 — Production Database (Supabase)](#phase-6--production-database-supabase)
8. [Phase 7 — AI Onboarding (Claude Code)](#phase-7--ai-onboarding-claude-code)
9. [Running the App](#running-the-app)
10. [What to Build Next](#what-to-build-next)
11. [Toolkit Reference](#toolkit-reference)
12. [Final Project Structure](#final-project-structure)

---

## Prerequisites

| Tool | Version | Why |
|---|---|---|
| **Node.js** | v22 LTS (or v24 LTS) | Even-numbered releases = Long Term Support. |
| **pnpm** | v9+ | Faster, disk-efficient package manager. We use `pnpx` for one-off execution. |
| **Git** | Any recent | Version control. |
| **Docker Desktop** | Latest | Runs PostgreSQL in a container — no local install needed. |

> **Optional:** [Claude Code](https://docs.claude.com) (`pnpm install -g @anthropic-ai/claude-code`) — an AI coding assistant that lives in your terminal. Phase 7 covers setup.

Install `pnpm` if you don't have it:

```bash
npm install -g pnpm
```

Verify your environment before continuing:

```bash
node -v    # Should print v22.x.x or v24.x.x
pnpm -v    # Should print 9.x+
git -v     # Any version
docker -v  # Any version
```

---

## Phase 1 — Monorepo Foundation

A monorepo means one Git repository holds both client and server. The root `package.json` exists solely for orchestration — it never holds application code.

### 1.1 Initialize the project

```bash
mkdir my-pern-app
cd my-pern-app
git init
pnpm init
```

### 1.2 Create `.gitignore` (do this immediately)

This is the single most important file you create first. One leaked `.env` with a database password or API key can compromise your entire project.

```
# Dependencies
node_modules/

# Environment secrets
.env
.env.local
.env.*.local

# Build output
dist/
build/

# TypeScript compiled output
*.js.map
*.d.ts.map

# OS / Editor junk
.DS_Store
Thumbs.db
.vscode/
.idea/

# Logs
*.log
pnpm-debug.log*

# pnpm
.pnpm-store/
```

> **Why no `.env.example` in `.gitignore`?** Because `.env.example` is meant to be committed — it documents which environment variables the project needs, without containing actual secrets.

### 1.3 Install root orchestration tools

```bash
pnpm add -D concurrently
```

**What this does:** `concurrently` runs multiple shell commands in parallel. We use it to start the client and server with a single `pnpm dev` from the root.

---

## Phase 2 — Backend (Express + TypeScript API)

We build a secure, structured API using Express and TypeScript with a PostgreSQL client.

### Why TypeScript on the backend?

TypeScript gives you compile-time type safety across your entire codebase. In a PERN app, this is especially valuable because you can share types between your server and client — a query result type defined once can be used in both your Express controller and your React component.

### 2.1 Setup and dependencies

```bash
mkdir server && cd server
pnpm init
```

Install production dependencies:

```bash
pnpm add express pg dotenv cors helmet morgan zod
```

Install dev dependencies:

```bash
pnpm add -D typescript ts-node nodemon @types/express @types/pg @types/cors @types/morgan @types/node eslint prettier eslint-config-prettier @typescript-eslint/eslint-plugin @typescript-eslint/parser
```

| Package | Purpose |
|---|---|
| `express` | Web framework. Handles routing, middleware, HTTP. |
| `pg` | The official PostgreSQL client for Node.js. Executes raw SQL queries and manages a connection pool. |
| `dotenv` | Loads variables from `.env` into `process.env` so secrets never touch your code. |
| `cors` | Controls which origins (domains) can call your API. |
| `helmet` | Sets secure HTTP headers automatically (prevents clickjacking, XSS sniffing, etc.). |
| `morgan` | Logs every HTTP request to the console (`GET /api/users 200 12ms`). Invaluable for debugging. |
| `zod` | Schema validation library. Validates and parses request bodies at runtime with full TypeScript inference. |
| `typescript` | TypeScript compiler. |
| `ts-node` | Runs TypeScript files directly without a separate compile step. Used by nodemon in development. |
| `nodemon` | Watches your files and restarts the server on save. Dev only. |
| `@types/*` | Type definitions for third-party packages that don't ship their own. |
| `eslint` + `prettier` | Code linting and formatting. Catches bugs and enforces consistent style. |
| `@typescript-eslint/*` | ESLint plugins that understand TypeScript syntax. |

### 2.2 TypeScript configuration — `server/tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "CommonJS",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

> **Why `CommonJS` on the server?** The `pg` package and most Node.js tooling in the ecosystem has historically assumed CommonJS. Using it server-side avoids edge cases with dynamic imports and `__dirname`. The frontend (Vite) uses ES Modules, which is correct for the browser.

### 2.3 ESLint + Prettier configuration

Create `server/.eslintrc.json`:

```json
{
  "parser": "@typescript-eslint/parser",
  "plugins": ["@typescript-eslint"],
  "extends": [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "prettier"
  ],
  "env": {
    "node": true,
    "es2022": true
  },
  "rules": {
    "@typescript-eslint/no-unused-vars": ["error", { "argsIgnorePattern": "^_" }],
    "@typescript-eslint/explicit-function-return-type": "off",
    "no-console": "off"
  }
}
```

Create `server/.prettierrc`:

```json
{
  "semi": true,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "es5",
  "printWidth": 100
}
```

### 2.4 Update `server/package.json` scripts

```json
{
  "name": "server",
  "version": "1.0.0",
  "scripts": {
    "dev": "nodemon --watch src --ext ts --exec ts-node src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js",
    "lint": "eslint src --ext .ts",
    "format": "prettier --write src/**/*.ts"
  }
}
```

### 2.5 Folder structure

Create this inside `/server/src`:

```
server/
├── src/
│   ├── config/
│   │   └── db.ts              # Database connection pool
│   ├── controllers/
│   │   └── exampleController.ts
│   ├── middleware/
│   │   └── errorMiddleware.ts  # Centralized error handling
│   ├── routes/
│   │   └── exampleRoutes.ts
│   ├── schemas/
│   │   └── exampleSchema.ts    # Zod validation schemas
│   ├── types/
│   │   └── index.ts            # Shared TypeScript types
│   └── index.ts                # Entry point
├── .env
├── .env.example
├── .eslintrc.json
├── .prettierrc
├── package.json
└── tsconfig.json
```

> **Why no `models/` folder?** In a SQL database, your schema is defined in SQL migrations, not in application code (unlike Mongoose schemas in MongoDB). Instead, we define TypeScript `types/` that mirror the shape of our database rows. This keeps the source of truth in the database where it belongs.

Create all the folders now:

```bash
mkdir -p src/config src/controllers src/middleware src/routes src/schemas src/types
```

### 2.6 Environment files

Create `server/.env`:

```
NODE_ENV=development
PORT=5000
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/my_pern_app
```

Create `server/.env.example` (commit this to Git):

```
NODE_ENV=development
PORT=5000
DATABASE_URL=postgresql://<user>:<password>@<host>:<port>/<database>
```

> **Tip:** When you move to Supabase in production, you replace `DATABASE_URL` in `.env` with your Supabase connection string. The code doesn't change.

### 2.7 Database connection pool — `server/src/config/db.ts`

```typescript
import { Pool } from 'pg';
import dotenv from 'dotenv';

dotenv.config();

if (!process.env.DATABASE_URL) {
  console.error('❌ DATABASE_URL is not set in environment variables.');
  process.exit(1);
}

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  // In production (Supabase), SSL is required
  ssl: process.env.NODE_ENV === 'production' ? { rejectUnauthorized: false } : false,
});

pool.on('connect', () => {
  console.log('✅ Connected to PostgreSQL');
});

pool.on('error', (err) => {
  console.error('❌ Unexpected PostgreSQL pool error:', err.message);
  process.exit(1);
});

export default pool;
```

**What's happening:** Rather than one persistent connection, `pg.Pool` maintains a pool of reusable connections. This is critical for performance under load — each incoming request can grab a free connection from the pool instead of waiting to open a new one. The pool is exported as a singleton so the same pool is reused across all your controllers.

### 2.8 TypeScript types — `server/src/types/index.ts`

Define types that mirror your database rows here. They can also be copied into the client for end-to-end type safety.

```typescript
export interface Example {
  id: number;
  title: string;
  description: string | null;
  created_at: Date;
  updated_at: Date;
}

export interface ApiResponse<T> {
  success: boolean;
  data?: T;
  message?: string;
  errors?: string[];
}
```

### 2.9 Zod schema — `server/src/schemas/exampleSchema.ts`

Zod validates incoming request bodies at runtime and infers TypeScript types from the schema automatically. You define the shape once and get both validation and types for free.

```typescript
import { z } from 'zod';

export const createExampleSchema = z.object({
  title: z.string().min(1, 'Title is required').max(255, 'Title must be 255 characters or fewer'),
  description: z.string().max(1000).optional(),
});

export const updateExampleSchema = createExampleSchema.partial();

// Infer TypeScript types directly from the schemas
export type CreateExampleInput = z.infer<typeof createExampleSchema>;
export type UpdateExampleInput = z.infer<typeof updateExampleSchema>;
```

### 2.10 Error middleware — `server/src/middleware/errorMiddleware.ts`

```typescript
import { Request, Response, NextFunction } from 'express';

export class AppError extends Error {
  statusCode: number;

  constructor(message: string, statusCode: number) {
    super(message);
    this.statusCode = statusCode;
    // Restore prototype chain (required when extending built-in classes in TypeScript)
    Object.setPrototypeOf(this, AppError.prototype);
  }
}

// Catches requests to routes that don't exist
export const notFound = (req: Request, res: Response, next: NextFunction): void => {
  next(new AppError(`Not Found - ${req.originalUrl}`, 404));
};

// Catches all errors thrown anywhere in the app
export const errorHandler = (
  err: AppError | Error,
  _req: Request,
  res: Response,
  _next: NextFunction
): void => {
  const statusCode = err instanceof AppError ? err.statusCode : 500;
  const message = err.message || 'Internal Server Error';

  res.status(statusCode).json({
    success: false,
    message,
    stack: process.env.NODE_ENV === 'production' ? undefined : err.stack,
  });
};
```

**Why a custom `AppError` class?** It lets you attach a `statusCode` to any error you throw in a controller. The central error handler reads it and sends the correct HTTP status. Without this, every error would be a generic 500.

### 2.11 Example controller — `server/src/controllers/exampleController.ts`

```typescript
import { Request, Response, NextFunction } from 'express';
import pool from '../config/db.js';
import { AppError } from '../middleware/errorMiddleware.js';
import { createExampleSchema, updateExampleSchema } from '../schemas/exampleSchema.js';
import type { Example, ApiResponse } from '../types/index.js';

// GET /api/examples
export const getExamples = async (
  _req: Request,
  res: Response<ApiResponse<Example[]>>,
  next: NextFunction
): Promise<void> => {
  try {
    const result = await pool.query<Example>(
      'SELECT * FROM examples ORDER BY created_at DESC'
    );
    res.json({ success: true, data: result.rows });
  } catch (err) {
    next(err);
  }
};

// GET /api/examples/:id
export const getExampleById = async (
  req: Request,
  res: Response<ApiResponse<Example>>,
  next: NextFunction
): Promise<void> => {
  try {
    const { id } = req.params;
    const result = await pool.query<Example>('SELECT * FROM examples WHERE id = $1', [id]);

    if (result.rows.length === 0) {
      throw new AppError('Example not found', 404);
    }

    res.json({ success: true, data: result.rows[0] });
  } catch (err) {
    next(err);
  }
};

// POST /api/examples
export const createExample = async (
  req: Request,
  res: Response<ApiResponse<Example>>,
  next: NextFunction
): Promise<void> => {
  try {
    const parsed = createExampleSchema.safeParse(req.body);

    if (!parsed.success) {
      res.status(400).json({
        success: false,
        message: 'Validation failed',
        errors: parsed.error.errors.map((e) => e.message),
      });
      return;
    }

    const { title, description } = parsed.data;
    const result = await pool.query<Example>(
      'INSERT INTO examples (title, description) VALUES ($1, $2) RETURNING *',
      [title, description ?? null]
    );

    res.status(201).json({ success: true, data: result.rows[0] });
  } catch (err) {
    next(err);
  }
};

// PUT /api/examples/:id
export const updateExample = async (
  req: Request,
  res: Response<ApiResponse<Example>>,
  next: NextFunction
): Promise<void> => {
  try {
    const { id } = req.params;
    const parsed = updateExampleSchema.safeParse(req.body);

    if (!parsed.success) {
      res.status(400).json({
        success: false,
        message: 'Validation failed',
        errors: parsed.error.errors.map((e) => e.message),
      });
      return;
    }

    const { title, description } = parsed.data;
    const result = await pool.query<Example>(
      `UPDATE examples
       SET title = COALESCE($1, title),
           description = COALESCE($2, description),
           updated_at = NOW()
       WHERE id = $3
       RETURNING *`,
      [title ?? null, description ?? null, id]
    );

    if (result.rows.length === 0) {
      throw new AppError('Example not found', 404);
    }

    res.json({ success: true, data: result.rows[0] });
  } catch (err) {
    next(err);
  }
};

// DELETE /api/examples/:id
export const deleteExample = async (
  req: Request,
  res: Response<ApiResponse<null>>,
  next: NextFunction
): Promise<void> => {
  try {
    const { id } = req.params;
    const result = await pool.query('DELETE FROM examples WHERE id = $1 RETURNING id', [id]);

    if (result.rows.length === 0) {
      throw new AppError('Example not found', 404);
    }

    res.json({ success: true, message: 'Example deleted successfully' });
  } catch (err) {
    next(err);
  }
};
```

**Why parameterized queries (`$1`, `$2`)?** This is the primary defense against SQL injection. The `pg` library sends the query and values separately, so the database treats user input as data, never as executable SQL. Never interpolate user input directly into a query string.

### 2.12 Routes — `server/src/routes/exampleRoutes.ts`

```typescript
import { Router } from 'express';
import {
  getExamples,
  getExampleById,
  createExample,
  updateExample,
  deleteExample,
} from '../controllers/exampleController.js';

const router = Router();

router.route('/').get(getExamples).post(createExample);
router.route('/:id').get(getExampleById).put(updateExample).delete(deleteExample);

export default router;
```

### 2.13 Server entry point — `server/src/index.ts`

```typescript
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import morgan from 'morgan';
import dotenv from 'dotenv';
import exampleRoutes from './routes/exampleRoutes.js';
import { notFound, errorHandler } from './middleware/errorMiddleware.js';

dotenv.config();

const app = express();
const PORT = process.env.PORT || 5000;

// ─── Security & Logging Middleware ───────────────────────────────────────────
app.use(helmet());
app.use(cors({
  origin: process.env.NODE_ENV === 'production'
    ? process.env.CLIENT_URL
    : 'http://localhost:5173',
  credentials: true,
}));
app.use(morgan(process.env.NODE_ENV === 'production' ? 'combined' : 'dev'));

// ─── Body Parsing ─────────────────────────────────────────────────────────────
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// ─── Health Check ─────────────────────────────────────────────────────────────
app.get('/health', (_req, res) => {
  res.json({ status: 'ok', timestamp: new Date().toISOString() });
});

// ─── API Routes ───────────────────────────────────────────────────────────────
app.use('/api/examples', exampleRoutes);

// ─── Error Handling (must be last) ────────────────────────────────────────────
app.use(notFound);
app.use(errorHandler);

app.listen(PORT, () => {
  console.log(`🚀 Server running on http://localhost:${PORT}`);
  console.log(`   Environment: ${process.env.NODE_ENV}`);
});

export default app;
```

---

## Phase 3 — Frontend (Vite + React + TypeScript)

### 3.1 Scaffold the React app

From the **root** of your project:

```bash
pnpx create-vite@latest client --template react-ts
cd client
pnpm install
```

### 3.2 Install frontend dependencies

```bash
pnpm add axios
pnpm add -D tailwindcss postcss autoprefixer eslint prettier eslint-config-prettier @typescript-eslint/eslint-plugin @typescript-eslint/parser eslint-plugin-react eslint-plugin-react-hooks
pnpx tailwindcss init -p
```

| Package | Purpose |
|---|---|
| `axios` | HTTP client. Makes API calls to the Express server with cleaner syntax than `fetch`. |
| `tailwindcss` | Utility-first CSS framework. Style components with class names directly in JSX. |
| `postcss` + `autoprefixer` | Required by Tailwind to process CSS. |
| `eslint-plugin-react` + `eslint-plugin-react-hooks` | React-specific lint rules (enforces hooks rules, JSX best practices). |

### 3.3 Configure Tailwind CSS

Open `client/tailwind.config.js` and set the `content` paths:

```javascript
/** @type {import('tailwindcss').Config} */
export default {
  content: [
    "./index.html",
    "./src/**/*.{js,ts,jsx,tsx}",
  ],
  theme: {
    extend: {},
  },
  plugins: [],
}
```

Replace the contents of `client/src/index.css`:

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

### 3.4 ESLint + Prettier for the client

Create `client/.eslintrc.json`:

```json
{
  "parser": "@typescript-eslint/parser",
  "plugins": ["@typescript-eslint", "react", "react-hooks"],
  "extends": [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "plugin:react/recommended",
    "plugin:react-hooks/recommended",
    "prettier"
  ],
  "env": {
    "browser": true,
    "es2022": true
  },
  "settings": {
    "react": { "version": "detect" }
  },
  "rules": {
    "react/react-in-jsx-scope": "off",
    "@typescript-eslint/no-unused-vars": ["error", { "argsIgnorePattern": "^_" }]
  }
}
```

Create `client/.prettierrc`:

```json
{
  "semi": true,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "es5",
  "printWidth": 100
}
```

### 3.5 Vite proxy — `client/vite.config.ts`

The proxy is the key to avoiding CORS issues in development. Any request your React app makes to `/api/*` is silently forwarded to your Express server.

```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173,
    proxy: {
      '/api': {
        target: 'http://localhost:5000',
        changeOrigin: true,
      },
    },
  },
})
```

> **Critical:** Always use relative paths in React (`/api/examples`). Never hardcode `http://localhost:5000`. The proxy makes the call on your behalf in development, and in production your server will serve the same origin.

### 3.6 Axios instance — `client/src/lib/axios.ts`

Create the directory and file:

```bash
mkdir -p src/lib
```

```typescript
import axios from 'axios';

const api = axios.create({
  baseURL: '/api',
  headers: {
    'Content-Type': 'application/json',
  },
});

// Response interceptor — centralized error handling
api.interceptors.response.use(
  (response) => response,
  (error) => {
    const message =
      error.response?.data?.message || error.message || 'An unexpected error occurred';
    console.error('API Error:', message);
    return Promise.reject(error);
  }
);

export default api;
```

**Why a custom Axios instance?** It lets you set a `baseURL` once and configure interceptors for auth headers, error logging, or token refresh logic. As your app grows, all API configuration lives in one place.

### 3.7 Types — `client/src/types/index.ts`

Mirror the types from the server. In a larger project, you'd share these via a dedicated `/packages/shared` workspace.

```bash
mkdir -p src/types
```

```typescript
export interface Example {
  id: number;
  title: string;
  description: string | null;
  created_at: string;
  updated_at: string;
}

export interface ApiResponse<T> {
  success: boolean;
  data?: T;
  message?: string;
  errors?: string[];
}
```

### 3.8 Example component — `client/src/App.tsx`

```tsx
import { useState, useEffect } from 'react';
import api from './lib/axios';
import type { Example, ApiResponse } from './types';

function App() {
  const [examples, setExamples] = useState<Example[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const fetchExamples = async () => {
      try {
        const { data } = await api.get<ApiResponse<Example[]>>('/examples');
        setExamples(data.data ?? []);
      } catch {
        setError('Failed to load examples. Is the server running?');
      } finally {
        setLoading(false);
      }
    };

    fetchExamples();
  }, []);

  return (
    <div className="min-h-screen bg-gray-50 p-8">
      <div className="max-w-2xl mx-auto">
        <h1 className="text-3xl font-bold text-gray-900 mb-6">PERN Stack — Examples</h1>

        {loading && <p className="text-gray-500">Loading...</p>}
        {error && (
          <div className="bg-red-50 border border-red-200 text-red-700 px-4 py-3 rounded">
            {error}
          </div>
        )}

        <ul className="space-y-3">
          {examples.map((ex) => (
            <li key={ex.id} className="bg-white shadow rounded-lg p-4">
              <h2 className="font-semibold text-gray-800">{ex.title}</h2>
              {ex.description && <p className="text-gray-600 text-sm mt-1">{ex.description}</p>}
            </li>
          ))}
        </ul>

        {!loading && examples.length === 0 && !error && (
          <p className="text-gray-400 text-center mt-10">
            No examples yet. POST to <code className="bg-gray-100 px-1 rounded">/api/examples</code> to add one.
          </p>
        )}
      </div>
    </div>
  );
}

export default App;
```

---

## Phase 4 — Orchestration

This phase connects both apps so you can run the entire stack with one command from the root.

### 4.1 Root `package.json`

```json
{
  "name": "my-pern-app",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "install:all": "pnpm install && pnpm --prefix client install && pnpm --prefix server install",
    "server": "pnpm --prefix server dev",
    "client": "pnpm --prefix client dev",
    "dev": "concurrently -n \"SERVER,CLIENT\" -c \"blue,green\" \"pnpm run server\" \"pnpm run client\"",
    "build:client": "pnpm --prefix client build",
    "build:server": "pnpm --prefix server build",
    "lint": "pnpm --prefix server lint && pnpm --prefix client lint"
  },
  "devDependencies": {
    "concurrently": "^8.x.x"
  }
}
```

| Script | What it does |
|---|---|
| `pnpm run install:all` | Installs dependencies for root, client, and server in one command. |
| `pnpm run server` | Starts only the backend (with nodemon + ts-node). |
| `pnpm run client` | Starts only the frontend (with Vite). |
| `pnpm run dev` | Starts **both** simultaneously with color-coded output. This is your daily driver. |
| `pnpm run lint` | Runs ESLint across both client and server. |

### 4.2 First-time setup for collaborators

When someone clones your repo, they run:

```bash
pnpm run install:all
```

Then they copy the environment template and fill in their values:

```bash
cp server/.env.example server/.env
```

Then they're ready to develop.

---

## Phase 5 — Database (Docker + PostgreSQL)

### Why Docker for PostgreSQL?

Installing PostgreSQL locally differs by operating system, version, and architecture. Docker guarantees the same behavior everywhere. One command starts it; one command stops it. Your data persists in a named volume even when the container is not running.

### 5.1 Create `docker-compose.yml` in the root

```yaml
services:
  postgres:
    image: postgres:16
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: my_pern_app
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./server/db/init.sql:/docker-entrypoint-initdb.d/init.sql

volumes:
  postgres-data:
```

> **The `init.sql` mount:** Docker will automatically run any `.sql` file placed in `/docker-entrypoint-initdb.d/` when the container is first created. This is how we seed the database schema.

### 5.2 Create the initial schema — `server/db/init.sql`

```bash
mkdir -p server/db
```

```sql
-- Create the examples table
CREATE TABLE IF NOT EXISTS examples (
  id          SERIAL PRIMARY KEY,
  title       VARCHAR(255) NOT NULL,
  description TEXT,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Automatically update the updated_at column on row modification
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER set_updated_at
  BEFORE UPDATE ON examples
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();

-- Seed with sample data
INSERT INTO examples (title, description) VALUES
  ('Hello, PERN!', 'Your first record in PostgreSQL.'),
  ('Type Safety', 'TypeScript + Zod means runtime and compile-time validation.'),
  ('Connection Pooling', 'pg.Pool handles concurrent requests efficiently.')
ON CONFLICT DO NOTHING;
```

**Why a trigger for `updated_at`?** It guarantees the field is always accurate, regardless of whether the application layer remembers to update it. The database enforces this invariant.

### 5.3 Start the database

```bash
docker compose up -d
```

The `-d` flag runs it in the background.

Useful commands:

```bash
docker compose down           # Stop the container (data is preserved in the volume)
docker compose down -v        # Stop AND delete all data (fresh start)
docker compose logs postgres  # View database logs
docker compose exec postgres psql -U postgres -d my_pern_app  # Open a SQL shell
```

---

## Phase 6 — Production Database (Supabase)

Supabase is a hosted PostgreSQL service with a generous free tier. Because it's standard PostgreSQL, no code changes are required — you only swap the `DATABASE_URL`.

### 6.1 Create a Supabase project

1. Go to [supabase.com](https://supabase.com) and sign in.
2. Click **New Project**.
3. Choose an organization, give it a name (e.g., `my-pern-app`), set a strong **database password**, and choose the region closest to your users.
4. Wait ~2 minutes for provisioning to complete.

### 6.2 Get your connection string

Supabase provides two connection methods. For a Node.js backend, use **connection pooling via Supavisor** (the recommended approach for serverless and long-running servers alike).

1. In your Supabase dashboard, go to **Project Settings → Database**.
2. Scroll to **Connection string**.
3. Select the **URI** tab and choose **Transaction** mode (port `6543`).

You will see a string like:

```
postgresql://postgres.xxxxxxxxxxxx:[YOUR-PASSWORD]@aws-0-us-east-1.pooler.supabase.com:6543/postgres
```

> **Transaction mode vs. Session mode:** Transaction mode (port `6543`) is ideal for connection pooling from Node.js servers. It multiplexes many application connections through a smaller pool of actual database connections. Session mode (port `5432`) gives you a dedicated connection — required only if you use PostgreSQL session-level features like advisory locks or `SET LOCAL`.

### 6.3 Configure environment variables for production

In your production environment (e.g., Railway, Render, Fly.io), set these environment variables:

```
NODE_ENV=production
PORT=5000
DATABASE_URL=postgresql://postgres.xxxxxxxxxxxx:[YOUR-PASSWORD]@aws-0-us-east-1.pooler.supabase.com:6543/postgres
CLIENT_URL=https://your-frontend-domain.com
```

**Never commit production credentials to Git.** Use your hosting platform's secrets/environment variable management.

### 6.4 Run your schema on Supabase

Two ways to apply `server/db/init.sql` to your Supabase database:

**Option A — Supabase SQL Editor (easiest):**
1. Go to your Supabase project → **SQL Editor**.
2. Paste the contents of `server/db/init.sql`.
3. Click **Run**.

**Option B — `psql` CLI:**

Get the **direct connection string** (not the pooler) from Project Settings → Database (port `5432`):

```bash
psql "postgresql://postgres:[YOUR-PASSWORD]@db.xxxxxxxxxxxx.supabase.co:5432/postgres" \
  -f server/db/init.sql
```

### 6.5 SSL is already handled

Notice the SSL configuration in `server/src/config/db.ts`:

```typescript
ssl: process.env.NODE_ENV === 'production' ? { rejectUnauthorized: false } : false,
```

When `NODE_ENV=production`, the pool automatically connects over SSL, which Supabase requires. In local development, SSL is disabled so your Docker container works without a certificate.

---

## Phase 7 — AI Onboarding (Claude Code)

> **This phase is optional.** Skip it if you don't use Claude Code.

Initialize the AI assistant *after* the project structure is in place. This way, Claude scans your directory and understands the monorepo layout, TypeScript configuration, and database conventions from the start.

### 7.1 Initialize

From the **root** directory:

```bash
claude init
```

### 7.2 Create `CLAUDE.md`

This file acts as persistent instructions for Claude Code — like a `.editorconfig` but for your AI assistant. Create `CLAUDE.md` in the root:

```markdown
# Project Rules — PERN Monorepo

## Architecture
- Monorepo: root orchestrates, `/client` is Vite + React + TypeScript, `/server` is Express + TypeScript.
- CommonJS on the server (`require`/`exports` is fine, but `import`/`export` preferred via TypeScript compilation).
- Database: PostgreSQL via `pg` connection pool. Pool is a singleton in `/server/src/config/db.ts`.

## Running
- `pnpm run dev` from the ROOT starts both client and server.
- Client: port 5173. Server: port 5000.
- Database: `docker compose up -d` must be running first.

## API Calls from React
- The client uses a Vite proxy: `/api` → `http://localhost:5000`.
- Always use relative paths: `api.get('/examples')`.
- NEVER hardcode `http://localhost:5000` in React code.
- All API calls go through the Axios instance in `/client/src/lib/axios.ts`.

## Database
- Use parameterized queries (`$1`, `$2`) — NEVER string interpolation.
- The `pg` pool is exported from `/server/src/config/db.ts`. Import it, do not create new pools.
- Schema is in `/server/db/init.sql`. For schema changes, add a migration SQL file in `/server/db/`.

## Validation
- Use Zod schemas in `/server/src/schemas/` to validate all request bodies.
- Use `schema.safeParse()` for explicit success/failure handling.
- Infer TypeScript types from Zod schemas: `z.infer<typeof mySchema>`.

## Error Handling
- Throw `AppError` (from `/server/src/middleware/errorMiddleware.ts`) with a status code: `throw new AppError('Not found', 404)`.
- Pass unexpected errors to `next(err)` in controllers.
- The central error handler in `errorMiddleware.ts` formats all responses.

## Types
- Shared types live in `/server/src/types/index.ts` and are mirrored in `/client/src/types/index.ts`.
- Server types use `Date` for timestamps; client types use `string` (JSON serialization converts them).

## Conventions
- Controllers: camelCase exports (`getExamples`, `createExample`).
- Routes: mounted at `/api/<resource>`.
- Secrets: never commit `.env`. Use `.env.example` for documentation.
- SQL table names: snake_case. TypeScript interfaces: PascalCase.
```

---

## Running the App

### First time

```bash
# 1. Start the database
docker compose up -d

# 2. Install everything
pnpm run install:all

# 3. Copy and fill in environment variables
cp server/.env.example server/.env

# 4. Start developing
pnpm run dev
```

### Every day after that

```bash
docker compose up -d   # if not already running
pnpm run dev
```

Your app is live:

| Service | URL |
|---|---|
| Frontend | http://localhost:5173 |
| Backend | http://localhost:5000 |
| Health Check | http://localhost:5000/health |
| API Test | http://localhost:5000/api/examples |

---

## What to Build Next

This skeleton is deliberately minimal. Here's what to add based on your project:

- **Authentication** — JWT with `jsonwebtoken` and `bcryptjs`, or session-based auth with `express-session` + `connect-pg-simple` (stores sessions in PostgreSQL).
- **Database migrations** — Replace the single `init.sql` with a migration tool like `node-pg-migrate` or `db-migrate` for versioned, reversible schema changes.
- **File uploads** — `multer` for handling multipart form data, with Supabase Storage as the file destination.
- **Testing** — `vitest` for frontend unit tests, `supertest` + `vitest` for API integration tests against a test database.
- **Rate limiting** — `express-rate-limit` to protect against brute force and abuse.
- **React Query** — `@tanstack/react-query` replaces manual `useEffect` + `useState` data fetching with caching, background refetching, and optimistic updates.
- **Production build** — Serve the Vite build from Express:
  ```typescript
  // In server/src/index.ts, after routes:
  import path from 'path';
  if (process.env.NODE_ENV === 'production') {
    app.use(express.static(path.join(__dirname, '../../client/dist')));
    app.get('*', (_req, res) => {
      res.sendFile(path.join(__dirname, '../../client/dist/index.html'));
    });
  }
  ```

---

## Toolkit Reference

Quick reference for every tool in this stack and why it was chosen.

### PostgreSQL + `pg`

A relational database. Data is organized into tables with defined columns and types. SQL gives you joins, transactions, constraints, and aggregations that document databases lack. The `pg` package is the official Node.js driver — it manages a connection pool, executes parameterized queries, and handles PostgreSQL-specific data types.

### Supabase

A managed PostgreSQL hosting platform. It is standard PostgreSQL under the hood — there is no vendor lock-in at the query level. Supabase adds a dashboard, automatic backups, connection pooling via Supavisor, and optional extras (Auth, Storage, Realtime) when your project needs them.

### TypeScript

The JavaScript superset that adds static types. On the server, it catches type mismatches between your SQL results and your API responses at compile time. On the client, it ensures your React components receive the correct props. Strict mode (`"strict": true` in `tsconfig.json`) enables the full set of checks — the strictest, most valuable option.

### Zod

A runtime validation library that infers TypeScript types from its schemas. Because TypeScript types are erased at runtime, you cannot trust `req.body` just because it's typed — a malicious or broken client can send anything. Zod validates the actual shape of incoming data and gives you a clean, typed object if it passes.

### Helmet

A single `app.use(helmet())` call sets 15+ security-related HTTP response headers. It prevents clickjacking (`X-Frame-Options`), MIME type sniffing (`X-Content-Type-Options`), and helps enforce HTTPS (`Strict-Transport-Security`). Zero configuration needed to start.

### Morgan

HTTP request logger middleware. When something goes wrong, Morgan's output tells you exactly what request was made, what status code was returned, and how long it took. The `'dev'` format gives colorized, concise output like `GET /api/examples 200 4.231 ms`. Switch to `'combined'` (Apache format) in production for structured logs.

### CORS

Controls which domains can call your API. In development, `app.use(cors())` allows all origins (safe because the server only runs locally). In production, you restrict it to your specific frontend domain to prevent unauthorized cross-origin requests.

### Dotenv

Reads a `.env` file and loads its key-value pairs into `process.env`. Your code references `process.env.DATABASE_URL`, and the actual value stays out of your source code and Git history.

### ESLint + Prettier

Two separate concerns working together. ESLint finds code quality problems (unused variables, incorrect hook usage, potential bugs). Prettier formats code consistently (indentation, quotes, trailing commas). `eslint-config-prettier` disables any ESLint formatting rules that would conflict with Prettier, so they never fight.

### Vite

Replaces Create React App (which is no longer maintained). Vite serves files via native ES Modules during development, making startup and hot reloads near-instant. The proxy configuration eliminates CORS issues in development by forwarding `/api` requests to your Express server transparently.

### Axios

HTTP client for the browser. Compared to `fetch`, Axios automatically parses JSON responses, has cleaner error handling (4xx/5xx responses throw by default), and supports interceptors for cross-cutting concerns like auth headers and error logging.

### Tailwind CSS

A utility-first CSS framework. Instead of writing `.btn { padding: 8px 16px; background: blue; }`, you write `className="px-4 py-2 bg-blue-500"`. The result is faster iteration, no naming fatigue, and styles that are co-located with the markup they affect.

### Nodemon + ts-node

`nodemon` watches your source files and restarts the Node.js process on save. `ts-node` runs TypeScript files directly without a separate compile step. Together, they give you a hot-reloading TypeScript development server — save your controller and the server restarts in under a second.

### Docker Compose

Defines and runs the PostgreSQL container. One `docker compose up -d` gives you a database with the correct version, credentials, and initial schema. The named volume (`postgres-data`) ensures your data survives container restarts. No OS-specific PostgreSQL installation is needed.

### pnpm

A faster, more disk-efficient alternative to `npm`. It stores packages in a global content-addressable store and links them into `node_modules` with hard links — so if ten projects use the same version of React, it's only stored once on disk. Use `pnpx` to execute binaries (equivalent to `npx`).

---

## Final Project Structure

```
my-pern-app/
├── client/                         # Vite + React + TypeScript frontend
│   ├── public/
│   ├── src/
│   │   ├── lib/
│   │   │   └── axios.ts            # Configured Axios instance
│   │   ├── types/
│   │   │   └── index.ts            # Mirrored TypeScript types
│   │   ├── App.tsx
│   │   └── main.tsx
│   ├── index.html
│   ├── .eslintrc.json
│   ├── .prettierrc
│   ├── package.json
│   ├── tailwind.config.js
│   ├── tsconfig.json
│   └── vite.config.ts
├── server/                         # Express + TypeScript backend
│   ├── db/
│   │   └── init.sql                # Schema + seed data
│   ├── src/
│   │   ├── config/
│   │   │   └── db.ts               # pg connection pool
│   │   ├── controllers/
│   │   │   └── exampleController.ts
│   │   ├── middleware/
│   │   │   └── errorMiddleware.ts  # AppError class + handlers
│   │   ├── routes/
│   │   │   └── exampleRoutes.ts
│   │   ├── schemas/
│   │   │   └── exampleSchema.ts    # Zod validation schemas
│   │   ├── types/
│   │   │   └── index.ts            # Shared TypeScript types
│   │   └── index.ts                # Entry point
│   ├── .env                        # Secrets (never committed)
│   ├── .env.example                # Template (committed)
│   ├── .eslintrc.json
│   ├── .prettierrc
│   ├── package.json
│   └── tsconfig.json
├── .gitignore
├── CLAUDE.md                       # AI assistant rules (optional)
├── docker-compose.yml
├── package.json                    # Root orchestration only
└── README.md
```