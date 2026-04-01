# Candidate B: Trunk-Based + Release Branches

Trunk-based development with release branches and tag-driven promotion through environment gates.

## How It Works

### Standard Release

```
         main (development)              release/1.2.0 (promotion)
         ==================              ===========================

    feature branch
    ──────┐
          ▼
  ──A──B──C──D──E──F──────►             D (branch cut here)
              │                          │
              │  Cut Release 1.2.0       ├── v1.2.0-rc.1
              └─────────────────────►    │     │
                                         │     ├── auto → test
         (dev continues on main)         │     ├── approval → preprod
                                         │     ├── approval → prod
  ──G──H──I──J──────────────►            │     │
                                         │     ├── v1.2.0 (final tag)
                                         │     ├── GitHub Release created
                                         │     └── merge-back issue (if fixes were made)
```

### Fix During Promotion

```
  release/1.3.0:   ──D──fix──fix2──
                      │    │     │
                    rc.1  rc.2  rc.3 ──► test → preprod → prod → v1.3.0
                   (fail)       (success)

  After finalise: merge-back issue created automatically
```

### Hotfix

```
  v1.3.0 tag ◄── immutable snapshot of what's in prod
       │
       └── Hotfix workflow creates release/1.3.1 from this tag
            │
            ├── push fix
            ├── Tag New RC → v1.3.1-rc.1
            ├── test → preprod → prod
            ├── v1.3.1 (final tag)
            └── merge-back issue created automatically

  If the hotfix is bad: abandon release/1.3.1, run Hotfix again → creates release/1.3.2
  The v1.3.0 tag is protected and untouched.
```

### Pipeline Flow

```
 ┌──────────────┐         ┌─────────────────────────────────────────────────────────┐
 │  Cut Release  │         │              Promote (auto-triggered)                   │
 │  (from main)  │         │                                                         │
 │               │ triggers│  ┌──────┐  ┌─────────┐  ┌──────┐  ┌─────────────────┐ │
 │ Creates:      │────────►│  │ Test │─►│ Preprod  │─►│ Prod │─►│ Finalise        │ │
 │ - branch      │         │  │(auto)│  │(approval)│  │(approval)│- tag vX.Y.Z   │ │
 │ - rc.1 tag    │         │  └──────┘  └─────────┘  └──────┘  │- GitHub Release │ │
 └──────────────┘         │                                     │- merge-back     │ │
                           │                                     │  issue (if needed)│
 ┌──────────────┐         │                                     └─────────────────┘ │
 │  Tag New RC   │ triggers│                                                         │
 │  (from        │────────►│  (pipeline restarts from test, cancels previous run)    │
 │  release/*)   │         │                                                         │
 └──────────────┘         └─────────────────────────────────────────────────────────┘

 ┌──────────────┐
 │  Hotfix       │  Creates a new release/X.Y.Z branch from a release tag.
 │  (from main)  │  Then: push fix → Tag New RC → same pipeline as above.
 └──────────────┘
```

## Key Concepts

| Concept | What it means |
|---------|---------------|
| **main** | Where all development happens. Never stops. |
| **release/X.Y.0** | Cut from main when ready to release. Lives ~1 week during promotion. |
| **release/X.Y.Z** (Z>0) | Hotfix branch, cut from a release tag. Isolated from the original. |
| **v1.2.0-rc.N** | Release candidate tag. Each RC triggers the promotion pipeline. |
| **v1.2.0** | Final release tag. Created automatically when an RC reaches prod. |
| **GitHub Release** | Created only when deployed to prod. This IS the release. |
| **Merge-back issue** | Created automatically at finalise if the release branch has fixes that aren't on main. |

## Environments

| Environment | Gate | Who approves |
|-------------|------|-------------|
| test | Automatic | Nobody — deploys immediately |
| preprod | Required reviewer | Configured in repo settings |
| prod | Required reviewer | Configured in repo settings |

## Workflows

