# SlipStream Analytics - Complete Technical Specification

**Version**: 1.0
**Last Updated**: 2025-12-28
**Purpose**: Comprehensive specification enabling complete recreation of SlipStream Analytics without access to source code

---

## Table of Contents

1. [Executive Overview](#executive-overview)
2. [Technology Stack](#technology-stack)
3. [Project Structure](#project-structure)
4. [Database Schema](#database-schema)
5. [Backend Architecture](#backend-architecture)
6. [Frontend Architecture](#frontend-architecture)
7. [API Specification](#api-specification)
8. [Service Layer Implementation](#service-layer-implementation)
9. [Key Algorithms](#key-algorithms)
10. [Configuration & Environment](#configuration--environment)
11. [Development & Deployment](#development--deployment)
12. [Critical Deep Dive: Data Sync & Identity Matching](#critical-deep-dive-data-sync--identity-matching)

---

## Executive Overview

### What is SlipStream Analytics?

SlipStream Analytics is a unified developer productivity platform that answers the critical question: **"How does AI usage correlate with actual output?"**

### Core Value Proposition

- **Unified View**: Aggregates data from Claude AI, GitHub, and Jira into a single dashboard
- **Identity Resolution**: Automatically links users across platforms via email matching
- **Productivity Insights**: Correlates AI costs with commit velocity, PR throughput, and ticket completion

### Key Features

1. **Three-Layer Identity System**: Canonical user mapping with auto-linking and manual override
2. **Real-Time Sync**: GitHub, Jira, and Claude metrics with configurable sync schedules
3. **Rich Analytics**: Per-user dashboards with charts, activity streams, and cost tracking
4. **Developer-Friendly**: Type-safe queries, comprehensive debugging tools, and clear APIs

---

## Technology Stack

### Backend Dependencies

```json
{
  "hono": "^4.11.1",                    // Lightweight web framework (Express alternative)
  "better-sqlite3": "^12.5.0",          // Fast synchronous SQLite
  "kysely": "^0.28.9",                  // Type-safe SQL query builder
  "node-cron": "^4.2.1",                // Task scheduling (daily syncs)
  "zod": "^3.24.1",                     // Schema validation
  "cors": "^2.8.5",                     // Cross-origin resource sharing
  "dotenv": "^16.5.0",                  // Environment variable loading
  "typescript": "^5.9.3",               // Type safety
  "tsx": "^4.20.1"                      // TypeScript execution (dev)
}
```

### Frontend Dependencies

```json
{
  "react": "^19.2.0",                   // UI framework
  "react-dom": "^19.2.0",               // DOM rendering
  "vite": "^7.2.4",                     // Build tool & dev server
  "tailwindcss": "^4.1.18",             // Utility-first CSS
  "recharts": "^3.6.0",                 // Charting library
  "framer-motion": "^12.23.26",         // Animation library
  "lucide-react": "^0.561.0",           // Icon library
  "react-hook-form": "^7.69.0",         // Form handling
  "@radix-ui/react-*": "various",       // Headless UI primitives
  "typescript": "^5.9.3"                // Type safety
}
```

### System Requirements

- **Node.js**: v18+ (LTS recommended)
- **npm**: v8+ (or pnpm/yarn)
- **OS**: macOS, Linux, or Windows with WSL
- **Disk**: 100MB minimum (SQLite database grows with data)
- **Memory**: 512MB minimum for backend, 1GB for frontend dev

---

## Project Structure

```
slipstream/
├── src/                              # Backend source code
│   ├── db/
│   │   ├── index.ts                  # Database connection & migrations
│   │   └── schema.ts                 # TypeScript table interfaces
│   ├── routes/
│   │   ├── analytics.ts              # Developer metrics endpoints
│   │   ├── config.ts                 # API key management
│   │   ├── data.ts                   # Raw data viewer
│   │   ├── ingest.ts                 # Claude metrics ingestion
│   │   ├── sync.ts                   # Sync endpoints (GitHub, Jira)
│   │   └── unified.ts                # Core unified view logic
│   ├── services/
│   │   ├── claude.ts                 # Claude API integration
│   │   ├── github.ts                 # GitHub API integration
│   │   ├── jira.ts                   # Jira API integration
│   │   ├── identity.ts               # Identity auto-linking
│   │   └── scheduler.ts              # Daily cron job
│   ├── logic/
│   │   └── correlation.ts            # Correlation algorithms (unused)
│   └── server.ts                     # Main entry point
│
├── frontend/                         # Frontend source code
│   ├── src/
│   │   ├── components/
│   │   │   ├── Layout.tsx            # Sidebar navigation
│   │   │   ├── SearchableSelect.tsx  # Custom dropdown
│   │   │   └── ui/                   # Radix UI wrappers
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx         # Per-user analytics
│   │   │   ├── UnifiedView.tsx       # Cross-platform table
│   │   │   ├── Settings.tsx          # API keys & configuration
│   │   │   ├── DataViewer.tsx        # Raw data explorer
│   │   │   ├── Identities.tsx        # Identity management
│   │   │   └── Status.tsx            # Sync status
│   │   ├── lib/
│   │   │   ├── api.ts                # API base URL config
│   │   │   └── utils.ts              # Utility functions
│   │   ├── App.tsx                   # Route handler
│   │   └── main.tsx                  # Entry point
│   ├── package.json
│   ├── vite.config.ts
│   ├── tailwind.config.ts
│   └── tsconfig.json
│
├── scripts/
│   └── sync-github.ts                # Standalone GitHub sync
├── package.json                      # Backend dependencies
├── tsconfig.json                     # Backend TypeScript config
├── .env                              # Environment variables (gitignored)
├── .env.example                      # Template
├── slipstream.db                     # SQLite database (generated)
├── Dockerfile                        # Backend container
├── docker-compose.yml                # Full stack orchestration
├── CLAUDE.md                         # Instructions for Claude Code
├── README.md                         # User-facing documentation
└── SPECIFICATION.md                  # This file
```

---

## Database Schema

### Overview

SlipStream uses **SQLite** via `better-sqlite3` with **Kysely** for type-safe queries. The database implements a **three-layer identity resolution system**:

1. **provider_users**: Raw user data from each provider
2. **identities**: Canonical mappings (email-based)
3. **identity_rejections**: Explicitly rejected links

**Canonical User ID**: Claude user email serves as the universal identifier. All GitHub and Jira users correlate to this via the `identities` table.

### Table Definitions

#### `api_configs`

Stores API keys and provider metadata per organization.

```sql
CREATE TABLE IF NOT EXISTS api_configs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    org_id TEXT NOT NULL,
    provider TEXT NOT NULL CHECK(provider IN ('claude', 'github', 'jira')),
    api_key TEXT NOT NULL,
    metadata TEXT,                    -- JSON: {email, domain} for Jira
    updated_at TEXT NOT NULL,         -- ISO 8601 timestamp
    UNIQUE(org_id, provider)
);
```

**TypeScript Interface**:
```typescript
interface ApiConfigsTable {
    id: Generated<number>
    org_id: string
    provider: 'claude' | 'github' | 'jira'
    api_key: string
    metadata: string | null
    updated_at: string
}
```

#### `provider_users`

Raw user data from each provider with emails, usernames, and metadata.

```sql
CREATE TABLE IF NOT EXISTS provider_users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    org_id TEXT NOT NULL,
    provider TEXT NOT NULL CHECK(provider IN ('claude', 'github', 'jira')),
    provider_user_id TEXT NOT NULL,   -- GitHub ID, Jira accountId, Claude email
    email TEXT,                       -- Normalized lowercase email
    display_name TEXT,                -- Full name or display name
    username TEXT,                    -- GitHub username, Jira key, etc.
    raw_json TEXT,                    -- Full API response JSON
    updated_at TEXT NOT NULL,
    UNIQUE(org_id, provider, provider_user_id)
);

CREATE INDEX IF NOT EXISTS idx_provider_users_email
    ON provider_users(email) WHERE email IS NOT NULL;
```

**TypeScript Interface**:
```typescript
interface ProviderUsersTable {
    id: Generated<number>
    org_id: string
    provider: 'claude' | 'github' | 'jira'
    provider_user_id: string
    email: string | null
    display_name: string | null
    username: string | null
    raw_json: string | null
    updated_at: string
}
```

#### `identities`

Canonical user mapping linking provider IDs to universal email identifiers.

```sql
CREATE TABLE IF NOT EXISTS identities (
    user_id TEXT NOT NULL,            -- Canonical email (Claude user email)
    provider TEXT NOT NULL CHECK(provider IN ('claude', 'github', 'jira')),
    provider_user_id TEXT NOT NULL,   -- Provider-specific ID
    confidence INTEGER NOT NULL DEFAULT 100,  -- 100 = auto-linked, others = manual
    created_at TEXT NOT NULL,
    PRIMARY KEY (user_id, provider, provider_user_id)
);

CREATE INDEX IF NOT EXISTS idx_identities_provider
    ON identities(provider, provider_user_id);
```

**TypeScript Interface**:
```typescript
interface IdentitiesTable {
    user_id: string
    provider: 'claude' | 'github' | 'jira'
    provider_user_id: string
    confidence: number
    created_at: string
}
```

#### `identity_rejections`

Explicitly rejected identity links to prevent incorrect auto-linking.

```sql
CREATE TABLE IF NOT EXISTS identity_rejections (
    user_id TEXT NOT NULL,
    provider TEXT NOT NULL CHECK(provider IN ('claude', 'github', 'jira')),
    provider_user_id TEXT NOT NULL,
    created_at TEXT NOT NULL,
    PRIMARY KEY (user_id, provider, provider_user_id)
);
```

**TypeScript Interface**:
```typescript
interface IdentityRejectionsTable {
    user_id: string
    provider: 'claude' | 'github' | 'jira'
    provider_user_id: string
    created_at: string
}
```

#### `ai_metric_samples`

Claude usage metrics indexed by canonical email.

```sql
CREATE TABLE IF NOT EXISTS ai_metric_samples (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    org_id TEXT NOT NULL,
    user_id TEXT NOT NULL,            -- Canonical email
    bucket_start TEXT NOT NULL,       -- ISO date (YYYY-MM-DD)
    bucket_end TEXT NOT NULL,
    repo_hint TEXT,                   -- Terminal type or repo context
    metrics_json TEXT NOT NULL        -- JSON blob (see below)
);

CREATE INDEX IF NOT EXISTS idx_ai_samples_user
    ON ai_metric_samples(org_id, user_id);
CREATE INDEX IF NOT EXISTS idx_ai_samples_date
    ON ai_metric_samples(bucket_start);
```

**metrics_json Schema**:
```json
{
  "claude_code.session.count": 5,
  "claude_code.token.usage": 250000,
  "claude_code.cost.usage": 1500,
  "claude_code.active_time.total": 7200,
  "claude_code.lines_of_code.count": 1500,
  "raw_api_data": { ... }
}
```

**TypeScript Interface**:
```typescript
interface AiMetricSamplesTable {
    id: Generated<number>
    org_id: string
    user_id: string
    bucket_start: string
    bucket_end: string
    repo_hint: string | null
    metrics_json: string
}
```

#### `repos`

GitHub repository metadata.

```sql
CREATE TABLE IF NOT EXISTS repos (
    id TEXT PRIMARY KEY,              -- GitHub repo ID as string
    org_id TEXT NOT NULL,
    name TEXT NOT NULL,               -- e.g., "frontend-app"
    full_name TEXT NOT NULL,          -- e.g., "example-org/frontend-app"
    url TEXT NOT NULL                 -- GitHub HTML URL
);
```

**TypeScript Interface**:
```typescript
interface ReposTable {
    id: string
    org_id: string
    name: string
    full_name: string
    url: string
}
```

#### `commits`

Git commit data with line statistics.

```sql
CREATE TABLE IF NOT EXISTS commits (
    sha TEXT PRIMARY KEY,             -- Git commit hash
    repo_id TEXT NOT NULL,
    author_user_id TEXT,              -- GitHub username
    committer_email TEXT,             -- Raw email from Git
    created_at TEXT NOT NULL,         -- ISO timestamp
    files_changed INTEGER NOT NULL DEFAULT 0,
    lines_added INTEGER NOT NULL DEFAULT 0,
    lines_deleted INTEGER NOT NULL DEFAULT 0,
    message TEXT NOT NULL,
    branch TEXT,
    org_id TEXT NOT NULL,
    FOREIGN KEY (repo_id) REFERENCES repos(id)
);

CREATE INDEX IF NOT EXISTS idx_commits_author
    ON commits(author_user_id);
CREATE INDEX IF NOT EXISTS idx_commits_email
    ON commits(committer_email);
CREATE INDEX IF NOT EXISTS idx_commits_date
    ON commits(created_at);
```

**TypeScript Interface**:
```typescript
interface CommitsTable {
    sha: string
    repo_id: string
    author_user_id: string | null
    committer_email: string | null
    created_at: string
    files_changed: number
    lines_added: number
    lines_deleted: number
    message: string
    branch: string | null
    org_id: string
}
```

#### `pull_requests`

Pull request metadata and status.

```sql
CREATE TABLE IF NOT EXISTS pull_requests (
    id TEXT PRIMARY KEY,              -- GitHub PR ID
    repo_id TEXT NOT NULL,
    number INTEGER NOT NULL,          -- PR number (repo-specific)
    author_user_id TEXT,              -- GitHub username
    author_email TEXT,                -- Raw email
    opened_at TEXT NOT NULL,
    merged_at TEXT,                   -- NULL if not merged
    title TEXT NOT NULL,
    body TEXT,
    base_branch TEXT NOT NULL,        -- Target branch
    head_branch TEXT NOT NULL,        -- Source branch
    status TEXT NOT NULL,             -- 'open', 'closed', 'merged'
    updated_at TEXT,
    org_id TEXT NOT NULL,
    FOREIGN KEY (repo_id) REFERENCES repos(id)
);

CREATE INDEX IF NOT EXISTS idx_prs_author
    ON pull_requests(author_user_id);
CREATE INDEX IF NOT EXISTS idx_prs_email
    ON pull_requests(author_email);
```

**TypeScript Interface**:
```typescript
interface PullRequestsTable {
    id: string
    repo_id: string
    number: number
    author_user_id: string | null
    author_email: string | null
    opened_at: string
    merged_at: string | null
    title: string
    body: string
    base_branch: string
    head_branch: string
    status: string
    updated_at: string | null
    org_id: string
}
```

#### `pull_events`

PR reviews, approvals, and comments.

```sql
CREATE TABLE IF NOT EXISTS pull_events (
    id TEXT PRIMARY KEY,
    pull_request_id TEXT NOT NULL,
    event_type TEXT NOT NULL CHECK(event_type IN ('approval', 'changes_requested', 'comment')),
    user_id TEXT,
    created_at TEXT NOT NULL,
    FOREIGN KEY (pull_request_id) REFERENCES pull_requests(id)
);
```

**TypeScript Interface**:
```typescript
interface PullEventsTable {
    id: string
    pull_request_id: string
    event_type: 'approval' | 'changes_requested' | 'comment'
    user_id: string | null
    created_at: string
}
```

#### `jira_projects`

Jira project metadata.

```sql
CREATE TABLE IF NOT EXISTS jira_projects (
    key TEXT PRIMARY KEY,             -- Project key e.g., "AUTH"
    name TEXT NOT NULL,
    org_id TEXT NOT NULL
);
```

**TypeScript Interface**:
```typescript
interface JiraProjectsTable {
    key: string
    name: string
    org_id: string
}
```

#### `jira_sprints`

Jira sprint information.

```sql
CREATE TABLE IF NOT EXISTS jira_sprints (
    id TEXT PRIMARY KEY,
    project_key TEXT NOT NULL,
    state TEXT NOT NULL CHECK(state IN ('active', 'closed', 'future')),
    start_date TEXT,
    end_date TEXT,
    FOREIGN KEY (project_key) REFERENCES jira_projects(key)
);
```

**TypeScript Interface**:
```typescript
interface JiraSprintsTable {
    id: string
    project_key: string
    state: 'active' | 'closed' | 'future'
    start_date: string | null
    end_date: string | null
}
```

#### `jira_issues`

Jira issue/ticket data.

```sql
CREATE TABLE IF NOT EXISTS jira_issues (
    key TEXT PRIMARY KEY,             -- Issue key e.g., "AUTH-123"
    project_key TEXT NOT NULL,
    sprint_id TEXT,
    type TEXT NOT NULL,               -- 'Story', 'Task', 'Bug'
    status TEXT NOT NULL,             -- 'To Do', 'In Progress', 'Done'
    status_category TEXT NOT NULL,
    assignee_user_id TEXT,            -- Jira accountId
    assignee_email TEXT,              -- Normalized email
    points INTEGER,                   -- Story points
    parent_key TEXT,                  -- Parent epic/story
    product TEXT,
    updated_at TEXT NOT NULL,
    status_change_at TEXT,
    labels TEXT NOT NULL DEFAULT '[]',        -- JSON array
    components TEXT NOT NULL DEFAULT '[]',    -- JSON array
    description_length INTEGER NOT NULL DEFAULT 0,
    ac_present INTEGER NOT NULL DEFAULT 0,    -- 0 or 1
    org_id TEXT NOT NULL,
    FOREIGN KEY (project_key) REFERENCES jira_projects(key),
    FOREIGN KEY (sprint_id) REFERENCES jira_sprints(id)
);

CREATE INDEX IF NOT EXISTS idx_jira_issues_assignee
    ON jira_issues(assignee_user_id);
CREATE INDEX IF NOT EXISTS idx_jira_issues_email
    ON jira_issues(assignee_email);
```

**TypeScript Interface**:
```typescript
interface JiraIssuesTable {
    key: string
    project_key: string
    sprint_id: string | null
    type: string
    status: string
    status_category: string
    assignee_user_id: string | null
    assignee_email: string | null
    points: number | null
    parent_key: string | null
    product: string | null
    updated_at: string
    status_change_at: string | null
    labels: string
    components: string
    description_length: number
    ac_present: number
    org_id: string
}
```

### Migration System

**Location**: `src/db/index.ts`

**Strategy**: Run migrations on server startup via `migrateToLatest()` function.

**Implementation Pattern**:
```typescript
export async function migrateToLatest() {
    // 1. Create all tables with IF NOT EXISTS
    await db.schema.createTable('api_configs')
        .ifNotExists()
        .addColumn('id', 'integer', col => col.primaryKey().autoIncrement())
        .addColumn('org_id', 'text', col => col.notNull())
        // ... (all columns)
        .execute()

    // 2. Add new columns to existing tables (idempotent via try-catch)
    try {
        await db.schema.alterTable('commits')
            .addColumn('branch', 'text')
            .execute()
    } catch (e) {
        // Column already exists, ignore
    }

    // 3. Create indexes
    await db.schema.createIndex('idx_commits_author')
        .ifNotExists()
        .on('commits')
        .column('author_user_id')
        .execute()
}
```

**Key Points**:
- Idempotent: Can run multiple times without errors
- Preserves data: Never drops tables or columns
- Auto-executes: Runs on server start before accepting requests

---

## Backend Architecture

### Entry Point (`src/server.ts`)

```typescript
import { Hono } from 'hono'
import { cors } from 'hono/cors'
import { migrateToLatest } from './db'
import { scheduler } from './services/scheduler'

const app = new Hono()

// Middleware
app.use('/*', cors())

// Routes
import configRoutes from './routes/config'
import analyticsRoutes from './routes/analytics'
import syncRoutes from './routes/sync'
import dataRoutes from './routes/data'
import unifiedRoutes from './routes/unified'
import ingestRoutes from './routes/ingest'

app.route('/api/config', configRoutes)
app.route('/api/developer', analyticsRoutes)
app.route('/api/sync', syncRoutes)
app.route('/api/data', dataRoutes)
app.route('/api/unified', unifiedRoutes)
app.route('/api', ingestRoutes)

// Initialization
async function init() {
    await migrateToLatest()
    await seedEnvConfigs()
    scheduler.start()
}

init().then(() => {
    const port = process.env.PORT || 3042
    console.log(`Server is running on port ${port}`)
    app.listen(port)
})

// Seed API keys from .env into database
async function seedEnvConfigs() {
    const orgId = process.env.ORG_ID || 'myorg'

    if (process.env.CLAUDE_API_KEY) {
        await db.insertInto('api_configs')
            .values({
                org_id: orgId,
                provider: 'claude',
                api_key: process.env.CLAUDE_API_KEY,
                metadata: null,
                updated_at: new Date().toISOString()
            })
            .onConflict(oc => oc.columns(['org_id', 'provider']).doNothing())
            .execute()
        console.log('[Config] Seeded claude key from .env')
    }

    // Similar for GITHUB_TOKEN and JIRA_API_TOKEN
}
```

### Database Layer (`src/db/`)

**index.ts**:
```typescript
import Database from 'better-sqlite3'
import { Kysely, SqliteDialect } from 'kysely'
import type { Database as DatabaseType } from './schema'

const dbPath = process.env.DB_PATH || 'slipstream.db'
const sqlite = new Database(dbPath)

export const db = new Kysely<DatabaseType>({
    dialect: new SqliteDialect({ database: sqlite })
})

export { migrateToLatest } from './migrations'
```

**schema.ts**:
```typescript
import type { Generated } from 'kysely'

export interface Database {
    api_configs: ApiConfigsTable
    provider_users: ProviderUsersTable
    identities: IdentitiesTable
    identity_rejections: IdentityRejectionsTable
    ai_metric_samples: AiMetricSamplesTable
    repos: ReposTable
    commits: CommitsTable
    pull_requests: PullRequestsTable
    pull_events: PullEventsTable
    jira_projects: JiraProjectsTable
    jira_sprints: JiraSprintsTable
    jira_issues: JiraIssuesTable
}

// (Table interfaces defined in Database Schema section)
```

### Utility Functions

**Email Normalization** (used across all services):
```typescript
function normalizeEmail(email?: string | null): string | null {
    if (!email) return null
    const trimmed = email.trim()
    return trimmed.length > 0 ? trimmed.toLowerCase() : null
}
```

---

## Frontend Architecture

### Routing (`src/App.tsx`)

```typescript
import { useEffect, useState } from 'react'
import { Layout } from './components/Layout'
import { Dashboard } from './pages/Dashboard'
import { UnifiedView } from './pages/UnifiedView'
import { Identities } from './pages/Identities'
import { DataViewer } from './pages/DataViewer'
import { Status } from './pages/Status'
import { Settings } from './pages/Settings'

export function App() {
    const [currentPath, setCurrentPath] = useState(window.location.hash.split('?')[0])

    useEffect(() => {
        const handleHashChange = () => {
            setCurrentPath(window.location.hash.split('?')[0])
        }
        window.addEventListener('hashchange', handleHashChange)
        return () => window.removeEventListener('hashchange', handleHashChange)
    }, [])

    const renderPage = () => {
        switch (currentPath) {
            case '#/': return <Dashboard />
            case '#/unified': return <UnifiedView />
            case '#/identities': return <Identities />
            case '#/data': return <DataViewer />
            case '#/status': return <Status />
            case '#/settings': return <Settings />
            default: return <Dashboard />
        }
    }

    return <Layout>{renderPage()}</Layout>
}
```

### Layout Component (`src/components/Layout.tsx`)

```typescript
import { Activity, LayoutDashboard, Users, Database, Radio, Settings as SettingsIcon } from 'lucide-react'
import { motion } from 'framer-motion'

const navItems = [
    { href: '#/', label: 'Dashboard', icon: LayoutDashboard },
    { href: '#/unified', label: 'Unified View', icon: Users },
    { href: '#/identities', label: 'Identities', icon: Users },
    { href: '#/data', label: 'Data & Sync', icon: Database },
    { href: '#/status', label: 'System Status', icon: Radio },
    { href: '#/settings', label: 'Settings', icon: SettingsIcon }
]

export function Layout({ children }: { children: React.ReactNode }) {
    const currentPath = window.location.hash.split('?')[0]

    return (
        <div className="flex h-screen bg-background">
            {/* Sidebar */}
            <aside className="w-64 border-r bg-card">
                <div className="p-6">
                    <div className="flex items-center gap-2 mb-8">
                        <Activity className="w-6 h-6" />
                        <h1 className="text-xl font-bold">SlipStream</h1>
                    </div>
                    <nav className="space-y-2">
                        {navItems.map(item => {
                            const Icon = item.icon
                            const isActive = currentPath === item.href
                            return (
                                <a
                                    key={item.href}
                                    href={item.href}
                                    className={cn(
                                        'flex items-center gap-3 px-3 py-2 rounded-lg transition-colors',
                                        isActive
                                            ? 'bg-primary text-primary-foreground'
                                            : 'hover:bg-accent'
                                    )}
                                >
                                    <Icon className="w-5 h-5" />
                                    <span>{item.label}</span>
                                </a>
                            )
                        })}
                    </nav>
                </div>
            </aside>

            {/* Main content */}
            <main className="flex-1 overflow-auto">
                <motion.div
                    initial={{ opacity: 0, y: 20 }}
                    animate={{ opacity: 1, y: 0 }}
                    transition={{ duration: 0.3 }}
                    className="p-8"
                >
                    {children}
                </motion.div>
            </main>
        </div>
    )
}
```

### SearchableSelect Component (`src/components/SearchableSelect.tsx`)

```typescript
import { useState, useRef, useEffect } from 'react'
import { ChevronDown, Search } from 'lucide-react'

interface Option {
    value: string
    label: string
    subtext?: string
}

interface SearchableSelectProps {
    options: Option[]
    value: string
    onChange: (value: string) => void
    placeholder?: string
}

export function SearchableSelect({ options, value, onChange, placeholder }: SearchableSelectProps) {
    const [isOpen, setIsOpen] = useState(false)
    const [search, setSearch] = useState('')
    const ref = useRef<HTMLDivElement>(null)

    useEffect(() => {
        const handleClickOutside = (e: MouseEvent) => {
            if (ref.current && !ref.current.contains(e.target as Node)) {
                setIsOpen(false)
            }
        }
        document.addEventListener('mousedown', handleClickOutside)
        return () => document.removeEventListener('mousedown', handleClickOutside)
    }, [])

    const filtered = options.filter(opt =>
        opt.label.toLowerCase().includes(search.toLowerCase()) ||
        opt.value.toLowerCase().includes(search.toLowerCase())
    )

    const selected = options.find(opt => opt.value === value)

    return (
        <div ref={ref} className="relative">
            <button
                onClick={() => setIsOpen(!isOpen)}
                className="w-full px-4 py-2 text-left bg-background border rounded-lg flex items-center justify-between hover:bg-accent"
            >
                <span>{selected?.label || placeholder}</span>
                <ChevronDown className="w-4 h-4" />
            </button>

            {isOpen && (
                <div className="absolute z-50 w-full mt-2 bg-background border rounded-lg shadow-lg max-h-96 overflow-auto">
                    <div className="p-2 border-b sticky top-0 bg-background">
                        <div className="flex items-center gap-2 px-3 py-2 bg-accent rounded-md">
                            <Search className="w-4 h-4 text-muted-foreground" />
                            <input
                                type="text"
                                value={search}
                                onChange={(e) => setSearch(e.target.value)}
                                placeholder="Search..."
                                className="flex-1 bg-transparent outline-none text-sm"
                                autoFocus
                            />
                        </div>
                    </div>
                    <div className="p-2">
                        {filtered.map(opt => (
                            <button
                                key={opt.value}
                                onClick={() => {
                                    onChange(opt.value)
                                    setIsOpen(false)
                                    setSearch('')
                                }}
                                className="w-full px-3 py-2 text-left rounded-md hover:bg-accent"
                            >
                                <div className="font-medium">{opt.label}</div>
                                {opt.subtext && (
                                    <div className="text-xs text-muted-foreground">{opt.subtext}</div>
                                )}
                            </button>
                        ))}
                    </div>
                </div>
            )}
        </div>
    )
}
```

### API Configuration (`src/lib/api.ts`)

```typescript
export const API_BASE_URL =
    import.meta.env.VITE_API_BASE_URL || 'http://localhost:3042'
```

**Override via**:
```bash
VITE_API_BASE_URL=http://localhost:3043 npm run dev
```

### Utility Functions (`src/lib/utils.ts`)

```typescript
import { clsx, type ClassValue } from 'clsx'
import { twMerge } from 'tailwind-merge'

export function cn(...inputs: ClassValue[]) {
    return twMerge(clsx(inputs))
}
```

---

## API Specification

### Configuration Endpoints

#### GET /api/config
Fetch all API configurations for an organization.

**Query Parameters**:
- `org_id` (string, default: `myorg`): Organization identifier

**Response**:
```json
{
  "configs": [
    {
      "provider": "claude",
      "config": { "api_key": "sk-ant-..." },
      "configured": true,
      "updated_at": "2025-12-28T10:00:00.000Z"
    },
    {
      "provider": "github",
      "config": { "api_key": "ghp_..." },
      "configured": true,
      "updated_at": "2025-12-28T10:00:00.000Z"
    },
    {
      "provider": "jira",
      "config": {
        "api_key": "...",
        "metadata": { "email": "user@company.com", "domain": "company.atlassian.net" }
      },
      "configured": true,
      "updated_at": "2025-12-28T10:00:00.000Z"
    }
  ]
}
```

#### POST /api/config
Create or update an API configuration.

**Request Body**:
```json
{
  "org_id": "myorg",
  "provider": "claude",
  "api_key": "sk-ant-..."
}
```

**For Jira** (requires metadata):
```json
{
  "org_id": "myorg",
  "provider": "jira",
  "api_key": "...",
  "metadata": {
    "email": "user@company.com",
    "domain": "company.atlassian.net"
  }
}
```

**Response**:
```json
{ "success": true }
```

---

### Ingestion Endpoints

#### POST /api/claude/metrics/ingest
Ingest Claude usage metrics (batch).

**Request Body**:
```json
{
  "org_id": "myorg",
  "samples": [
    {
      "provider_user_id": "user@company.com",
      "bucket_start": "2025-12-28T00:00:00.000Z",
      "bucket_end": "2025-12-28T23:59:59.999Z",
      "repo_hint": "Apple_Terminal",
      "metrics": {
        "claude_code.session.count": 5,
        "claude_code.token.usage": 250000,
        "claude_code.cost.usage": 1500,
        "claude_code.active_time.total": 7200,
        "claude_code.lines_of_code.count": 1500
      }
    }
  ]
}
```

**Response**:
```json
{ "success": true, "count": 1 }
```

---

### Sync Endpoints

#### GET /api/sync/status
Get current sync status (for progress monitoring).

**Response**:
```json
{
  "active": true,
  "stage": "Fetching commits for example-org/frontend-app",
  "total_repos": 10,
  "processed_repos": 5,
  "current_repo": "example-org/frontend-app",
  "stats": {
    "commits": 1234,
    "prs": 56
  },
  "last_updated": "2025-12-28T10:05:30.000Z"
}
```

#### POST /api/sync/full
Trigger full GitHub sync (all repos, all commits/PRs).

**Request Body**:
```json
{ "org_id": "myorg" }
```

**Response**:
```json
{
  "repos": 10,
  "commits": 1234,
  "prs": 56
}
```

#### POST /api/sync
Generic sync endpoint (provider-agnostic).

**Claude Sync**:
```json
{
  "org_id": "myorg",
  "provider": "claude",
  "resource": "data",
  "workspace_id": "optional-workspace-id",
  "since": "2025-11-01T00:00:00.000Z"
}
```

**GitHub Sync**:
```json
{
  "org_id": "myorg",
  "provider": "github",
  "resource": "data",
  "since": "2025-11-01T00:00:00.000Z"
}
```

**Jira Sync**:
```json
{
  "org_id": "myorg",
  "provider": "jira",
  "resource": "data",
  "target_user": "user@company.com",
  "custom_jql": "assignee = currentUser() AND updated >= -30d"
}
```

**Response**:
```json
{ "success": true, "count": 100 }
```

---

### Analytics Endpoints

#### GET /api/developer/users
List all canonical users with aggregated Claude metrics.

**Query Parameters**:
- `org_id` (string, default: `myorg`)

**Response**:
```json
{
  "users": [
    {
      "user_id": "sarah.chen@company.com",
      "display_name": "[HO] Sarah Chen",
      "total_tokens": 5000000,
      "total_sessions": 45
    }
  ]
}
```

#### GET /api/developer/identity-suggestions
Get identity linking suggestions for manual review.

**Query Parameters**:
- `org_id` (string, default: `myorg`)

**Response**:
```json
{
  "suggestions": [
    {
      "canonical_user": "sarah.chen@company.com",
      "github": {
        "linked": {
          "provider_user_id": "sarahc-dev",
          "email": "sarah.chen@company.com",
          "confidence": 100
        },
        "suggestions": []
      },
      "jira": {
        "linked": {
          "provider_user_id": "sarah.chen",
          "email": "sarah.chen@company.com",
          "confidence": 100
        },
        "suggestions": []
      }
    }
  ]
}
```

#### POST /api/developer/identities
Manually create identity link.

**Request Body**:
```json
{
  "org_id": "myorg",
  "provider": "github",
  "user_id": "sarah.chen@company.com",
  "provider_user_id": "sarahc-dev",
  "confidence": 100
}
```

**Response**:
```json
{ "success": true }
```

---

### Unified View Endpoint

#### GET /api/unified/table
**Core endpoint**: Correlates all data sources via identity mapping.

**Query Parameters**:
- `org_id` (string, default: `myorg`)

**Response**:
```json
{
  "rows": [
    {
      "user": "sarah.chen@company.com",
      "github_id": "sarahc-dev",
      "jira_id": "sarah.chen",
      "claude_cost": 15000,
      "claude_loc": 45000,
      "github_loc": 52000,
      "prs_created": 23,
      "jira_tickets": 12
    }
  ]
}
```

**Algorithm**:
1. Auto-link identities (ensure fresh map)
2. Build identity map: `"provider:provider_user_id"` → canonical email
3. Aggregate Claude cost & LOC from `ai_metric_samples`
4. Aggregate GitHub LOC & PRs via identity map or direct email match
5. Aggregate Jira tickets via identity map or direct email match
6. Return correlated rows

---

## Service Layer Implementation

### Identity Service (`src/services/identity.ts`)

#### autoLinkIdentities(orgId: string)

**Purpose**: Auto-link users across providers based on exact email matches.

**Algorithm**:
```
1. Extract canonical users from ai_metric_samples.user_id (Claude emails)
2. Fetch all provider_users with non-null emails
3. For each provider user:
   a. Normalize email (lowercase, trim)
   b. Check if normalized email matches any canonical user
   c. If match found:
      - Check if link already exists in identities table
      - Check if link is explicitly rejected in identity_rejections
      - If neither, insert link with confidence=100
      - For GitHub: use username if available, else provider_user_id
      - For Jira/Claude: use provider_user_id
4. Return count of new links created
```

**Implementation**:
```typescript
export async function autoLinkIdentities(orgId: string): Promise<number> {
    // 1. Get canonical users
    const claudeUsers = await db
        .selectFrom('ai_metric_samples')
        .select('user_id')
        .where('org_id', '=', orgId)
        .distinct()
        .execute()

    const canonicalSet = new Set(
        claudeUsers.map(u => normalizeEmail(u.user_id)).filter(Boolean)
    )

    // 2. Get provider users with emails
    const providerUsers = await db
        .selectFrom('provider_users')
        .selectAll()
        .where('org_id', '=', orgId)
        .where('email', 'is not', null)
        .execute()

    let newLinks = 0

    // 3. Match and link
    for (const pu of providerUsers) {
        const normalizedEmail = normalizeEmail(pu.email)
        if (!normalizedEmail || !canonicalSet.has(normalizedEmail)) continue

        // Determine provider_user_id to use
        const linkId = pu.provider === 'github' && pu.username
            ? pu.username
            : pu.provider_user_id

        // Check if link exists
        const existing = await db
            .selectFrom('identities')
            .selectAll()
            .where('user_id', '=', normalizedEmail)
            .where('provider', '=', pu.provider)
            .where('provider_user_id', '=', linkId)
            .executeTakeFirst()

        if (existing) continue

        // Check if rejected
        const rejected = await db
            .selectFrom('identity_rejections')
            .selectAll()
            .where('user_id', '=', normalizedEmail)
            .where('provider', '=', pu.provider)
            .where('provider_user_id', '=', linkId)
            .executeTakeFirst()

        if (rejected) continue

        // Create link
        await db
            .insertInto('identities')
            .values({
                user_id: normalizedEmail,
                provider: pu.provider,
                provider_user_id: linkId,
                confidence: 100,
                created_at: new Date().toISOString()
            })
            .execute()

        newLinks++
    }

    return newLinks
}
```

---

### Claude Service (`src/services/claude.ts`)

#### syncClaudeMetrics(orgId, apiKey, workspaceId?, since?)

**Purpose**: Fetch Claude usage data from API and store in database.

**API Endpoint**: `https://api.anthropic.com/v1/organizations/workspaces/{workspaceId}/usage/daily`

**Date Range**: `since` (default: 30 days ago) for 32 days total

**Rate Limiting**: Batch fetches 5 days in parallel

**Implementation**:
```typescript
export async function syncClaudeMetrics(
    orgId: string,
    apiKey: string,
    workspaceId?: string,
    since?: string
): Promise<{ count: number }> {
    const startDate = since ? new Date(since) : new Date(Date.now() - 30 * 24 * 60 * 60 * 1000)
    const dates: string[] = []

    // Generate 32 days of dates
    for (let i = 0; i < 32; i++) {
        const date = new Date(startDate)
        date.setDate(date.getDate() + i)
        dates.push(date.toISOString().split('T')[0])
    }

    // Batch fetch 5 days in parallel
    const batches: string[][] = []
    for (let i = 0; i < dates.length; i += 5) {
        batches.push(dates.slice(i, i + 5))
    }

    let count = 0

    for (const batch of batches) {
        const promises = batch.map(async (date) => {
            const url = workspaceId
                ? `https://api.anthropic.com/v1/organizations/workspaces/${workspaceId}/usage/daily?starting_at=${date}`
                : `https://api.anthropic.com/v1/organizations/usage/daily?starting_at=${date}`

            const response = await fetch(url, {
                headers: {
                    'x-api-key': apiKey,
                    'anthropic-version': '2023-06-01'
                }
            })

            if (!response.ok) {
                throw new Error(`Claude API error: ${response.status}`)
            }

            return response.json()
        })

        const results = await Promise.all(promises)

        // Process results
        for (const data of results) {
            if (!data.data || data.data.length === 0) continue

            for (const sample of data.data) {
                const userEmail = sample.user_email || sample.user_id

                // Delete overlapping data
                await db
                    .deleteFrom('ai_metric_samples')
                    .where('org_id', '=', orgId)
                    .where('user_id', '=', userEmail)
                    .where('bucket_start', '>=', sample.starting_at)
                    .execute()

                // Insert new sample
                await db
                    .insertInto('ai_metric_samples')
                    .values({
                        org_id: orgId,
                        user_id: userEmail,
                        bucket_start: sample.starting_at,
                        bucket_end: sample.ending_at || sample.starting_at,
                        repo_hint: null,
                        metrics_json: JSON.stringify({
                            'claude_code.session.count': sample.session_count || 0,
                            'claude_code.token.usage': sample.token_usage || 0,
                            'claude_code.cost.usage': sample.cost_usd_cents || 0,
                            'claude_code.active_time.total': sample.active_time_seconds || 0,
                            'claude_code.lines_of_code.count': sample.lines_of_code || 0,
                            'raw_api_data': sample
                        })
                    })
                    .execute()

                count++
            }
        }
    }

    return { count }
}
```

---

### GitHub Service (`src/services/github.ts`)

#### syncGithubActivity(orgId, token, since?)

**Purpose**: Sync GitHub activity (commits, PRs) for all accessible repos.

**API Base**: `https://api.github.com`

**Rate Limiting**: Tracks remaining quota from response headers.

**Implementation**:
```typescript
let rateLimitRemaining = 5000
let rateLimitReset = Date.now()

async function checkRateLimit() {
    if (rateLimitRemaining < 10 && Date.now() < rateLimitReset) {
        const waitMs = rateLimitReset - Date.now()
        throw new Error(`GitHub rate limit exhausted. Reset in ${Math.ceil(waitMs / 1000)}s`)
    }
}

function updateRateLimit(headers: Headers) {
    const remaining = headers.get('x-ratelimit-remaining')
    const reset = headers.get('x-ratelimit-reset')

    if (remaining) rateLimitRemaining = parseInt(remaining)
    if (reset) rateLimitReset = parseInt(reset) * 1000
}

export async function syncGithubActivity(
    orgId: string,
    token: string,
    since?: string
): Promise<{ repos: number, commits: number, prs: number }> {
    await checkRateLimit()

    // Fetch repos
    const reposResponse = await fetch('https://api.github.com/user/repos?per_page=100', {
        headers: {
            'Authorization': `Bearer ${token}`,
            'Accept': 'application/vnd.github.v3+json'
        }
    })
    updateRateLimit(reposResponse.headers)

    const repos = await reposResponse.json()
    let commitCount = 0
    let prCount = 0

    for (const repo of repos) {
        // Filter by pushed_at if since provided
        if (since && new Date(repo.pushed_at) < new Date(since)) {
            continue
        }

        // Insert repo
        await db
            .insertInto('repos')
            .values({
                id: String(repo.id),
                org_id: orgId,
                name: repo.name,
                full_name: repo.full_name,
                url: repo.html_url
            })
            .onConflict(oc => oc.column('id').doUpdateSet({
                name: repo.name,
                full_name: repo.full_name,
                url: repo.html_url
            }))
            .execute()

        // Fetch commits
        const commitsUrl = since
            ? `https://api.github.com/repos/${repo.full_name}/commits?since=${since}`
            : `https://api.github.com/repos/${repo.full_name}/commits`

        await checkRateLimit()
        const commitsResponse = await fetch(commitsUrl, {
            headers: {
                'Authorization': `Bearer ${token}`,
                'Accept': 'application/vnd.github.v3+json'
            }
        })
        updateRateLimit(commitsResponse.headers)

        const commits = await commitsResponse.json()

        for (const commit of commits) {
            // Fetch commit details for line stats
            await checkRateLimit()
            const detailsResponse = await fetch(
                `https://api.github.com/repos/${repo.full_name}/commits/${commit.sha}`,
                {
                    headers: {
                        'Authorization': `Bearer ${token}`,
                        'Accept': 'application/vnd.github.v3+json'
                    }
                }
            )
            updateRateLimit(detailsResponse.headers)

            const details = await detailsResponse.json()

            await db
                .insertInto('commits')
                .values({
                    sha: commit.sha,
                    repo_id: String(repo.id),
                    author_user_id: commit.author?.login || null,
                    committer_email: normalizeEmail(commit.commit?.committer?.email),
                    created_at: commit.commit?.committer?.date || new Date().toISOString(),
                    files_changed: details.files?.length || 0,
                    lines_added: details.stats?.additions || 0,
                    lines_deleted: details.stats?.deletions || 0,
                    message: commit.commit?.message || '',
                    branch: null,
                    org_id: orgId
                })
                .onConflict(oc => oc.column('sha').doNothing())
                .execute()

            commitCount++
        }

        // Fetch PRs
        const prsUrl = `https://api.github.com/repos/${repo.full_name}/pulls?state=all&per_page=100`

        await checkRateLimit()
        const prsResponse = await fetch(prsUrl, {
            headers: {
                'Authorization': `Bearer ${token}`,
                'Accept': 'application/vnd.github.v3+json'
            }
        })
        updateRateLimit(prsResponse.headers)

        const prs = await prsResponse.json()

        for (const pr of prs) {
            if (since && new Date(pr.created_at) < new Date(since)) {
                continue
            }

            await db
                .insertInto('pull_requests')
                .values({
                    id: String(pr.id),
                    repo_id: String(repo.id),
                    number: pr.number,
                    author_user_id: pr.user?.login || null,
                    author_email: normalizeEmail(pr.user?.email),
                    opened_at: pr.created_at,
                    merged_at: pr.merged_at || null,
                    title: pr.title,
                    body: pr.body || '',
                    base_branch: pr.base?.ref || 'main',
                    head_branch: pr.head?.ref || '',
                    status: pr.state,
                    updated_at: pr.updated_at,
                    org_id: orgId
                })
                .onConflict(oc => oc.column('id').doNothing())
                .execute()

            prCount++
        }
    }

    return { repos: repos.length, commits: commitCount, prs: prCount }
}
```

---

### Jira Service (`src/services/jira.ts`)

#### syncJiraTickets(orgId, email, token, domain)

**Purpose**: Sync all Jira issues (updated in last 365 days).

**API Endpoint**: `https://{domain}/rest/api/3/search/jql`

**JQL Query**: `updated >= -365d ORDER BY updated DESC`

**Pagination**: 1000 results per request

**Implementation**:
```typescript
export async function syncJiraTickets(
    orgId: string,
    email: string,
    token: string,
    domain: string
): Promise<{ issues: number }> {
    const jql = 'updated >= -365d ORDER BY updated DESC'
    const url = `https://${domain}/rest/api/3/search/jql`

    let startAt = 0
    let total = 0
    let issueCount = 0

    do {
        const response = await fetch(`${url}?jql=${encodeURIComponent(jql)}&startAt=${startAt}&maxResults=1000`, {
            headers: {
                'Authorization': `Basic ${Buffer.from(`${email}:${token}`).toString('base64')}`,
                'Accept': 'application/json'
            }
        })

        if (!response.ok) {
            throw new Error(`Jira API error: ${response.status}`)
        }

        const data = await response.json()
        total = data.total

        for (const issue of data.issues) {
            // Extract story points (customfield_10016 is standard)
            const storyPoints = issue.fields.customfield_10016 || null

            await db
                .insertInto('jira_issues')
                .values({
                    key: issue.key,
                    project_key: issue.fields.project.key,
                    sprint_id: null,  // Extract from sprint field if needed
                    type: issue.fields.issuetype.name,
                    status: issue.fields.status.name,
                    status_category: issue.fields.status.statusCategory.name,
                    assignee_user_id: issue.fields.assignee?.accountId || null,
                    assignee_email: normalizeEmail(issue.fields.assignee?.emailAddress),
                    points: storyPoints,
                    parent_key: issue.fields.parent?.key || null,
                    product: null,  // Extract from custom field if needed
                    updated_at: issue.fields.updated,
                    status_change_at: issue.fields.statuscategorychangedate || null,
                    labels: JSON.stringify(issue.fields.labels || []),
                    components: JSON.stringify(issue.fields.components || []),
                    description_length: issue.fields.description?.length || 0,
                    ac_present: issue.fields.description?.includes('Acceptance Criteria') ? 1 : 0,
                    org_id: orgId
                })
                .onConflict(oc => oc.column('key').doUpdateSet({
                    status: issue.fields.status.name,
                    status_category: issue.fields.status.statusCategory.name,
                    updated_at: issue.fields.updated
                }))
                .execute()

            issueCount++
        }

        startAt += data.issues.length
    } while (startAt < total)

    return { issues: issueCount }
}
```

---

### Scheduler Service (`src/services/scheduler.ts`)

**Purpose**: Run daily GitHub sync at midnight.

**Cron Pattern**: `0 0 * * *` (midnight every day)

**Implementation**:
```typescript
import cron from 'node-cron'
import { syncFullGithubActivity } from './github'
import { db } from '../db'

class SchedulerService {
    private job: cron.ScheduledTask | null = null

    start() {
        console.log('[Scheduler] Initializing...')

        // Run at midnight every day
        this.job = cron.schedule('0 0 * * *', async () => {
            console.log('[Scheduler] Starting daily GitHub sync...')

            try {
                const orgId = process.env.ORG_ID || 'myorg'

                // Fetch GitHub token
                const config = await db
                    .selectFrom('api_configs')
                    .select('api_key')
                    .where('org_id', '=', orgId)
                    .where('provider', '=', 'github')
                    .executeTakeFirst()

                if (!config) {
                    console.error('[Scheduler] No GitHub token configured')
                    return
                }

                const result = await syncFullGithubActivity(orgId, config.api_key)
                console.log('[Scheduler] Sync complete:', result)
            } catch (error) {
                console.error('[Scheduler] Sync failed:', error)
            }
        })

        console.log('[Scheduler] Scheduling daily sync job for midnight.')
    }

    stop() {
        if (this.job) {
            this.job.stop()
            console.log('[Scheduler] Stopped.')
        }
    }
}

export const scheduler = new SchedulerService()
```


---

## Key Algorithms

### 1. Identity Auto-Linking Algorithm

**Input**: Organization ID
**Output**: Count of new identity links created

**Steps**:
1. Extract canonical users from `ai_metric_samples.user_id` (Claude emails)
2. Create set of normalized canonical emails (lowercase, trimmed)
3. Fetch all `provider_users` with non-null emails
4. For each provider user:
   - Normalize email
   - Check if it matches any canonical email
   - If match found:
     - Check if link already exists in `identities` table
     - Check if link is in `identity_rejections` table
     - If neither, create new identity link with confidence=100
     - For GitHub: use `username` if available, else `provider_user_id`
     - For Jira/Claude: use `provider_user_id`
5. Return count of new links

**Key Characteristics**:
- Only auto-links on 100% confidence (exact email match)
- Idempotent: runs before each unified query
- Respects explicit rejections
- Case-insensitive matching

---

### 2. Unified View Correlation Algorithm

**Input**: Organization ID
**Output**: Array of correlated user metrics

**Steps**:
1. **Auto-link identities** (ensure fresh mapping)
2. **Build identity map** (`"provider:provider_user_id"` → canonical email)
   - Query `identities` table
   - Create lookup map for fast resolution
3. **Aggregate Claude metrics** by canonical email
   - Query `ai_metric_samples`
   - Sum cost and LOC from `metrics_json`
4. **Aggregate GitHub metrics** via two-pass approach:
   - First pass: Use identity map to resolve `author_user_id` to canonical email
   - Second pass: Direct email match on `committer_email`
   - Sum lines_added + lines_deleted
   - Count pull requests
5. **Aggregate Jira metrics** via two-pass approach:
   - First pass: Use identity map to resolve `assignee_user_id` to canonical email
   - Second pass: Direct email match on `assignee_email`
   - Count issues
6. **Merge all metrics** by canonical email
7. **Return rows** with:
   - user (canonical email)
   - github_id, jira_id (provider IDs from identity map)
   - claude_cost, claude_loc
   - github_loc, prs_created
   - jira_tickets

**Key Characteristics**:
- Dual resolution: identity map + email fallback
- Handles partial identity linking gracefully
- Single query per data source for performance
- Returns NULL for missing provider links

---

### 3. GitHub Sync Process

**Input**: Organization ID, GitHub token, optional since date
**Output**: Counts of synced repos, commits, PRs

**Steps**:
1. **Rate limit check** (throws if exhausted)
2. **Fetch repositories**
   - GET `/user/repos?per_page=100`
   - Handle pagination if needed
   - Update rate limit from headers
3. **For each repository**:
   - **Filter** by `pushed_at >= since` (if since provided)
   - **Insert/update** repo in database
   - **Fetch commits**:
     - GET `/repos/{owner}/{repo}/commits?since={since}`
     - For each commit:
       - Fetch commit details: GET `/repos/{owner}/{repo}/commits/{sha}`
       - Extract line stats (additions/deletions)
       - Insert into `commits` table
   - **Fetch pull requests**:
     - GET `/repos/{owner}/{repo}/pulls?state=all&per_page=100`
     - Filter by `created_at >= since` (if provided)
     - Insert into `pull_requests` table
4. **Return** total counts

**Key Characteristics**:
- Respects GitHub rate limits (5000 requests/hour)
- Normalizes emails (lowercase, trim)
- Upsert strategy prevents duplicates
- Shallow sync (date-filtered) vs deep sync (full history)

---


## Configuration & Environment

### Environment Variables

**Backend** (`.env`):
```bash
# Claude API
CLAUDE_API_KEY=sk-ant-api03-...

# GitHub
GITHUB_TOKEN=ghp_...

# Jira
JIRA_API_TOKEN=...
JIRA_EMAIL=user@company.com
JIRA_DOMAIN=company.atlassian.net

# Database
DB_PATH=slipstream.db

# Server
PORT=3042
ORG_ID=myorg
```

**Frontend** (`.env` in `frontend/`):
```bash
VITE_API_BASE_URL=http://localhost:3042
```

**Override at runtime**:
```bash
# Backend
PORT=3043 npm run start:server

# Frontend
VITE_API_BASE_URL=http://localhost:3043 npm run dev
```

### Initialization Flow

**Server startup** (`src/server.ts`):
1. Load `.env` via `dotenv/config`
2. Run `migrateToLatest()` (create tables, add columns)
3. Run `seedEnvConfigs()` (insert API keys from .env)
4. Start `scheduler` (daily midnight sync)
5. Listen on port

**seedEnvConfigs()** implementation:
```typescript
async function seedEnvConfigs() {
    const orgId = process.env.ORG_ID || 'myorg'

    // Claude
    if (process.env.CLAUDE_API_KEY) {
        await db.insertInto('api_configs')
            .values({
                org_id: orgId,
                provider: 'claude',
                api_key: process.env.CLAUDE_API_KEY,
                metadata: null,
                updated_at: new Date().toISOString()
            })
            .onConflict(oc => oc.columns(['org_id', 'provider']).doNothing())
            .execute()
    }

    // GitHub
    if (process.env.GITHUB_TOKEN) {
        await db.insertInto('api_configs')
            .values({
                org_id: orgId,
                provider: 'github',
                api_key: process.env.GITHUB_TOKEN,
                metadata: null,
                updated_at: new Date().toISOString()
            })
            .onConflict(oc => oc.columns(['org_id', 'provider']).doNothing())
            .execute()
    }

    // Jira
    if (process.env.JIRA_API_TOKEN && process.env.JIRA_EMAIL && process.env.JIRA_DOMAIN) {
        await db.insertInto('api_configs')
            .values({
                org_id: orgId,
                provider: 'jira',
                api_key: process.env.JIRA_API_TOKEN,
                metadata: JSON.stringify({
                    email: process.env.JIRA_EMAIL,
                    domain: process.env.JIRA_DOMAIN
                }),
                updated_at: new Date().toISOString()
            })
            .onConflict(oc => oc.columns(['org_id', 'provider']).doNothing())
            .execute()
    }
}
```

---

## Development & Deployment

### Local Development

**Backend Setup**:
```bash
npm install
npx tsx src/server.ts
# Server runs on port 3042 (default)
```

**Frontend Setup**:
```bash
cd frontend
npm install
npm run dev
# Frontend runs on port 5180
```

**Full Stack (Concurrent)**:
```bash
npm run start:dev
# Backend: PORT=3043
# Frontend: port 5180 with VITE_API_BASE_URL=http://localhost:3043
```

### Production Build

**Backend**:
```bash
npm run build:server  # Compiles TypeScript to dist/
PORT=3043 npm run start:server
```

**Frontend**:
```bash
cd frontend
npm run build  # Outputs to dist/
# Serve with Nginx or other static server
```

### Docker Deployment

**docker-compose.yml**:
```yaml
version: '3.8'

services:
  backend:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "3043:3043"
    environment:
      - PORT=3043
      - DB_PATH=/data/slipstream.db
      - CLAUDE_API_KEY=${CLAUDE_API_KEY}
      - GITHUB_TOKEN=${GITHUB_TOKEN}
      - JIRA_API_TOKEN=${JIRA_API_TOKEN}
      - JIRA_EMAIL=${JIRA_EMAIL}
      - JIRA_DOMAIN=${JIRA_DOMAIN}
    volumes:
      - ./data:/data

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports:
      - "80:80"
    depends_on:
      - backend
```

**Backend Dockerfile**:
```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .
RUN npm run build:server

EXPOSE 3043

CMD ["node", "dist/server.js"]
```

**Frontend Dockerfile**:
```dockerfile
FROM node:18-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

FROM nginx:alpine

COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

**nginx.conf**:
```nginx
server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

### GitHub Sync Script (Standalone)

**Usage**:
```bash
ORG_ID=myorg GITHUB_SINCE=2025-01-01T00:00:00.000Z npm run sync:github
```

**Implementation** (`scripts/sync-github.ts`):
```typescript
import 'dotenv/config'
import { db, migrateToLatest } from '../src/db'
import { syncGithubActivity } from '../src/services/github'

async function main() {
    await migrateToLatest()

    const orgId = process.env.ORG_ID || 'myorg'
    const since = process.env.GITHUB_SINCE || undefined

    const config = await db
        .selectFrom('api_configs')
        .select('api_key')
        .where('org_id', '=', orgId)
        .where('provider', '=', 'github')
        .executeTakeFirst()

    if (!config) {
        console.error('No GitHub token configured')
        process.exit(1)
    }

    console.log(`[Sync] Starting GitHub sync for ${orgId}...`)
    const result = await syncGithubActivity(orgId, config.api_key, since)
    console.log('[Sync] Complete:', result)
}

main()
```

---

## Additional Implementation Notes

### Multi-Org Support

All tables include `org_id` column for future multi-tenancy:
- Currently hardcoded to `myorg` in examples
- Can be extended for per-org API keys and data isolation
- Unique constraints include `org_id` to prevent cross-org conflicts

### Type Safety with Kysely

Kysely provides compile-time type safety:
- Query result types inferred from `schema.ts`
- Column names validated at compile time
- JOIN operations type-checked
- No runtime overhead

**Example**:
```typescript
// This will fail TypeScript compilation if 'user_id' doesn't exist:
const users = await db
    .selectFrom('ai_metric_samples')
    .select('user_id')  // ← Type-safe!
    .execute()

// Result type: { user_id: string }[]
```

### Error Handling

**Sync operations**: Log errors, continue with other repos/users
**Migrations**: try-catch around ALTER TABLE for idempotency
**API routes**: Standardized error responses with stack traces in dev

**Example** (sync error handling):
```typescript
for (const repo of repos) {
    try {
        await syncRepo(repo)
    } catch (error) {
        console.error(`Failed to sync ${repo.name}:`, error)
        // Continue with next repo
    }
}
```

### Rate Limiting

**GitHub**: In-memory tracking of remaining quota and reset time
**Claude**: No explicit rate limiting (batches 5 days in parallel)
**Jira**: Handled by API (no explicit implementation)

### Click-to-Scroll Feature

**Location**: `frontend/src/pages/Dashboard.tsx`

**Implementation**:
```typescript
<ComposedChart
    data={filledStream}
    onClick={(e: any) => {
        if (e && e.activeLabel) {
            const dateId = `activity-${e.activeLabel}`
            const element = document.getElementById(dateId)
            if (element) {
                element.scrollIntoView({ behavior: 'smooth', block: 'center' })
                element.classList.add('ring-2', 'ring-primary', 'ring-offset-2')
                setTimeout(() => {
                    element.classList.remove('ring-2', 'ring-primary', 'ring-offset-2')
                }, 2000)
            }
        }
    }}
>
```

**Activity Stream Items**:
```typescript
<div
    key={i}
    id={`activity-${item.window.window_start}`}
    className="p-4 hover:bg-accent/5 transition-colors"
>
```

### URL State Management

**Location**: `frontend/src/pages/Dashboard.tsx`

**Pattern**: Store selected user in query param `?user=email`

**Implementation**:
```typescript
// Initialize from URL on mount
useEffect(() => {
    const params = new URLSearchParams(window.location.search)
    const userFromUrl = params.get('user')
    if (userFromUrl) {
        setSelectedUser(decodeURIComponent(userFromUrl))
    }
}, [])

// Update URL when user changes
const handleUserChange = (userId: string) => {
    setSelectedUser(userId)
    if (userId) {
        const params = new URLSearchParams()
        params.set('user', encodeURIComponent(userId))
        const hashPath = window.location.hash.split('?')[0]
        window.history.pushState({}, '', `${window.location.pathname}${hashPath}?${params.toString()}`)
    } else {
        const hashPath = window.location.hash.split('?')[0]
        window.history.pushState({}, '', `${window.location.pathname}${hashPath}`)
    }
}
```

---

## Critical Deep Dive: Data Sync & Identity Matching

This section provides exhaustive detail on the most complex aspects of SlipStream: pulling data from GitHub and Jira, then matching users across all three platforms (Claude, GitHub, Jira). These systems were refined through extensive iteration and edge case handling.

### Problem Statement

The core challenge: **How do you correlate a developer's AI usage (Claude) with their code output (GitHub) and project management activity (Jira) when each platform uses different identifiers?**

- **Claude**: Uses email addresses as user IDs
- **GitHub**: Uses usernames (logins) as primary identifiers, emails are optional/private
- **Jira**: Uses accountIds (UUIDs) as primary identifiers, emails in user objects

**Solution**: Three-layer identity resolution system with email as the canonical identifier.

---

### Part 1: GitHub Data Sync - The Complete Picture

#### GitHub API Challenges

1. **Rate Limiting**: 5,000 requests/hour for authenticated requests
2. **Pagination**: Most endpoints return 30 items by default, max 100
3. **Line Stats Not in List View**: Must fetch individual commit details to get additions/deletions
4. **Private Emails**: Users can hide email addresses from commit metadata
5. **Multiple Email Addresses**: Same user might commit with different emails

#### GitHub API Endpoints Used

**1. List User Repositories**
```http
GET https://api.github.com/user/repos
Authorization: Bearer {token}
Accept: application/vnd.github.v3+json

Query Parameters:
  - per_page: 100 (max)
  - page: 1, 2, 3... (pagination)
  - type: all (default includes public, private, and member repos)
```

**Response Shape**:
```json
[
  {
    "id": 123456789,
    "name": "frontend-app",
    "full_name": "myorg/frontend-app",
    "html_url": "https://github.com/myorg/frontend-app",
    "pushed_at": "2025-12-20T10:30:00Z",
    "private": false
  }
]
```

**2. List Commits for Repository**
```http
GET https://api.github.com/repos/{owner}/{repo}/commits
Authorization: Bearer {token}
Accept: application/vnd.github.v3+json

Query Parameters:
  - since: 2025-11-01T00:00:00Z (ISO 8601 timestamp)
  - per_page: 100
  - page: 1, 2, 3...
```

**Response Shape** (important fields):
```json
[
  {
    "sha": "abc123def456",
    "commit": {
      "author": {
        "name": "Sarah Chen",
        "email": "sarah.chen@company.com",
        "date": "2025-12-20T14:23:45Z"
      },
      "committer": {
        "name": "Sarah Chen",
        "email": "sarah.chen@company.com",
        "date": "2025-12-20T14:23:45Z"
      },
      "message": "feat: Add authentication flow"
    },
    "author": {
      "login": "sarahc-dev",
      "id": 987654,
      "avatar_url": "https://avatars.githubusercontent.com/u/987654"
    }
  }
]
```

**Key Observation**: The list view does NOT include line stats (additions/deletions).

**3. Get Individual Commit Details** (for line stats)
```http
GET https://api.github.com/repos/{owner}/{repo}/commits/{sha}
Authorization: Bearer {token}
Accept: application/vnd.github.v3+json
```

**Response Shape** (important fields):
```json
{
  "sha": "abc123def456",
  "stats": {
    "total": 150,
    "additions": 120,
    "deletions": 30
  },
  "files": [
    {
      "filename": "src/auth.ts",
      "status": "modified",
      "additions": 50,
      "deletions": 10,
      "changes": 60
    }
  ]
}
```

**Critical**: This is the ONLY way to get line-level statistics. Must make one request per commit.

**4. List Pull Requests**
```http
GET https://api.github.com/repos/{owner}/{repo}/pulls
Authorization: Bearer {token}
Accept: application/vnd.github.v3+json

Query Parameters:
  - state: all (open, closed, merged)
  - per_page: 100
  - sort: created
  - direction: desc
```

**Response Shape**:
```json
[
  {
    "id": 111222333,
    "number": 42,
    "state": "closed",
    "title": "Add authentication flow",
    "body": "This PR adds OAuth2 authentication...",
    "user": {
      "login": "sarahc-dev",
      "id": 987654,
      "email": "sarah.chen@company.com"  // Optional, often null
    },
    "created_at": "2025-12-15T10:00:00Z",
    "updated_at": "2025-12-20T14:00:00Z",
    "merged_at": "2025-12-20T14:00:00Z",  // null if not merged
    "head": {
      "ref": "feature/auth",
      "sha": "abc123"
    },
    "base": {
      "ref": "main",
      "sha": "def456"
    }
  }
]
```

#### GitHub Sync Implementation Strategy

**Shallow Sync** (incremental, daily updates):
```typescript
export async function syncGithubActivity(
    orgId: string,
    token: string,
    since?: string  // ISO timestamp like "2025-11-01T00:00:00Z"
): Promise<{ repos: number, commits: number, prs: number }> {
    // 1. Check rate limit before starting
    await checkRateLimit()

    // 2. Fetch all accessible repositories
    const repos = await fetchAllRepos(token)

    let commitCount = 0
    let prCount = 0

    // 3. For each repository
    for (const repo of repos) {
        // OPTIMIZATION: Skip repos not updated since 'since' date
        if (since && new Date(repo.pushed_at) < new Date(since)) {
            continue
        }

        // 4. Upsert repository metadata
        await db.insertInto('repos')
            .values({
                id: String(repo.id),
                org_id: orgId,
                name: repo.name,
                full_name: repo.full_name,
                url: repo.html_url
            })
            .onConflict(oc => oc.column('id').doUpdateSet({
                name: repo.name,
                full_name: repo.full_name,
                url: repo.html_url
            }))
            .execute()

        // 5. Fetch commits (with 'since' filter)
        const commits = await fetchCommits(token, repo.full_name, since)

        // 6. For each commit, fetch details and store
        for (const commit of commits) {
            await checkRateLimit()

            // CRITICAL: Fetch commit details for line stats
            const details = await fetchCommitDetails(token, repo.full_name, commit.sha)

            // Extract author information
            const authorLogin = commit.author?.login || null
            const committerEmail = normalizeEmail(commit.commit?.committer?.email)

            await db.insertInto('commits')
                .values({
                    sha: commit.sha,
                    repo_id: String(repo.id),
                    author_user_id: authorLogin,  // GitHub username
                    committer_email: committerEmail,  // Email for matching
                    created_at: commit.commit?.committer?.date || new Date().toISOString(),
                    files_changed: details.files?.length || 0,
                    lines_added: details.stats?.additions || 0,
                    lines_deleted: details.stats?.deletions || 0,
                    message: commit.commit?.message || '',
                    branch: null,  // Not available in this endpoint
                    org_id: orgId
                })
                .onConflict(oc => oc.column('sha').doNothing())  // Idempotent
                .execute()

            commitCount++
        }

        // 7. Fetch pull requests (filter by created_at)
        const prs = await fetchPRs(token, repo.full_name, since)

        for (const pr of prs) {
            if (since && new Date(pr.created_at) < new Date(since)) {
                continue
            }

            await db.insertInto('pull_requests')
                .values({
                    id: String(pr.id),
                    repo_id: String(repo.id),
                    number: pr.number,
                    author_user_id: pr.user?.login || null,
                    author_email: normalizeEmail(pr.user?.email),
                    opened_at: pr.created_at,
                    merged_at: pr.merged_at || null,
                    title: pr.title,
                    body: pr.body || '',
                    base_branch: pr.base?.ref || 'main',
                    head_branch: pr.head?.ref || '',
                    status: pr.state,
                    updated_at: pr.updated_at,
                    org_id: orgId
                })
                .onConflict(oc => oc.column('id').doNothing())
                .execute()

            prCount++
        }
    }

    return { repos: repos.length, commits: commitCount, prs: prCount }
}
```

#### Rate Limit Tracking Implementation

```typescript
// Global state (in-memory)
let rateLimitRemaining = 5000
let rateLimitReset = Date.now()

async function checkRateLimit(): Promise<void> {
    if (rateLimitRemaining < 10 && Date.now() < rateLimitReset) {
        const waitMs = rateLimitReset - Date.now()
        const waitSec = Math.ceil(waitMs / 1000)
        throw new Error(
            `GitHub rate limit exhausted (${rateLimitRemaining} remaining). ` +
            `Reset in ${waitSec} seconds at ${new Date(rateLimitReset).toISOString()}`
        )
    }
}

function updateRateLimit(headers: Headers): void {
    const remaining = headers.get('x-ratelimit-remaining')
    const reset = headers.get('x-ratelimit-reset')

    if (remaining) {
        rateLimitRemaining = parseInt(remaining, 10)
    }

    if (reset) {
        rateLimitReset = parseInt(reset, 10) * 1000  // Convert Unix timestamp to milliseconds
    }

    // Log when getting low
    if (rateLimitRemaining < 100) {
        console.warn(
            `[GitHub] Rate limit low: ${rateLimitRemaining} remaining, ` +
            `resets at ${new Date(rateLimitReset).toISOString()}`
        )
    }
}

// Use in every fetch
async function githubFetch(url: string, token: string): Promise<Response> {
    await checkRateLimit()

    const response = await fetch(url, {
        headers: {
            'Authorization': `Bearer ${token}`,
            'Accept': 'application/vnd.github.v3+json'
        }
    })

    updateRateLimit(response.headers)

    if (!response.ok) {
        throw new Error(`GitHub API error: ${response.status} ${response.statusText}`)
    }

    return response
}
```

#### Email Extraction and Normalization

**Critical Pattern**: GitHub provides emails in TWO places, and we need BOTH:

1. **commit.commit.committer.email**: Raw Git metadata (most reliable)
2. **commit.author.login**: GitHub username (for identity linking)

**Why Both?**
- `login` links to GitHub profile (needed for identity map)
- `committer.email` may not match GitHub account email (configured locally)
- We store BOTH to enable dual-path resolution

**Normalization Function**:
```typescript
function normalizeEmail(email?: string | null): string | null {
    if (!email) return null

    const trimmed = email.trim()
    if (trimmed.length === 0) return null

    return trimmed.toLowerCase()
}
```

**Applied Everywhere**:
- Claude user IDs (already emails)
- GitHub committer emails
- GitHub PR author emails (when available)
- Jira assignee emails
- provider_users.email column

---

### Part 2: Jira Data Sync - The Complete Picture

#### Jira API Challenges

1. **Authentication**: Basic Auth with email + API token (not OAuth)
2. **Custom Fields**: Story points field ID varies by instance (usually `customfield_10016`)
3. **Account IDs vs Emails**: Jira uses UUIDs as primary IDs, emails in separate user objects
4. **JQL Complexity**: Querying requires Jira Query Language
5. **Pagination**: Handled via `startAt` + `maxResults` parameters

#### Jira API Endpoints Used

**1. Search Issues (JQL)**
```http
POST https://{domain}.atlassian.net/rest/api/3/search/jql
Authorization: Basic {base64(email:token)}
Content-Type: application/json
Accept: application/json

Query Parameters:
  - jql: updated >= -365d ORDER BY updated DESC
  - startAt: 0, 100, 200... (pagination offset)
  - maxResults: 100 (default 50, max 100)
  - fields: * (or specify: assignee,status,customfield_10016,etc.)
```

**Request Body** (alternative to query params):
```json
{
  "jql": "assignee = currentUser() AND updated >= -30d",
  "startAt": 0,
  "maxResults": 100,
  "fields": ["*all"]
}
```

**Response Shape**:
```json
{
  "total": 1234,
  "startAt": 0,
  "maxResults": 100,
  "issues": [
    {
      "key": "AUTH-123",
      "fields": {
        "issuetype": {
          "name": "Story"
        },
        "status": {
          "name": "In Progress",
          "statusCategory": {
            "name": "In Progress",
            "key": "indeterminate"
          }
        },
        "assignee": {
          "accountId": "5a1234567890abcdef123456",
          "emailAddress": "sarah.chen@company.com",
          "displayName": "Sarah Chen"
        },
        "project": {
          "key": "AUTH",
          "name": "Authentication"
        },
        "customfield_10016": 8,  // Story points (field ID varies!)
        "parent": {
          "key": "AUTH-100"
        },
        "labels": ["backend", "security"],
        "components": [
          {
            "name": "API"
          }
        ],
        "description": "Implement OAuth2 flow with JWT tokens...",
        "updated": "2025-12-20T14:30:00.000+0000",
        "statuscategorychangedate": "2025-12-15T10:00:00.000+0000"
      }
    }
  ]
}
```

#### Jira Sync Implementation Strategy

```typescript
export async function syncJiraTickets(
    orgId: string,
    email: string,
    token: string,
    domain: string
): Promise<{ issues: number }> {
    // 1. Construct JQL query (365-day window)
    const jql = 'updated >= -365d ORDER BY updated DESC'
    const baseUrl = `https://${domain}/rest/api/3/search/jql`

    let startAt = 0
    let total = 0
    let issueCount = 0

    // 2. Pagination loop
    do {
        // 3. Build request URL
        const url = `${baseUrl}?jql=${encodeURIComponent(jql)}&startAt=${startAt}&maxResults=100`

        // 4. Make request with Basic Auth
        const authHeader = Buffer.from(`${email}:${token}`).toString('base64')
        const response = await fetch(url, {
            headers: {
                'Authorization': `Basic ${authHeader}`,
                'Accept': 'application/json'
            }
        })

        if (!response.ok) {
            const text = await response.text()
            throw new Error(`Jira API error ${response.status}: ${text}`)
        }

        const data = await response.json()
        total = data.total

        // 5. Process each issue
        for (const issue of data.issues) {
            // CRITICAL: Extract story points from custom field
            // Field ID varies by Jira instance, commonly customfield_10016
            const storyPoints = issue.fields.customfield_10016 || null

            // Extract assignee email (may be null)
            const assigneeEmail = normalizeEmail(issue.fields.assignee?.emailAddress)
            const assigneeId = issue.fields.assignee?.accountId || null

            // 6. Upsert issue (KEY is primary key)
            await db.insertInto('jira_issues')
                .values({
                    key: issue.key,
                    project_key: issue.fields.project.key,
                    sprint_id: null,  // Extract from sprint custom field if needed
                    type: issue.fields.issuetype.name,
                    status: issue.fields.status.name,
                    status_category: issue.fields.status.statusCategory.name,
                    assignee_user_id: assigneeId,
                    assignee_email: assigneeEmail,  // CRITICAL for matching
                    points: storyPoints,
                    parent_key: issue.fields.parent?.key || null,
                    product: null,  // Extract from custom field if configured
                    updated_at: issue.fields.updated,
                    status_change_at: issue.fields.statuscategorychangedate || null,
                    labels: JSON.stringify(issue.fields.labels || []),
                    components: JSON.stringify(issue.fields.components || []),
                    description_length: issue.fields.description?.length || 0,
                    ac_present: issue.fields.description?.includes('Acceptance Criteria') ? 1 : 0,
                    org_id: orgId
                })
                .onConflict(oc => oc.column('key').doUpdateSet({
                    status: issue.fields.status.name,
                    status_category: issue.fields.status.statusCategory.name,
                    assignee_user_id: assigneeId,
                    assignee_email: assigneeEmail,
                    updated_at: issue.fields.updated,
                    status_change_at: issue.fields.statuscategorychangedate
                }))
                .execute()

            issueCount++
        }

        // 7. Advance pagination
        startAt += data.issues.length

        console.log(`[Jira] Synced ${startAt} of ${total} issues`)

    } while (startAt < total)

    return { issues: issueCount }
}
```

#### Custom Field Discovery

**Problem**: Story points field ID varies across Jira instances.

**Solution**: Query field metadata endpoint:

```http
GET https://{domain}.atlassian.net/rest/api/3/field
Authorization: Basic {base64(email:token)}
```

**Response** (excerpt):
```json
[
  {
    "id": "customfield_10016",
    "name": "Story Points",
    "custom": true,
    "schema": {
      "type": "number"
    }
  }
]
```

**Implementation**:
```typescript
async function findStoryPointsField(
    email: string,
    token: string,
    domain: string
): Promise<string | null> {
    const url = `https://${domain}/rest/api/3/field`
    const authHeader = Buffer.from(`${email}:${token}`).toString('base64')

    const response = await fetch(url, {
        headers: {
            'Authorization': `Basic ${authHeader}`,
            'Accept': 'application/json'
        }
    })

    const fields = await response.json()

    // Find field with name "Story Points" or similar
    const storyPointField = fields.find(
        (f: any) => f.custom && (
            f.name.toLowerCase().includes('story point') ||
            f.name.toLowerCase().includes('points')
        )
    )

    return storyPointField?.id || null
}
```

---

### Part 3: Identity Matching - The Complete Picture

#### The Three-Layer System

**Layer 1: provider_users**
- Raw user data from each provider
- Stores ALL available identifiers: email, username, display_name, provider_user_id
- One row per (org_id, provider, provider_user_id)

**Layer 2: identities**
- Canonical mapping: `(canonical_email, provider, provider_user_id)`
- Created via auto-linking (100% confidence) or manual linking
- Primary key: `(user_id, provider, provider_user_id)` - composite

**Layer 3: identity_rejections**
- Explicitly rejected links to prevent auto-linking
- Same primary key structure as identities
- Checked before creating new links

#### Auto-Linking Algorithm (Detailed)

**Trigger**: Runs before each unified view query

**Input**: Organization ID

**Output**: Count of new links created

**Full Implementation**:
```typescript
export async function autoLinkIdentities(orgId: string): Promise<number> {
    // STEP 1: Extract canonical users (Claude emails from AI metrics)
    const claudeUsersResult = await db
        .selectFrom('ai_metric_samples')
        .select('user_id')
        .where('org_id', '=', orgId)
        .distinct()
        .execute()

    // Normalize and create set for O(1) lookup
    const canonicalSet = new Set<string>()
    for (const row of claudeUsersResult) {
        const normalized = normalizeEmail(row.user_id)
        if (normalized) {
            canonicalSet.add(normalized)
        }
    }

    console.log(`[Identity] Found ${canonicalSet.size} canonical users from Claude metrics`)

    // STEP 2: Fetch all provider users with emails
    const providerUsers = await db
        .selectFrom('provider_users')
        .selectAll()
        .where('org_id', '=', orgId)
        .where('email', 'is not', null)
        .execute()

    console.log(`[Identity] Found ${providerUsers.length} provider users with emails`)

    let newLinks = 0

    // STEP 3: For each provider user, check for email match
    for (const pu of providerUsers) {
        const normalizedEmail = normalizeEmail(pu.email)

        // Skip if no email or email doesn't match any canonical user
        if (!normalizedEmail || !canonicalSet.has(normalizedEmail)) {
            continue
        }

        // STEP 4: Determine provider_user_id to use for linking
        // For GitHub: prefer username over provider_user_id (numeric ID)
        // For Jira/Claude: use provider_user_id as-is
        const linkId = (pu.provider === 'github' && pu.username)
            ? pu.username
            : pu.provider_user_id

        // STEP 5: Check if link already exists
        const existingLink = await db
            .selectFrom('identities')
            .selectAll()
            .where('user_id', '=', normalizedEmail)
            .where('provider', '=', pu.provider)
            .where('provider_user_id', '=', linkId)
            .executeTakeFirst()

        if (existingLink) {
            continue  // Link already exists, skip
        }

        // STEP 6: Check if link is explicitly rejected
        const rejectedLink = await db
            .selectFrom('identity_rejections')
            .selectAll()
            .where('user_id', '=', normalizedEmail)
            .where('provider', '=', pu.provider)
            .where('provider_user_id', '=', linkId)
            .executeTakeFirst()

        if (rejectedLink) {
            console.log(
                `[Identity] Skipping rejected link: ${normalizedEmail} → ` +
                `${pu.provider}:${linkId}`
            )
            continue
        }

        // STEP 7: Create new identity link
        await db
            .insertInto('identities')
            .values({
                user_id: normalizedEmail,
                provider: pu.provider,
                provider_user_id: linkId,
                confidence: 100,  // Auto-linked via exact email match
                created_at: new Date().toISOString()
            })
            .execute()

        console.log(
            `[Identity] Auto-linked: ${normalizedEmail} → ${pu.provider}:${linkId}`
        )

        newLinks++
    }

    console.log(`[Identity] Created ${newLinks} new identity links`)
    return newLinks
}
```

#### Building the Identity Map

**Used By**: Unified view correlation

**Purpose**: Fast O(1) lookup from provider ID to canonical email

**Implementation**:
```typescript
async function buildIdentityMap(orgId: string): Promise<Map<string, string>> {
    // Fetch all identity links
    const identities = await db
        .selectFrom('identities')
        .selectAll()
        .execute()

    // Build map: "provider:provider_user_id" → canonical_email
    const identityMap = new Map<string, string>()

    for (const identity of identities) {
        const key = `${identity.provider}:${identity.provider_user_id}`
        identityMap.set(key, identity.user_id)
    }

    return identityMap
}
```

**Usage Example** (in unified view):
```typescript
const identityMap = await buildIdentityMap(orgId)

// Resolve GitHub username to canonical email
const githubUsername = 'sarahc-dev'
const lookupKey = `github:${githubUsername}`
const canonicalEmail = identityMap.get(lookupKey)

if (canonicalEmail) {
    console.log(`${githubUsername} maps to ${canonicalEmail}`)
} else {
    console.log(`${githubUsername} has no identity link`)
}
```

#### Unified View Correlation (Detailed)

**Most Complex Query**: Aggregates data across three providers using dual-path resolution.

**Full Implementation**:
```typescript
async function getUnifiedTable(orgId: string) {
    // STEP 1: Auto-link identities (ensure fresh mapping)
    const newLinks = await autoLinkIdentities(orgId)
    console.log(`[Unified] Auto-linked ${newLinks} new identities`)

    // STEP 2: Build identity map for fast lookups
    const identityMap = await buildIdentityMap(orgId)
    console.log(`[Unified] Built identity map with ${identityMap.size} entries`)

    // STEP 3: Aggregate Claude metrics by canonical email
    const claudeMetrics = await db
        .selectFrom('ai_metric_samples')
        .select(['user_id'])
        .select((eb) => [
            eb.fn.sum<number>(
                eb.fn('json_extract', ['metrics_json', '$.claude_code.cost.usage'])
            ).as('total_cost'),
            eb.fn.sum<number>(
                eb.fn('json_extract', ['metrics_json', '$.claude_code.lines_of_code.count'])
            ).as('total_loc')
        ])
        .where('org_id', '=', orgId)
        .groupBy('user_id')
        .execute()

    // Convert to map: canonical_email → { cost, loc }
    const claudeMap = new Map<string, { cost: number, loc: number }>()
    for (const row of claudeMetrics) {
        const email = normalizeEmail(row.user_id)
        if (email) {
            claudeMap.set(email, {
                cost: row.total_cost || 0,
                loc: row.total_loc || 0
            })
        }
    }

    // STEP 4: Aggregate GitHub metrics (DUAL PATH)

    // Path A: Via identity map (author_user_id → canonical email)
    const githubCommits = await db
        .selectFrom('commits')
        .select(['author_user_id', 'committer_email'])
        .select((eb) => [
            eb.fn.sum<number>('lines_added').as('total_added'),
            eb.fn.sum<number>('lines_deleted').as('total_deleted'),
            eb.fn.count<number>('sha').as('commit_count')
        ])
        .where('org_id', '=', orgId)
        .groupBy(['author_user_id', 'committer_email'])
        .execute()

    const githubMap = new Map<string, { loc: number, commits: number }>()

    for (const row of githubCommits) {
        let canonicalEmail: string | null = null

        // Resolution path 1: Identity map (author_user_id)
        if (row.author_user_id) {
            const lookupKey = `github:${row.author_user_id}`
            canonicalEmail = identityMap.get(lookupKey) || null
        }

        // Resolution path 2: Direct email match (committer_email)
        if (!canonicalEmail && row.committer_email) {
            canonicalEmail = normalizeEmail(row.committer_email)
        }

        if (canonicalEmail) {
            const existing = githubMap.get(canonicalEmail) || { loc: 0, commits: 0 }
            existing.loc += (row.total_added || 0) + (row.total_deleted || 0)
            existing.commits += row.commit_count || 0
            githubMap.set(canonicalEmail, existing)
        }
    }

    // Path B: Pull requests
    const githubPRs = await db
        .selectFrom('pull_requests')
        .select(['author_user_id', 'author_email'])
        .select((eb) => [
            eb.fn.count<number>('id').as('pr_count')
        ])
        .where('org_id', '=', orgId)
        .groupBy(['author_user_id', 'author_email'])
        .execute()

    const prMap = new Map<string, number>()

    for (const row of githubPRs) {
        let canonicalEmail: string | null = null

        // Resolution path 1: Identity map
        if (row.author_user_id) {
            const lookupKey = `github:${row.author_user_id}`
            canonicalEmail = identityMap.get(lookupKey) || null
        }

        // Resolution path 2: Direct email match
        if (!canonicalEmail && row.author_email) {
            canonicalEmail = normalizeEmail(row.author_email)
        }

        if (canonicalEmail) {
            prMap.set(canonicalEmail, (prMap.get(canonicalEmail) || 0) + (row.pr_count || 0))
        }
    }

    // STEP 5: Aggregate Jira metrics (DUAL PATH)
    const jiraIssues = await db
        .selectFrom('jira_issues')
        .select(['assignee_user_id', 'assignee_email'])
        .select((eb) => [
            eb.fn.count<number>('key').as('issue_count')
        ])
        .where('org_id', '=', orgId)
        .groupBy(['assignee_user_id', 'assignee_email'])
        .execute()

    const jiraMap = new Map<string, number>()

    for (const row of jiraIssues) {
        let canonicalEmail: string | null = null

        // Resolution path 1: Identity map
        if (row.assignee_user_id) {
            const lookupKey = `jira:${row.assignee_user_id}`
            canonicalEmail = identityMap.get(lookupKey) || null
        }

        // Resolution path 2: Direct email match
        if (!canonicalEmail && row.assignee_email) {
            canonicalEmail = normalizeEmail(row.assignee_email)
        }

        if (canonicalEmail) {
            jiraMap.set(canonicalEmail, (jiraMap.get(canonicalEmail) || 0) + (row.issue_count || 0))
        }
    }

    // STEP 6: Collect all canonical users
    const allUsers = new Set<string>()
    claudeMap.forEach((_, email) => allUsers.add(email))
    githubMap.forEach((_, email) => allUsers.add(email))
    prMap.forEach((_, email) => allUsers.add(email))
    jiraMap.forEach((_, email) => allUsers.add(email))

    // STEP 7: Build unified rows
    const rows = []

    for (const canonicalEmail of allUsers) {
        const claude = claudeMap.get(canonicalEmail) || { cost: 0, loc: 0 }
        const github = githubMap.get(canonicalEmail) || { loc: 0, commits: 0 }
        const prs = prMap.get(canonicalEmail) || 0
        const jiraTickets = jiraMap.get(canonicalEmail) || 0

        // Find provider IDs from identity map (reverse lookup)
        let githubId: string | null = null
        let jiraId: string | null = null

        for (const [key, email] of identityMap.entries()) {
            if (email === canonicalEmail) {
                const [provider, providerId] = key.split(':')
                if (provider === 'github') githubId = providerId
                if (provider === 'jira') jiraId = providerId
            }
        }

        rows.push({
            user: canonicalEmail,
            github_id: githubId,
            jira_id: jiraId,
            claude_cost: claude.cost,
            claude_loc: claude.loc,
            github_loc: github.loc,
            prs_created: prs,
            jira_tickets: jiraTickets
        })
    }

    return { rows }
}
```

#### Edge Cases Handled

1. **Missing Identity Links**: Falls back to direct email matching
2. **Multiple GitHub Emails**: Aggregates by canonical email (consolidates multiple Git configs)
3. **Private GitHub Emails**: Uses author_user_id (username) for linking when email hidden
4. **Jira UUID Changes**: Links via email (stable) rather than accountId (can change on user merge)
5. **Case-Sensitive Emails**: Normalization ensures case-insensitive matching
6. **Whitespace in Emails**: Trim() applied before comparison
7. **Null/Empty Emails**: Skipped gracefully, no errors thrown

---

### Part 4: Common Pitfalls and Solutions

#### Pitfall 1: Rate Limit Exhaustion (GitHub)

**Symptom**: Sync fails midway with 403 error

**Root Cause**: Fetching commit details for every commit (5000 commits = 5000+ requests)

**Solution**:
- Track rate limit in-memory
- Check before each request
- Log warnings at 100 remaining
- Throw descriptive error with reset time

#### Pitfall 2: Story Points Field Not Found (Jira)

**Symptom**: All issues show null story points

**Root Cause**: Custom field ID varies by Jira instance

**Solution**:
- Document common field IDs (customfield_10016 most common)
- Provide field discovery endpoint
- Allow configuration override

#### Pitfall 3: Email Mismatch (GitHub)

**Symptom**: User with commits shows 0 commits in unified view

**Root Cause**: Git commit email ≠ GitHub account email

**Solution**:
- Store BOTH author_user_id (username) AND committer_email
- Use dual-path resolution: identity map first, email fallback second
- Auto-linking creates map from username to canonical email

#### Pitfall 4: Duplicate Identity Links

**Symptom**: Same user linked multiple times to same provider

**Root Cause**: Auto-linking runs multiple times without idempotency

**Solution**:
- Primary key on (user_id, provider, provider_user_id) prevents duplicates
- `.onConflict().doNothing()` in insert makes it idempotent

#### Pitfall 5: Pagination Incomplete

**Symptom**: Only first 100 results returned

**Root Cause**: Forgot to implement pagination loop

**Solution**:
- GitHub: Check for `Link` header, follow `next` rel
- Jira: Use `startAt + maxResults < total` condition in while loop

---

### Part 5: Testing Strategy

#### Integration Test: End-to-End Sync

```typescript
describe('GitHub Sync Integration', () => {
    it('should sync commits and match to canonical user', async () => {
        // 1. Insert Claude user
        await db.insertInto('ai_metric_samples').values({
            org_id: 'test_org',
            user_id: 'sarah.chen@company.com',
            bucket_start: '2025-12-01T00:00:00Z',
            bucket_end: '2025-12-01T23:59:59Z',
            repo_hint: null,
            metrics_json: JSON.stringify({
                'claude_code.session.count': 5,
                'claude_code.cost.usage': 1000
            })
        }).execute()

        // 2. Sync GitHub data (mock or use test repo)
        await syncGithubActivity('test_org', TEST_GITHUB_TOKEN, '2025-12-01T00:00:00Z')

        // 3. Check commits stored
        const commits = await db
            .selectFrom('commits')
            .selectAll()
            .where('org_id', '=', 'test_org')
            .execute()

        expect(commits.length).toBeGreaterThan(0)

        // 4. Run auto-linking
        const newLinks = await autoLinkIdentities('test_org')
        expect(newLinks).toBeGreaterThan(0)

        // 5. Check unified view
        const unified = await getUnifiedTable('test_org')
        const sarahRow = unified.rows.find(r => r.user === 'sarah.chen@company.com')

        expect(sarahRow).toBeDefined()
        expect(sarahRow.github_loc).toBeGreaterThan(0)
        expect(sarahRow.claude_cost).toBe(1000)
    })
})
```

---

## Summary

This specification provides complete technical documentation for recreating SlipStream Analytics from scratch without access to source code.

### Key Architectural Principles

1. **Three-Layer Identity System**: Canonical email-based mapping with auto-linking and manual override
2. **Separation of Concerns**: Routes, services, and database layers clearly delineated
3. **Type Safety**: TypeScript + Kysely for compile-time query validation
4. **Extensibility**: Multi-org support via `org_id` column in all tables
5. **Developer Experience**: Debug tools, comprehensive logging, and clear APIs
6. **Scheduler Integration**: Daily automated syncs with cron
7. **Frontend-Backend Separation**: Vite/React + Hono with clean API contracts

### Implementation Checklist

- [ ] Set up Node.js project with TypeScript
- [ ] Install backend dependencies (Hono, Kysely, better-sqlite3, node-cron)
- [ ] Implement database schema and migrations
- [ ] Create service layer (claude.ts, github.ts, jira.ts, identity.ts)
- [ ] Implement API routes (config, sync, unified)
- [ ] Set up frontend with Vite + React
- [ ] Create Layout and page components
- [ ] Implement SearchableSelect custom component
- [ ] Configure Tailwind CSS v4
- [ ] Test identity auto-linking algorithm
- [ ] Test unified view correlation
- [ ] Configure environment variables
- [ ] Set up scheduler for daily syncs
- [ ] Deploy with Docker (optional)

All code patterns, algorithms, and configurations are documented with sufficient detail for implementation by a skilled full-stack developer with TypeScript experience.

---

**End of Specification**
