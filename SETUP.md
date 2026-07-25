# Setup: Hub Repo + Onboarding Existing Repos

This repo is the **hub**: `.github/workflows/*.yml` are reusable workflows
(`workflow_call`) that hold all the actual agent logic. Every repo you want
the pipeline running in (a **consumer repo**) gets a handful of thin caller
workflows from `templates/consumer-repo/` that just point at this hub. Update
a stage once here, every consumer repo picks it up on its next run — no
copy-paste drift across repos.

## Part 1 — One-time hub setup

1. **Push this repo to GitHub** (e.g. `github.com/<your-org>/agent-pipeline`).
2. **Find-and-replace the placeholder.** Every file under `templates/consumer-repo/`
   has `uses: <YOUR_ORG>/agent-pipeline/.github/workflows/...@main` — replace
   `<YOUR_ORG>/agent-pipeline` with the real path from step 1.
   ```
   grep -rl '<YOUR_ORG>/agent-pipeline' templates/ | xargs sed -i '' 's#<YOUR_ORG>/agent-pipeline#your-org/agent-pipeline#g'
   ```
3. **If the hub repo is private**: consumer repos in the same org can't call
   its reusable workflows by default. Go to the hub repo's
   **Settings → Actions → General → Access**, and either:
   - "Accessible from repositories in the '`<org>`' organization" (simplest if
     everything is in one org), or
   - Add the specific consumer repos to the allow-list.
   If the hub is public, this step is a no-op.
4. **Create org-level secrets/variables once** (org Settings → Secrets and
   variables → Actions), scoped to "selected repositories" = hub + every
   consumer repo you onboard. See Part 2 for how to generate the values:
   - Secret `ANTHROPIC_API_KEY`
   - Secret `GH_AUTOMATION_TOKEN`
   - (Optional, only if syncing Projects v2) Variables `PROJECT_OWNER`,
     `PROJECT_NUMBER`, and if they differ from the defaults,
     `PROJECT_STATUS_FIELD_NAME` / `PROJECT_DONE_OPTION_NAME`.
   Doing this at the org level means step 4 in Part 3 below is a checkbox,
   not a re-entry, for every new repo you onboard.

## Part 2 — Generating the tokens

### `ANTHROPIC_API_KEY`

1. Go to https://console.anthropic.com.
2. Create a dedicated **Workspace** for this automation (e.g. "github-agent-pipeline")
   rather than reusing a personal one — isolates cost/usage and makes it a
   one-click revoke if something misbehaves across many repos.
3. In that workspace: Settings → API Keys → Create Key. Name it
   `github-actions-agent-pipeline`.
4. If your plan supports it, set a spend limit / budget alert on the
   workspace — autonomous agents across N repos can burn through quota
   faster than a human would notice.
5. Copy the key once, store it as the org secret `ANTHROPIC_API_KEY` (Part 1,
   step 4).

### `GH_AUTOMATION_TOKEN` — fine-grained PAT

*(This is the fast-start path. It's tied to your personal GitHub account —
fine for getting several repos running now; consider migrating to a GitHub
App later if the person who owns this token ever changes or the token needs
to outlive one person. That migration is a small, contained change: swap the
`secrets.GH_AUTOMATION_TOKEN` references for a token minted via
`actions/create-github-app-token` — ask if you want that done later.)*

1. Go to https://github.com/settings/personal-access-tokens/new
2. **Resource owner**: your org (so it can reach every consumer repo you
   pick, not just your personal repos).
3. **Expiration**: set a real date (e.g. 90 days) rather than "no expiration"
   — put a reminder on your calendar to rotate it. A silently-expired token
   just makes the pipeline stall at whichever stage needs it; nothing pages
   you.
4. **Repository access**: "Only select repositories" → pick every consumer
   repo you're onboarding now (you can add more later by editing the token).
5. **Permissions** (Repository permissions):
   - Contents: **Read and write**
   - Issues: **Read and write**
   - Pull requests: **Read and write**
   - Metadata: Read-only (auto-selected)
6. **Projects v2 sync (optional)**: fine-grained PATs can grant org Projects
   access only if an org owner approves it, and it's occasionally flaky to
   configure. If it's not available/greyed out for you, generate a *second*,
   narrower **classic** PAT
   (https://github.com/settings/tokens/new) with just the `project` scope,
   and use that one only for the Projects sync step in `06-close-out.yml`
   (store as a separate secret, e.g. `GH_PROJECTS_TOKEN`, and swap it in for
   `GH_AUTOMATION_TOKEN` in that one script block).
7. Generate, copy immediately (shown once), store as the org secret
   `GH_AUTOMATION_TOKEN` (Part 1, step 4).

## Part 3 — Onboarding an existing repo

Repeat for each repo:

1. Copy the contents of `templates/consumer-repo/` into the repo root
   (i.e. its `.github/ISSUE_TEMPLATE/`, `.github/workflows/`, `.claude/`
   land alongside whatever the repo already has — check for filename
   collisions with existing workflows first, the numeric prefixes make that
   unlikely).
2. Edit that repo's new `.claude/pipeline.config.yml` with its real
   install/build/test/lint commands and any `protected_paths`.
3. Confirm the **Claude GitHub App** has access to this repo:
   https://github.com/settings/installations → Claude → Configure → add the
   repository. (Separate from `GH_AUTOMATION_TOKEN` — the App is what runs
   the `claude-code-action` steps; the PAT is what lets bot actions chain
   workflows.)
4. If you did org-level secrets/variables in Part 1, nothing to do here. If
   this repo is outside that scope, add `ANTHROPIC_API_KEY` and
   `GH_AUTOMATION_TOKEN` as repo-level secrets instead.
5. From the repo's **Actions** tab, run "Agent Pipeline: Setup Labels" once
   (`workflow_dispatch`) to create the label set.
6. **Projects v2 auto-add (optional)**: in the target Project → `...` menu →
   Workflows → "Auto-add to project", filter on the `spec` label. This is a
   native Projects v2 feature — nothing to configure in this repo for it.
7. Check the repo's branch protection on its default branch doesn't block a
   new branch being pushed (protection usually only gates *merging*, which
   is untouched here since merges stay human — see README).
8. **Dry run**: file a test issue using the "Feature / Change Spec"
   template, and walk it through `/approve-plan` to confirm the whole chain
   fires before trusting it on real work.

## Updating the pipeline later

Change a prompt or add a stage once in this hub repo's `.github/workflows/`.
Every consumer repo picks it up automatically on its next trigger (they
reference `@main`, not a pinned copy) — no need to touch consumer repos
unless the *caller* shape itself changes (new trigger event, new label, new
required secret), in which case update `templates/consumer-repo/` and
re-copy that one file into each consumer repo.
