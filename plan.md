# Comprehensive Backend Architecture & Implementation Blueprint

> **Purpose**: This document provides the complete architectural context, database schema, operational workflows, and a step-by-step implementation guide to replicate this exact backend (Auth, Role-Based Access Control, GitHub Contribution Sync Engine, Real-time Leaderboard, Admin Command Center, and Badge System) into any new Next.js project.

---

## 1. System Architecture Overview

The backend is built around **Next.js (App Router)** and **Supabase (Auth + PostgreSQL + Realtime)**, coupled with an external integration with the **GitHub REST API**.

```
                           +------------------------+
                           |      Next.js Client    |
                           |   (React Server/Client)|
                           +-----------+------------+
                                       |
                   +-------------------+-------------------+
                   |                   |                   |
         [Browser Client]       [Server Client]     [Server Actions]
          Anon Key + RLS        Cookie-based SSR     Admin Service Role
                   |                   |                   |
                   v                   v                   v
        +--------------------------------------------------------+
        |                 Supabase Backend                       |
        |  - GoTrue Auth (Email + OAuth)                         |
        |  - PostgreSQL Database (profiles table)                |
        |  - Row Level Security (RLS)                            |
        |  - Postgres Triggers (auto-create profile)             |
        |  - Realtime CDC (Postgres changes -> WebSocket)        |
        +--------------------------+-----------------------------+
                                   |
                   +---------------+---------------+
                   |                               |
                   v                               v
       +-----------------------+       +-----------------------+
       |   GitHub REST API     |       |   Static Project Data |
       | Search Issues & PRs   |       | competition repos list|
       +-----------------------+       +-----------------------+
```

### The Three Supabase Client Tiers

To maintain strict security and cookie consistency, the system uses 3 distinct Supabase clients:

1. **Browser Client (`src/lib/supabase/client.ts`)**:
   - Uses `createBrowserClient` with `NEXT_PUBLIC_SUPABASE_URL` and `NEXT_PUBLIC_SUPABASE_ANON_KEY`.
   - Runs in client components for auth state listeners, realtime subscriptions, and client-side fetches.
2. **Server Client (`src/lib/supabase/server.ts`)**:
   - Uses `createServerClient` from `@supabase/ssr` with Next.js `cookies()`.
   - Runs in Server Components and Server Actions to read/write authenticated sessions securely via cookies.
3. **Admin Client (`src/lib/supabase/admin.ts`)**:
   - Uses `createClient` from `@supabase/supabase-js` with `SUPABASE_SERVICE_ROLE_KEY`.
   - **Bypasses Row Level Security (RLS)**.
   - Strictly reserved for server-side actions: sync operations, admin role updates, and scoring overrides.

---

## 2. Complete Database Schema & Supabase Setup

The original project had schema migrations split across files, and several columns were added dynamically. Below is the **unified, production-ready SQL script** to set up the entire backend database from scratch.

### Consolidated Database Migration (`schema.sql`)

```sql
-- 1. Enable UUID Extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- 2. Create the unified 'profiles' table
CREATE TABLE public.profiles (
  id UUID REFERENCES auth.users ON DELETE CASCADE NOT NULL PRIMARY KEY,
  full_name TEXT,
  email TEXT,
  github TEXT UNIQUE,
  linkedin TEXT,
  phone TEXT,
  country_code TEXT DEFAULT '+91',
  country TEXT,
  nexfellow_id TEXT,
  avatar_url TEXT,
  role TEXT DEFAULT 'contributor' CHECK (role IN ('contributor', 'mentor', 'project-admin', 'admin')),
  is_admin BOOLEAN DEFAULT FALSE,
  score INTEGER DEFAULT 0,
  merged_prs INTEGER DEFAULT 0,
  projects_count INTEGER DEFAULT 0,
  badges_created INTEGER DEFAULT 0,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT timezone('utc'::text, now()) NOT NULL,
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT timezone('utc'::text, now()) NOT NULL
);

-- 3. Create Performance & Lookup Indexes
CREATE INDEX idx_profiles_role ON public.profiles (role);
CREATE INDEX idx_profiles_score ON public.profiles (score DESC);
CREATE INDEX idx_profiles_github ON public.profiles (github);
CREATE INDEX idx_profiles_email ON public.profiles (email);

-- 4. Enable Row Level Security (RLS)
ALTER TABLE public.profiles ENABLE ROW LEVEL SECURITY;

-- 5. Define RLS Policies
-- Anyone can view profiles (needed for leaderboard, public profile preview)
CREATE POLICY "Public profiles are viewable by everyone" 
  ON public.profiles
  FOR SELECT 
  USING (true);

-- Authenticated users can insert their own initial profile row
CREATE POLICY "Users can insert their own profile" 
  ON public.profiles
  FOR INSERT 
  WITH CHECK (auth.uid() = id);

-- Users can only update their own profile
CREATE POLICY "Users can update own profile" 
  ON public.profiles
  FOR UPDATE 
  USING (auth.uid() = id);

-- 6. Trigger Function to automatically create profile row on auth signup
CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO public.profiles (id, full_name, email, role, is_admin)
  VALUES (
    new.id,
    COALESCE(new.raw_user_meta_data->>'full_name', split_part(new.email, '@', 1)),
    new.email,
    'contributor',
    FALSE
  );
  RETURN new;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- 7. Attach Trigger to Supabase Auth table
DROP TRIGGER IF EXISTS on_auth_user_created ON auth.users;
CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE PROCEDURE public.handle_new_user();

-- 8. Enable Realtime Replication for the Leaderboard
-- Allows client to listen to UPDATE/INSERT/DELETE events live
ALTER PUBLICATION supabase_realtime ADD TABLE public.profiles;
```