| Workflow | Run from branch | Trigger | Purpose |
|----------|----------------|---------|---------|
| **Cut Release** | `main` | Manual | Creates release branch + first RC tag, triggers Promote |
| **Tag New RC** | `release/*` | Manual | Creates next RC tag after a fix, triggers Promote |
| **Hotfix** | `main` | Manual | Creates a new patch release branch from a release tag |
| **Promote** | `release/*` | Auto-triggered | Deploys test → preprod → prod, finalises release |
| **Deploy** | `release/*` | Called by Promote | Simulates deployment to one environment |

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
4. **Finalise** — automatically creates tag `v1.2.0`, GitHub Release, and checks if merge-back is needed (it won't be for a clean release)

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
4. **Do NOT approve preprod** — it will be cancelled when a new RC is tagged

### Step 2: Fix the issue on the release branch

```bash
git checkout release/1.3.0
git pull origin release/1.3.0
# Make the fix
git add environments/test.json
git commit -m "fix: correct api endpoint in test config"
git push origin release/1.3.0
```

### Step 3: Tag a new RC and restart promotion

1. Go to **Actions → Tag New RC**
2. In the branch dropdown, **select `release/1.3.0`** (not main)
3. Enter version: `1.3.0`
4. Click **Run workflow**

This auto-detects the next RC number (e.g., `rc.2`), tags it, and triggers Promote on the release branch. The new promote run **automatically cancels** the previous one. The pipeline restarts from test with the fix included.

### Step 4: Approve through environments

If another issue is found at preprod, fix it, run **Tag New RC** again (creates `rc.3`), and the pipeline restarts from test.

Eventually an RC makes it all the way through. The finalise step creates `v1.3.0` and the GitHub Release.

### Step 5: Merge back

Because fixes were made on the release branch, the finalise step automatically:
- Annotates the GitHub Release with a merge-back notice
- Creates a **merge-back issue** with step-by-step instructions

Follow the instructions in the issue to merge the fixes back to main.

---

## Scenario 3: Hotfix

Production has a critical issue. Main has moved on with new work. Fix what's in prod without pulling in new features.

### Step 1: Create a hotfix branch

1. Go to **Actions → Hotfix → Run workflow**
2. Enter base version: `1.3.0` (the version currently in prod)
3. Click **Run workflow**

This creates `release/1.3.1` branched from the `v1.3.0` tag — an isolated copy of exactly what's in prod.

### Step 2: Push your fix

```bash
git fetch origin
git checkout release/1.3.1
# Make the fix
git add .
git commit -m "fix: increase auth timeout to prevent 504s"
git push origin release/1.3.1
```

Or via a PR targeting `release/1.3.1`.

### Step 3: Start promotion

1. Go to **Actions → Tag New RC**
2. In the branch dropdown, **select `release/1.3.1`** (not main)
3. Enter version: `1.3.1`
4. Click **Run workflow**

This creates `v1.3.1-rc.1` and triggers the promote pipeline on the release branch.

### Step 4: Approve through all environments

Same flow as always:
1. Test — automatic
2. Preprod — approve
3. Prod — approve
4. Finalise — creates `v1.3.1` and GitHub Release

### Step 5: Merge back

The finalise step automatically creates a **merge-back issue** with instructions to merge the hotfix back to main. Follow the instructions in the issue.

---

## Merge-Back Process

When a release has fixes that aren't on main (scenarios 2 and 3), the finalise step creates an issue with these instructions:

```bash
git fetch origin
git checkout main
git pull origin main
git checkout -b merge-back/1.3.0
git merge origin/release/1.3.0
# Resolve conflicts if any
git push origin merge-back/1.3.0
```

Then open a PR from `merge-back/1.3.0` to `main`, review, and merge. Close the issue once done.

This process is the same whether it's a standard release with fixes or a hotfix. The release branch is never modified — the merge happens on a temporary branch off main.

---

## Quick Reference

| Action | How |
|--------|-----|
| Cut a release | Actions → **Cut Release** (from main) → enter version |
| Approve promotion | Click paused workflow run → **Review deployments** → approve |
| Fix during promotion | Push fix to release branch → Actions → **Tag New RC** (from release branch) → enter version |
| Start a hotfix | Actions → **Hotfix** (from main) → enter base version currently in prod |
| Promote a hotfix | Push fix to hotfix branch → Actions → **Tag New RC** (from hotfix branch) → enter version |
| Merge back fixes | Follow the merge-back issue created by finalise |
| See what's in prod | **Releases** page → latest release |
| See what changed | Click the release → auto-generated notes |
| See promotion history | Tags: `rc.1`, `rc.2`, ... → final `v1.2.0` |

## Branching Rules

| Branch | Who commits here | Deploys to |
|--------|-----------------|------------|
| `main` | Everyone (via PRs) | dev |
| `release/X.Y.0` | Fixes only during promotion | test → preprod → prod |
| `release/X.Y.Z` (Z>0) | Hotfix only | test → preprod → prod |
| `feature/*` | Developer working on a feature | ephemeral (via PR) |
