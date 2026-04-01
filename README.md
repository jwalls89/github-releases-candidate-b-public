# Candidate B: Trunk-Based + Release Branches

Trunk-based development with release branches and tag-driven promotion through environment gates.

## How It Works

```
         main (development)              release/1.2.0 (promotion)
         ==================              ===========================

    feature branch
    ──────┐
          ▼
  ──A──B──C──D──E──F──────►             D (branch cut here)
              │                          │
              │  cut-release 1.2.0       ├── tag v1.2.0-rc.1
              └─────────────────────►    │     │
                                         │     ├── auto → test
         (dev continues on main)         │     ├── approval → preprod
                                         │     ├── approval → prod
  ──G──H──I──J──────────────►            │     │
                                         │     ├── tag v1.2.0 (final)
                                         │     └── GitHub Release created
                                         │
                                         └── branch kept for hotfixes
```

### Pipeline Flow

```
 ┌──────────────┐         ┌─────────────────────────────────────────────────┐
 │  Cut Release  │         │              Promote (auto-triggered)           │
 │  (manual)     │         │                                                 │
 │               │ triggers│  ┌──────┐  ┌─────────┐  ┌──────┐  ┌─────────┐ │
 │ Creates:      │────────►│  │ Test │─►│ Preprod  │─►│ Prod │─►│Finalise │ │
 │ - branch      │         │  │(auto)│  │(approval)│  │(approval)│(auto)  │ │
 │ - rc.1 tag    │         │  └──────┘  └─────────┘  └──────┘  └─────────┘ │
 └──────────────┘         │                                                 │
                           │  Creates: final tag + GitHub Release            │
 ┌──────────────┐         │                                                 │
 │  Tag New RC   │ triggers│  (same pipeline restarts from test)             │
 │  (manual)     │────────►│                                                 │
 │               │         └─────────────────────────────────────────────────┘
 │ After a fix,  │
 │ creates next  │
 │ RC tag        │
 └──────────────┘
```

### Key Concepts

| Concept | What it means |
|---------|---------------|
| **main** | Where all development happens. Never stops. |
| **release/X.Y.Z** | Cut from main when ready to release. Lives ~1 week. |
| **v1.2.0-rc.N** | Release candidate tag. Triggers the promotion pipeline. |
| **v1.2.0** | Final release tag. Created automatically when RC reaches prod. |
| **GitHub Release** | Created only when deployed to prod. This IS the release. |

### Environments

| Environment | Gate | Who approves |
|-------------|------|-------------|
| test | Automatic | Nobody — deploys immediately |
| preprod | Required reviewer | Configured in repo settings |
| prod | Required reviewer | Configured in repo settings |

### Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| **Cut Release** | Manual (workflow_dispatch) | Creates release branch + first RC tag, triggers Promote |
| **Tag New RC** | Manual (workflow_dispatch) | Creates next RC tag on release branch after a fix, triggers Promote |
| **Promote** | Auto-triggered by Cut Release or Tag New RC | Deploys through test → preprod → prod, finalises release |
| **Deploy** | Called by Promote | Reusable workflow that simulates deployment to one environment |

---

## Setup

Before running scenarios, configure GitHub Environments:

1. Go to **Settings → Environments**
2. Create environment `test` — no protection rules needed
3. Create environment `preprod`:
   - Tick **Required reviewers**
   - Add yourself (or your team)
   - Click **Save protection rules**
4. Create environment `prod`:
   - Tick **Required reviewers**
   - Add yourself (or your team)
   - Click **Save protection rules**

---

## Scenario 1: Happy Path

A clean release — no fixes needed, straight through to prod.

### Step 1: Cut the release

1. Go to **Actions → Cut Release → Run workflow**
2. Enter version: `1.2.0`
3. Click **Run workflow**

This creates branch `release/1.2.0`, tag `v1.2.0-rc.1`, and automatically triggers the Promote workflow.

### Step 2: Watch it flow

Go to **Actions** — you'll see two workflow runs:
- **Cut Release** (completed) — created the branch and tag
- **Promote** (in progress) — deploying through environments

The Promote workflow:
1. **Test** — deploys automatically
2. **Preprod** — pauses waiting for approval
   - Click the workflow run → **Review deployments** → tick `preprod` → **Approve and deploy**
3. **Prod** — pauses waiting for approval
   - Click **Review deployments** → tick `prod` → **Approve and deploy**
4. **Finalise** — automatically creates tag `v1.2.0` and GitHub Release

### Step 3: Verify

- **Releases** page → `v1.2.0` is the latest release with auto-generated notes
- **Settings → Environments** → each environment shows deployment history
- **Code → Tags** → `v1.2.0-rc.1` and `v1.2.0` both exist

