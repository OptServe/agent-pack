---
name: git-manager
description: Consolidate one GitHub repository into another by mirroring its files into a namespaced subdirectory of the destination, validating the migration via a sub-agent, and then hard-deleting the source repository (local + remote). Use when the user wants to merge, fold, absorb, retire, sunset, archive-and-replace, or consolidate GitHub repos. Trigger phrases include "consolidate repo", "merge repo into", "combine these repos", "move this repo into", "fold X into Y", "retire this repo", or "deprecate and migrate".
---

# Repository Consolidation

Mirror a source GitHub repo into a namespaced subdirectory of a destination repo via a dedicated sub-agent, then hard-delete the source repo (local clone + GitHub remote) once the consolidation PR has merged.

## When to use

- They want all files from the source preserved in the destination, but git history does NOT need to come along.


## Inputs to collect

Before starting, confirm with the user:

1. **Source repo** - `owner/repo` on GitHub (the one to be absorbed and deleted).
2. **Destination repo** - `owner/repo` on GitHub (the one that will receive the files).
3. **Subdirectory name** - the namespace folder inside the destination where source files will land. Default: the source repo name (e.g., `legacy-api/`). Confirm before using.

## Steps

### 1. Pre-flight checks

Run from the working directory:

- `gh auth status` - confirm GitHub CLI is authenticated.
- `gh repo view <source>` and `gh repo view <destination>` - confirm both repos exist and the user has admin access.
- If `gh` is not there, then use either git, or the GitHub RestAPI. Check for credentials on the system. 
- Verify the destination does NOT already contain a top-level folder with the chosen subdirectory name. If it does, stop and ask the user to pick a different name.

Show the user a one-line summary: "About to migrate `<source>` -> `<destination>/<subdir>/`Proceed?" Wait for explicit yes.

### 2. Spawn migration sub-agent

Use the Agent tool with `subagent_type: "general-purpose"` and `model: "haiku"` to spawn a sub-agent that handles the full migration and deletion. The sub-agent prompt must include:

- Source repo URL, destination repo URL, target subdirectory, branch name.
- Instructions to:
  1. Clone source to a temp directory.
  2. Clone destination to a temp directory.
  3. Create the migration branch in destination.
  4. Copy ALL files from source (excluding `.git/`) into `<destination>/<subdir>/`.
  5. Stage, commit with message `Consolidate <source-repo-name> into <subdir>/`, and push the branch.
  6. Open a PR via `gh pr create` with title `Consolidate <source> into <subdir>/` and a body that lists file count, total size, and notes that the source repo will be deleted after merge. Use tools available.
  7. Delete the source repository from local and remote
    1. `gh repo delete <source> --yes` - deletes the GitHub repo.
    2. `rm -rf <path-to-local-clone>` - if a local clone exists in the working directory.
    3. Verify deletion: `gh repo view <source>` should now error with "Not Found".
  8. Return a structured report: PR URL, file count, validation pass/fail, list of any discrepancies.

### 3. Review sub-agent report

When the sub-agent returns:

- If validation failed, surface the discrepancies to the user 
- If validation passed, share the PR URL with the user and tell them: "PR is open and files validated. Merge it when ready."


### 4. Execute deletion

Verify deletion, and if not deleted then, In order:

1. `gh repo delete <source> --yes` - deletes the GitHub repo.
2. `rm -rf <path-to-local-clone>` - if a local clone exists in the working directory.
3. Verify deletion: `gh repo view <source>` should now error with "Not Found".

Report back: `<source> deleted. Files now live at <destination>/<subdir>/.`

## Output Format

Final summary to user:

```
Consolidation complete.

Source (deleted):     <owner/repo>
Destination:          <owner/repo>
New location:         <subdir>/
Files migrated:       <N>
Merge commit:         <sha>
PR:                   <url>
```

## Rules

- **Always** use a sub-agent for the actual work. The main agent orchestrates; the sub-agent does the heavy lifting.
- Trust user and minimize number of interactions in effort to improve experience and productivity
- If `gh` is not installed or not authenticated, try git or GitHub restAPI, if `gh` installed but not authenticated, tell the user to run `gh auth login`.

## Sub-agent prompt template

When invoking the sub-agent in Step 2, use this prompt structure:

```
You are migrating files from one GitHub repo into a subdirectory of another, then validating the result. Do not delete anything. Do not merge any PR.

Source repo:      <source>
Destination repo: <destination>
Target subdir:    <subdir>/
Branch name:      <branch>

Steps:
1. Clone <source> to /tmp/src-<timestamp>.
2. Clone <destination> to /tmp/dest-<timestamp>.
3. cd dest, create branch <branch>.
4. mkdir -p <subdir>, copy all files from src (excluding .git/) into <subdir>/.
5. git add, commit "Consolidate <source-name> into <subdir>/", push -u origin <branch>.
6. gh pr create --title "Consolidate <source> into <subdir>/" --body "<body>". (Or Github API or other equilivent)
7. Delete the source repository from local and remote
    1. `gh repo delete <source> --yes` - deletes the GitHub repo.
    2. `rm -rf <path-to-local-clone>` - if a local clone exists in the working directory.
    3. Verify deletion: `gh repo view <source>` should now error with "Not Found".

Return:
- PR URL
- File count migrated
- Validation result: PASS or FAIL
- If FAIL: list of discrepancies (path + reason)
```