### Initial Admin Setup Script (`admin_seed.sql`)

To grant a user Super Admin privileges:

```sql
UPDATE public.profiles
SET is_admin = TRUE,
    role = 'admin',
    score = 0,
    merged_prs = 0,
    projects_count = 0
WHERE email = 'your-admin-email@example.com';
```

---

## 3. Core Workflows & Logic

### Workflow 1: Authentication & Route Protection

```
User visits /admin or /dashboard
              │
              ▼
   Next.js middleware.ts
   (Calls supabase.auth.getUser())
              │
    ┌─────────┴─────────┐
    │                   │
No User?            User Exists?
    │                   │
    ▼                   ▼
Redirect to         Is it /admin?
/sign-in                ├── Yes ──► Check profile.is_admin
                        │             ├── False ──► Redirect to /
                        │             └── True  ──► Allow Next.js page
                        └── No (e.g. /dashboard) ──► Allow Next.js page
```

- **Next.js Edge Middleware (`src/middleware.ts`)**:
  - Intercepts requests matching `/admin/:path*`, `/dashboard/:path*`, and `/badge/:path*`.
  - Calls `supabase.auth.getUser()` (never trust local cookies blindly; `getUser()` validates the token against Supabase auth servers).
  - Queries `profiles.is_admin` for `/admin` routes. Non-admins are bounced to `/`. Unauthenticated users are sent to `/sign-in`.
- **OAuth Callback (`src/app/auth/callback/route.ts`)**:
  - Exchanges the authorization `code` returned by GitHub/Google for a session cookie:
    ```ts
    await supabase.auth.exchangeCodeForSession(code);
    ```
  - Redirects user to the requested page or `/admin`.

---

### Workflow 2: GitHub Contribution Sync & Scoring Engine

This is the most critical and sophisticated backend component (`src/lib/actions/github.ts`).

#### How GitHub Sync Works Step-by-Step

```
syncGitHubContribution(userId, githubHandle)
  │
  ├─► 1. Normalize handle (strip '@', strip 'https://github.com/')
  │
  ├─► 2. Check Role:
  │      If role !== 'contributor' (admin, project-admin):
  │      Reset score=0, merged_prs=0, projects_count=0 -> RETURN EARLY
  │
  ├─► 3. Extract Allowed Repositories:
  │      Parse PROJECTS array -> extract "owner/repo" in lowercase.
  │
  ├─► 4. Fetch GitHub Data (Search API):
  │      Query A: "author:{handle} type:pr is:closed"
  │      Query B: "assignee:{handle} type:issue is:closed"
  │
  ├─► 5. Filter:
  │      Only keep PRs/Issues whose repository matches the competition list!
  │
  ├─► 6. Calculate Difficulty & Inheritance:
  │      Inspect PR labels, title, body for keywords.
  │      Look for "fixes #123" -> find linked issue.
  │      If linked issue difficulty > PR difficulty, inherit higher weight!
  │
  ├─► 7. Compute Score:
  │      Score = (easy * 10) + (med * 20) + (hard * 30) + (exp * 50)
  │
  └─► 8. Update Supabase profiles:
         merged_prs, projects_count, score, updated_at
```