---

## Scenario 2: Unpolished Path

Release hits problems during promotion. Fix, re-tag, restart.

### Step 1: Cut and start promotion

1. **Cut Release** with version `1.3.0`
2. Promote triggers automatically, deploys to test
3. Something is wrong — bad config value in test
4. **Do NOT approve preprod** — let it sit or reject it

### Step 2: Fix the issue on the release branch

**Option A — Fix on main first (preferred):**

```bash
# Fix via a normal PR to main
git checkout main
git checkout -b fix/test-config
# Edit environments/test.json
git add environments/test.json
git commit -m "fix: correct api endpoint in test config"
git push origin fix/test-config
# Open PR to main, review, merge

# Cherry-pick the fix to the release branch
git checkout release/1.3.0
git pull origin release/1.3.0
git cherry-pick <commit-sha-from-main>
git push origin release/1.3.0
```

**Option B — Fix directly on release branch (when urgent):**

```bash
git checkout release/1.3.0
# Edit environments/test.json
git add environments/test.json
git commit -m "fix: correct api endpoint in test config"
git push origin release/1.3.0

# IMPORTANT: cherry-pick back to main afterwards so the fix isn't lost
git checkout main
git cherry-pick <commit-sha-from-release>
git push origin main
```

### Step 3: Tag a new RC and restart promotion

1. Go to **Actions → Tag New RC → Run workflow**
2. Enter version: `1.3.0`
3. Click **Run workflow**

This auto-detects the next RC number (e.g., `rc.2`), tags it, and triggers Promote. The pipeline restarts from test with the fix included.

### Step 4: Approve through environments

If another issue is found at preprod, fix it, run **Tag New RC** again (creates `rc.3`), and the pipeline restarts from test.

Eventually an RC makes it all the way through. The finalise step creates `v1.3.0` and the GitHub Release.

The RC history (`rc.1`, `rc.2`, `rc.3`) documents exactly what happened during promotion.

---

## Scenario 3: Hotfix

Production has a critical issue. Main has moved on. Fix what's in prod without pulling in new work.

### Step 1: Check what's in prod

The release branch `release/1.3.0` still exists and contains exactly what's in production (the commit tagged `v1.3.0`).

### Step 2: Fix on the release branch

```bash
git checkout release/1.3.0
git checkout -b hotfix/fix-timeout
# Make the fix
git add .
git commit -m "fix: increase auth timeout to prevent 504s"
git push origin hotfix/fix-timeout
```

Open a PR targeting **release/1.3.0** (not main). Review and merge.

### Step 3: Tag the hotfix RC

1. Go to **Actions → Tag New RC → Run workflow**
2. Enter version: `1.3.0`
3. Click **Run workflow**

> **Note:** The tag will be `v1.3.0-rc.N` (next available number). The finalise step will create `v1.3.0` — but wait, `v1.3.0` already exists from the original release. For a **hotfix**, you need to use a patch version instead.

**For hotfixes, use Cut Release with the patch version:**

1. Manually create the tag instead:
   ```bash
   git checkout release/1.3.0
   git pull origin release/1.3.0
   git tag v1.3.1-rc.1
   git push origin v1.3.1-rc.1
   ```
2. Then trigger Promote manually:
   - Go to **Actions → Promote → Run workflow**
   - Enter version: `v1.3.1-rc.1`

### Step 4: Promote through all environments

The pipeline deploys through test → preprod → prod as usual. Approve at each gate.

The finalise step creates `v1.3.1` and a GitHub Release documenting the hotfix.

### Step 5: Cherry-pick to main

```bash
git checkout main
git cherry-pick <hotfix-commit-sha>
git push origin main
```

This ensures the fix is included in the next release.

---

## Quick Reference

| Action | How |
|--------|-----|
| Cut a release | Actions → **Cut Release** → enter version (e.g., `1.2.0`) |
| Approve promotion | Click paused workflow run → **Review deployments** → approve |
| Fix during promotion | Commit to release branch → Actions → **Tag New RC** → enter version |
| Hotfix production | PR to release branch → `git tag v1.3.1-rc.1` → Actions → **Promote** |
| See what's in prod | **Releases** page → latest release |
| See what changed | Click the release → auto-generated notes |
| See promotion history | Tags: `rc.1`, `rc.2`, ... → final `v1.2.0` |

## Branching Rules

| Branch | Who commits here | Deploys to |
|--------|-----------------|------------|
| `main` | Everyone (via PRs) | dev |
| `release/X.Y.Z` | Fixes only (cherry-picks or direct) | test → preprod → prod |
| `feature/*` | Developer working on a feature | ephemeral (via PR) |
| `hotfix/*` | Developer fixing prod | merged into release branch via PR |
