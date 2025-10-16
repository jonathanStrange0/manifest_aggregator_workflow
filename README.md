# Aggregating Manifests and SBOMs Across Repositories

This repository contains a reusable GitHub Actions workflow that clones a defined list of repositories, collects dependency manifests and SBOM files, writes them into `manifests/`, commits any changes, and triggers an optional downstream analysis.

Use this README to set it up in your own GitHub repository.

---

## What this workflow does

- Checks out the aggregator repo that holds this workflow.
- Clones a list of source repositories read only.
- Copies common manifest and SBOM files into `manifests/<repo_name>/`.
- Generates `manifests/metadata.json` with a timestamp and the list of repositories scanned.
- Commits and pushes changes when differences exist.
- Optionally fires a `repository_dispatch` event named `manifest_update` that you can catch in other workflows.
- Runs on a daily schedule and on manual dispatch. You can also keep or remove push triggers.

---

## Repository layout

```
.
├── .github/
│   └── workflows/
│       └── aggregate-manifests.yml   # the workflow in this README
└── manifests/                        # created by the workflow at runtime
```

---

## Prerequisites

1) **GitHub CLI token for cloning source repos**

- Create a Personal Access Token that can read the repositories you want to aggregate.
  - Prefer a Fine Grained PAT with the following:
    - Resource owner: your org or user
    - Repositories: only the repos you will scan
    - Repository permissions: Contents: Read
  - Classic PAT works too with `repo` scope, but restrict to least privilege when possible.

2) **Actions and token permissions**

- Repo Settings → Actions → General
  - Actions permissions: Allow all actions and reusable workflows
  - Workflow permissions: Read and write

3) **Optional org approvals**

- If your org requires approval for GitHub Apps or Actions permissions, get those approvals first.

---

## Quick start

1) **Create or choose a repository** that will host the aggregated output and the workflow.

2) **Add the secret** used by the GitHub CLI inside the workflow:

- Settings → Secrets and variables → Actions → New repository secret
  - Name: `GHPAT_TOKEN`
  - Value: your PAT

3) **Create the workflow file** at `.github/workflows/aggregate-manifests.yml` with the content below.

4) **Edit the `REPOS=(...)` array** to list the repositories you want to scan. You can generate it with the CLI snippets in this README.

5) **Run it**

- Manual: Actions tab → Select the workflow → Run workflow.
- Scheduled: The default cron runs daily at 02:00 UTC.

---

## The workflow

> Copy this into `.github/workflows/aggregate-manifests.yml` and tailor `REPOS` and the schedule.