#### Scoring & Difficulty Weights

| Difficulty Level | Detected Keywords in Labels/Title/Body | Points Awarded |
|---|---|---|
| **Easy** | `easy`, `beginner`, `starter` (or default if unlabelled) | **10 pts** |
| **Medium** | `medium`, `med`, `intermediate`, `mid` | **20 pts** |
| **Hard** | `hard`, `high` | **30 pts** |
| **Expert** | `expert`, `exp`, `advanced` | **50 pts** |

#### Linked Issue Inheritance Feature
If a developer opens a PR that mentions `Fixes #45` or `Closes #45`:
1. The engine extracts issue number `#45` from the PR body.
2. It looks up issue `#45` within the fetched repository issues.
3. If the issue has a higher difficulty rating (e.g. Issue is `hard`, PR was labelled `easy`), the PR automatically inherits the higher difficulty rating (`hard` = 30 pts).

#### When Sync is Triggered
1. **Lazy Sync**: Every time a user visits their `/dashboard`, the client fetches profile data and automatically fires `syncGitHubContribution` in the background.
2. **Admin Single Sync**: Admins can click the refresh button next to any user in `/admin`.
3. **Admin Bulk Sync**: Admins can click **Sync All Users** in `/admin`. It iterates through all contributors with a 2-second delay between users to strictly avoid GitHub API secondary rate limits.

---

### Workflow 3: The Real-time Leaderboard

The leaderboard (`src/app/leaderboard/page.tsx`) displays rankings dynamically without requiring manual page refreshes:

1. **Initial Data Fetch**:
   - Queries `profiles` table for `role = 'contributor'`, ordered by `score DESC` (limit 100).
   - Queries all GitHub handles linked to `admin`, `project-admin`, or `is_admin = true`.
2. **Security & Integrity Filters (In-Memory)**:
   - **Admin Scrubbing**: Discards any row whose GitHub handle matches an administrator.
   - **De-duplication**: Discards duplicate handles (only keeping the highest score).
   - **Score Sanity Check**: Ensures users with `0 PRs` and `0 projects` do not show phantom scores.
   - Limits the displayed leaderboard to the **Top 50**.
3. **Supabase Realtime Channel**:
   - Listens to PostgreSQL `INSERT`/`UPDATE`/`DELETE` on `profiles`:
     ```ts
     const channel = supabase
       .channel("leaderboard_feed")
       .on("postgres_changes", { event: "*", schema: "public", table: "profiles" }, () => {
         fetchLeaderboard();
       })
       .subscribe();
     ```
   - Whenever an admin awards points, or a user syncs PRs, the leaderboard updates for all viewers simultaneously.
4. **Visual Layout**:
   - **Podium (Top 3)**: Rank 1 (Center, Gold), Rank 2 (Left, Silver), Rank 3 (Right, Bronze).
   - **List (Ranks 4 to 50)**: Detailed rows showing Rank, Name, Country Flag, PR count, Projects count, and Score.

---

### Workflow 4: Role-Based Access Control (RBAC) & Administrative Actions

There are 4 distinct roles:

| Role | Permissions & Behavior |
|---|---|
| **Contributor** | Can participate, earn points, track PRs, generate badges, and appear on the Leaderboard. |
| **Mentor** | Can create badges with a distinct Gold/Purple "MENTOR" theme. Excluded from leaderboard points. |
| **Project Admin** | Accesses the Project Admin Console (`/dashboard`), can assign merit points (+10 to +100 or Reset) to contributors. Excluded from leaderboard points. |
| **Admin (Super Admin)** | Full access to `/admin`, can manage all roles, export CSV, trigger individual & global syncs. Excluded from leaderboard points. |

#### Administrative Rules Implemented in Code:
- **No Admin Scoring**: If a user is promoted to `admin` or `project-admin`, their `score`, `merged_prs`, and `projects_count` are automatically reset to `0`.
- **Anti-Tampering / No Self-Scoring**: In `updateUserScore`, the backend strictly validates:
  ```ts
  if (requester.id === userId) {
    return { success: false, error: "Self-scoring is strictly prohibited." };
  }
  ```
- **Contributor-Only Targets**: Project Admins cannot score other Admins or Project Admins.

---

### Workflow 5: Badge Creation & Limit System

- Located in `src/app/badge/page.tsx` and `src/lib/supabase/database.ts`.
- Allows users to customize and export a personalized verification badge (PNG) using `html-to-image`.
- **Account Limit Enforcement**:
  - `profiles.badges_created` tracks the total downloads.
  - Server enforces a maximum limit of **3 badges per account**.
  - Once downloaded, `incrementBadgeCount(userId)` increments `badges_created`.

