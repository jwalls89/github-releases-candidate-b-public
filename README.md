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
 ┌──────────────┐     ┌─────────────────────────────────────────────────┐
 │  Cut Release  │     │              Promote (tag-triggered)            │
 │  (manual)     │     │                                                 │
 │               │     │  ┌──────┐  ┌─────────┐  ┌──────┐  ┌─────────┐ │
 │ Creates:      │────►│  │ Test │─►│ Preprod  │─►│ Prod │─►│Finalise │ │
 │ - branch      │ tag │  │(auto)│  │(approval)│  │(approval)│(auto)  │ │
 │ - rc.1 tag    │push │  └──────┘  └─────────┘  └──────┘  └─────────┘ │
 └──────────────┘     │                                                 │
                       │  Creates: final tag + GitHub Release            │
                       └─────────────────────────────────────────────────┘
```

### Key Concepts

| Concept | What it means |
|---------|---------------|
| **main** | Where all development happens. Never stops. |
| **release/X.Y.Z** | Cut from main when ready to release. Lives ~1 week. |
| **v1.2.0-rc.N** | Release candidate tag. Triggers the promotion pipeline. |
| **v1.2.0** | Final release tag. Created automatically when RC reaches prod. |
| **GitHub Release** | Created only at prod. This IS the release. |

### Environments

| Environment | Gate | Who approves |
|-------------|------|-------------|
| test | Automatic | Nobody — deploys on tag push |
| preprod | Required reviewer | Configured in Settings → Environments |
| prod | Required reviewer | Configured in Settings → Environments |

---

## Setup

Before running scenarios, configure GitHub Environments:

1. Go to **Settings → Environments**
2. Create environment `test` — no protection rules needed
3. Create environment `preprod`:
   - Tick **Required reviewers**
   - Add yourself (or your team)
   - Save protection rules
4. Create environment `prod`:
   - Tick **Required reviewers**
   - Add yourself (or your team)
   - Save protection rules

---

## Scenario 1: Happy Path

A clean release — no fixes needed, straight through to prod.

### Step 1: Cut the release

1. Go to **Actions → Cut Release → Run workflow**
2. Enter version: `1.2.0`
3. Click **Run workflow**

This creates branch `release/1.2.0` and tag `v1.2.0-rc.1`.

### Step 2: Watch it flow

The tag push **automatically triggers** the Promote workflow:

1. **Test** — deploys automatically, no approval needed
2. **Preprod** — workflow pauses, waiting for your approval
   - You'll see a yellow "Waiting" badge on the workflow run
   - Click **Review deployments** → tick `preprod` → **Approve and deploy**
3. **Prod** — workflow pauses again
   - Click **Review deployments** → tick `prod` → **Approve and deploy**
4. **Finalise** — automatically creates tag `v1.2.0` and a GitHub Release with auto-generated notes

### Step 3: Verify

- Check **Releases** page — `v1.2.0` should be the latest release
- Check **Settings → Environments** — each environment shows its deployment history
- Check **Code → Tags** — you should see `v1.2.0-rc.1` and `v1.2.0`

---

## Scenario 2: Unpolished Path

Release hits problems during promotion. Fix, re-tag, restart.

### Step 1: Cut and start promotion

1. **Cut Release** with version `1.3.0`
2. The Promote workflow triggers automatically
3. Test deploys... but something is wrong (bad config value, etc.)
4. **Do NOT approve preprod** — let the workflow sit (or it fails at test)

### Step 2: Fix the issue

**Option A — Fix on main first (preferred):**

```bash
# Fix on main via a normal PR
git checkout main
git checkout -b fix/test-config
# Fix environments/test.json
git add environments/test.json
git commit -m "fix: correct api endpoint in test config"
git push origin fix/test-config
# Open PR to main, review, merge

# Cherry-pick the fix to the release branch
git checkout release/1.3.0
git cherry-pick <commit-sha-from-main>
git push origin release/1.3.0
```

**Option B — Fix directly on release branch (when urgent):**

```bash
git checkout release/1.3.0
# Fix environments/test.json
git add environments/test.json
git commit -m "fix: correct api endpoint in test config"
git push origin release/1.3.0

# IMPORTANT: cherry-pick back to main afterwards
git checkout main
git cherry-pick <commit-sha-from-release>
git push origin main
```

### Step 3: Tag a new RC

```bash
git checkout release/1.3.0
git tag v1.3.0-rc.2
git push origin v1.3.0-rc.2
```

This triggers a **new** Promote workflow run. The pipeline starts from test again.

### Step 4: Approve through environments

1. Test deploys automatically — passes this time
2. Approve preprod — if another issue is found, fix it, tag `rc.3`, repeat
3. Approve prod
4. Finalise creates `v1.3.0` and GitHub Release

The RC history (`rc.1`, `rc.2`, `rc.3`) documents exactly what happened.

---

## Scenario 3: Hotfix

Production has a critical issue. Main has moved on. Fix what's in prod without pulling in new work.

### Step 1: Check what's in prod

The release branch `release/1.3.0` still exists and represents exactly what's in production.

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

### Step 3: Tag the hotfix

```bash
git checkout release/1.3.0
git pull origin release/1.3.0
git tag v1.3.1-rc.1
git push origin v1.3.1-rc.1
```

Note the **patch version bump**: `1.3.0` → `1.3.1`.

### Step 4: Promote through all environments

The tag push triggers the Promote workflow. Same flow as always:
1. Test — automatic
2. Preprod — approve
3. Prod — approve
4. Finalise — creates `v1.3.1` and GitHub Release

### Step 5: Cherry-pick to main

```bash
git checkout main
git cherry-pick <hotfix-commit-sha>
git push origin main
```

This ensures the fix is in the next release.

---

## Quick Reference

| Action | How |
|--------|-----|
| Cut a release | Actions → **Cut Release** → enter version |
| Approve promotion | Click **Review deployments** on the paused workflow |
| Fix during promotion | Commit to release branch → `git tag vX.Y.Z-rc.N` → `git push origin --tags` |
| Hotfix production | PR to release branch → tag patch RC → promotes through all envs |
| See what's in prod | **Releases** page → latest release |
| See what changed | Click the release → auto-generated notes |
| See promotion history | Tags: `rc.1`, `rc.2`, ... → `v1.2.0` |

## Branching Rules

| Branch | Who commits here | Deploys to |
|--------|-----------------|------------|
| `main` | Everyone (via PRs) | dev |
| `release/X.Y.Z` | Fixes only (cherry-picks or direct) | test → preprod → prod |
| `feature/*` | Developer working on a feature | ephemeral (via PR) |
| `hotfix/*` | Developer fixing prod | merged into release branch via PR |