```yaml
name: Aggregate_Manifest_and_SBOM_Files

on:
  schedule:
    - cron: '0 2 * * *'   # daily at 02:00 UTC
  workflow_dispatch:       # allow manual runs

permissions:
  contents: write

concurrency:
  group: aggregate-manifests
  cancel-in-progress: false

jobs:
  aggregate-manifests:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout aggregator repo
        uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Setup directories
        run: |
          mkdir -p manifests
          rm -rf manifests/*

      - name: Clone and copy manifests
        env:
          GH_TOKEN: ${{ secrets.GHPAT_TOKEN }}   # PAT with read access to source repos
        run: |
          REPOS=(
            "org-or-user/repo-one"
            "org-or-user/repo-two"
          )

          for repo in "${REPOS[@]}"; do
            echo "Processing $repo..."
            repo_name=$(echo "$repo" | cut -d'/' -f2)
            gh repo clone "$repo" "/tmp/$repo_name" -- --depth 1
            mkdir -p "manifests/$repo_name"

            # JavaScript and Node
            find "/tmp/$repo_name" -name "package.json" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true
            find "/tmp/$repo_name" -name "package-lock.json" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true
            find "/tmp/$repo_name" -name "yarn.lock" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true
            find "/tmp/$repo_name" -name "pnpm-lock.yaml" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true

            # Python
            find "/tmp/$repo_name" -name "requirements.txt" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true
            find "/tmp/$repo_name" -name "Pipfile" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true
            find "/tmp/$repo_name" -name "Pipfile.lock" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true
            find "/tmp/$repo_name" -name "pyproject.toml" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true
            find "/tmp/$repo_name" -name "setup.py" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true
            find "/tmp/$repo_name" -name "environment.yml" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true

            # Go
            find "/tmp/$repo_name" -name "go.mod" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true
            find "/tmp/$repo_name" -name "go.sum" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true

            # Java and Kotlin
            find "/tmp/$repo_name" -name "pom.xml" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true
            find "/tmp/$repo_name" -name "build.gradle" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true
            find "/tmp/$repo_name" -name "build.gradle.kts" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true
            find "/tmp/$repo_name" -name "gradle.lockfile" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true

            # Ruby
            find "/tmp/$repo_name" -name "Gemfile" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true
            find "/tmp/$repo_name" -name "Gemfile.lock" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true
            find "/tmp/$repo_name" -name "*.gemspec" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true

            # .NET
            find "/tmp/$repo_name" -name "*.csproj" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true
            find "/tmp/$repo_name" -name "packages.config" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true
            find "/tmp/$repo_name" -name "Directory.Packages.props" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true

            # Scala
            find "/tmp/$repo_name" -name "build.sbt" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true
            find "/tmp/$repo_name" -name "plugins.sbt" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true
            find "/tmp/$repo_name" -name "build.properties" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true

            # Rust
            find "/tmp/$repo_name" -name "Cargo.toml" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true
            find "/tmp/$repo_name" -name "Cargo.lock" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true

            # C and C++
            find "/tmp/$repo_name" -name "CMakeLists.txt" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true
            find "/tmp/$repo_name" -name "conanfile.txt" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true
            find "/tmp/$repo_name" -name "conanfile.py" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true
            find "/tmp/$repo_name" -name "vcpkg.json" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true

            # PHP
            find "/tmp/$repo_name" -name "composer.json" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true
            find "/tmp/$repo_name" -name "composer.lock" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true

            # Swift and Objective-C
            find "/tmp/$repo_name" -name "Podfile" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true
            find "/tmp/$repo_name" -name "Podfile.lock" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true
            find "/tmp/$repo_name" -name "Package.swift" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true
            find "/tmp/$repo_name" -name "Package.resolved" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true

            # Elixir
            find "/tmp/$repo_name" -name "mix.exs" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true
            find "/tmp/$repo_name" -name "mix.lock" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true

            # SBOM and Socket facts
            find "/tmp/$repo_name" -name "*.cdx.json" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true
            find "/tmp/$repo_name" -name "*.spdx.json" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true
            find "/tmp/$repo_name" -name "socket.facts.json" -exec cp {} "manifests/$repo_name/" \; 2>/dev/null || true

            rm -rf "/tmp/$repo_name"
          done

          # Make the repo list available to the metadata step
          printf '%s\n' "${REPOS[@]}" > repos.list

      - name: Create metadata
        run: |
          repos_json=$(jq -R -s 'split("\n") | map(select(length>0))' < repos.list)
          cat > manifests/metadata.json << EOF
          {
            "last_updated": "$(date -u +"%Y-%m-%dT%H:%M:%SZ")",
            "repositories": $repos_json
          }
          EOF

      - name: Configure Git and Push Changes
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add manifests/
          if git diff --staged --quiet; then
            echo "No changes to commit"
            exit 0
          fi
          git commit -m "Update manifests - $(date -u +"%Y-%m-%d %H:%M:%S UTC")"
          git pull --rebase origin main || git pull origin main
          git push

      - name: Trigger GitHub App analysis
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          gh api \
            --method POST \
            -H "Accept: application/vnd.github+json" \
            /repos/${{ github.repository }}/dispatches \
            -f event_type='manifest_update'
```

---

## Choosing which repositories to scan