---

## 4. Environment Variables Required

Create `.env.local` with the following variables:

```bash
# ==========================================
# 1. SUPABASE (Required)
# ==========================================
# Found in: Supabase Dashboard -> Settings -> API
NEXT_PUBLIC_SUPABASE_URL=https://your-project-ref.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
SUPABASE_SERVICE_ROLE_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

# ==========================================
# 2. GITHUB INTEGRATION (Required for Sync)
# ==========================================
# GitHub Personal Access Token (classic with public_repo or repo scope)
# Needed to prevent GitHub Search API rate-limiting (60 req/hr unauthenticated vs 5000 req/hr authenticated)
GITHUB_ACCESS_TOKEN=ghp_yourPersonalAccessTokenHere

# ==========================================
# 3. APPLICATION CONFIGURATION (Optional)
# ==========================================
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

---

## 5. Step-by-Step Guide to Replicate in a New Project

Follow these exact steps to build this backend in any new Next.js project:

### Step 1: Install Dependencies
```bash
npm install @supabase/supabase-js @supabase/ssr lucide-react sonner
```

### Step 2: Set Up Database & Supabase Settings
1. Create a project at [supabase.com](https://supabase.com).
2. Go to **SQL Editor** and run the consolidated migration script from **Section 2** of this document.
3. Go to **Authentication -> Providers**:
   - Enable **Email** (disable email confirmations if prototyping).
   - (Optional) Enable **GitHub** and **Google** OAuth, providing `http://localhost:3000/auth/callback` as redirect URI.

### Step 3: Implement Supabase Client Helpers
Replicate the files in `src/lib/supabase/`:
- `client.ts`: Browser client initialization.
- `server.ts`: Server client with cookie handling.
- `admin.ts`: Supabase Admin client with service role key.
- `auth.ts`: Auth helper methods (`signUpWithEmail`, `signInWithEmail`, `signInWithGoogle`, etc.).
- `database.ts`: Profile helper methods (`getProfile`, `incrementBadgeCount`).

### Step 4: Implement Competition Projects Registry
Create `src/data/projects.ts` exporting an array of projects with `githubRepo` fields:
```ts
export interface Project {
  title: string;
  githubRepo: string;
}

export const PROJECTS: Project[] = [
  { title: "Project Alpha", githubRepo: "https://github.com/org/project-alpha" },
  { title: "Project Beta", githubRepo: "https://github.com/org/project-beta" }
];
```

### Step 5: Implement Server Actions
Replicate the actions in `src/lib/actions/`:
- `github.ts`: The GitHub sync algorithm, difficulty extraction, linked issue resolution, and score calculations.
- `admin.ts`: Admin queries (`getAdminData`), role modification (`updateUserRole`), manual scoring (`updateUserScore`), and batch user synchronization (`syncAllUsers`).

### Step 6: Configure Edge Middleware
Create `src/middleware.ts` to protect `/admin`, `/dashboard`, and `/badge`.

### Step 7: Build UI Pages
- `/leaderboard`: Subscribes to `leaderboard_feed` via Supabase Realtime, displays Podium & Top 50.
- `/dashboard`: Triggers lazy sync on mount, displays user stats, or Project Admin scoring console if role is `project-admin`.
- `/admin`: Super Admin portal to filter users, export CSV, change roles, and trigger single/bulk syncs.
- `/badge`: Image upload, adjustment controls, limit checker (max 3 badges), and PNG export.

---

## 6. Testing & Debugging Checklist

- [ ] **Test Auth Signup**: Sign up with email -> verify new row in `profiles` table via trigger.
- [ ] **Test GitHub Sync**:
  - Run standalone script: `npx tsx src/scripts/debug-sync.ts <github_username>`
  - Verify GitHub API query matches competition repos.
  - Verify `merged_prs`, `projects_count`, and `score` update in database.
- [ ] **Test Realtime Leaderboard**:
  - Open `/leaderboard` in one browser tab.
  - In another tab (or via SQL), update a contributor's score.
  - Verify the leaderboard updates instantly without page refresh.
- [ ] **Test Admin Security**:
  - Verify standard users cannot access `/admin`.
  - Verify Project Admins cannot score themselves.
  - Verify promoting a user to Admin resets their score and PR metrics to `0`.