You can generate a ready to paste Bash array with the GitHub CLI. Replace `OWNER` with your org or username.

```bash
gh repo list OWNER --limit 1000 \
  --json nameWithOwner,isArchived,isFork \
  --jq '"REPOS=(\n"
        + (map(select(.isArchived==false and .isFork==false)
               | "  \"" + .nameWithOwner + "\"")
           | join("\n"))
        + "\n)"'
```

To load directly into an array in your shell and print the block:

```bash
readarray -t REPOS < <(
  gh repo list OWNER --limit 1000 \
    --json nameWithOwner,isArchived,isFork \
    --jq '.[] | select(.isArchived==false and .isFork==false) | .nameWithOwner'
)

printf 'REPOS=(\n'; printf '  "%s"\n' "${REPOS[@]}"; printf ')\n'
```

Filters you may want:

- Only public: `gh repo list OWNER --public ...`
- Only private: `gh repo list OWNER --private ...`
- Name regex filter example inside `jq`:

```bash
gh repo list OWNER --limit 1000 \
  --json nameWithOwner,isArchived,isFork \
  --jq 'map(select(.isArchived==false and .isFork==false)
             | select(.nameWithOwner | test("web|api|mobile")))
        | .[] | .nameWithOwner'
```

---

## Scheduling

- The default schedule is daily at 02:00 UTC: `0 2 * * *`.
- Change it by editing the cron expression under `on.schedule`.
- Schedules are paused if no repository activity occurs for long periods. Any push or manual run reactivates them.

Common patterns:

- Twice per day at 02:00 and 14:00 UTC

```yaml
on:
  schedule:
    - cron: '0 2,14 * * *'
```

- Weekdays only at 02:00 UTC

```yaml
on:
  schedule:
    - cron: '0 2 * * 1-5'
```

---

## Using the repository_dispatch event

This workflow sends a `repository_dispatch` with `event_type: manifest_update`. You can catch it with another workflow in the same repo or a different repo that has a GitHub App or token configured.

Example receiver in the same repo:

```yaml
name: Downstream on manifest update

on:
  repository_dispatch:
    types: [manifest_update]

jobs:
  react:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Manifests updated. Kick off analysis here."
```

---

## Output and metadata

- All collected files are placed under `manifests/<repo_name>/`.
- `manifests/metadata.json` has this shape:

```json
{
  "last_updated": "2025-01-01T00:00:00Z",
  "repositories": [
    "owner/repo-one",
    "owner/repo-two"
  ]
}
```

---

## Troubleshooting

- **No changes to commit**: Either nothing new was found or the prior run already committed the latest content.
- **Permission denied when cloning**: Confirm `GHPAT_TOKEN` exists, is spelled correctly, and can read the source repos. For private org repos, the fine grained PAT must be granted access to each repo.
- **GITHUB_TOKEN cannot push**: Check repo Settings → Actions → General → Workflow permissions. Set to Read and write. You already set `permissions: contents: write` in the workflow.
- **Scheduled runs never fire**: Make a manual Run workflow. Check org policy for scheduled workflows. GitHub may pause schedules in inactive repos.
- **Conflicts on push**: The workflow pulls with rebase before pushing. If it still fails, the next run will succeed after the merge created by the fallback `git pull origin main`.
- **`gh` command not found**: `ubuntu-latest` includes GitHub CLI. If you change runners, ensure `gh` is available.

---

## Security notes

- Use a limited scope Fine Grained PAT for cloning source repos. Avoid broad classic tokens when possible.
- Do not write secrets into the repository. All sensitive values must stay in Actions secrets.
- Consider running on a private repository to avoid exposing third party manifests.
- Review the list of file patterns to ensure you are not copying sensitive files.

---

## Socket.dev

Once you have this workflow copying the desired manifest files, you can setup [Socket.dev](https://socket.dev) via the [GitHub App](https://docs.socket.dev/docs/socket-for-github) to watch this repo for commits and PRs.
