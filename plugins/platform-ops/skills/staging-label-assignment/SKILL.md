---
name: staging-label-assignment
description: Check available staging slots and assign deploy labels to Loyal Guru API pull requests. Use when a PR needs a staging environment assigned, when a staging label failed to deploy because it was already taken, when the deploy workflow fails with "another PR with deploy_staging found", or when the user asks which staging slots are free, which PR has deploy_staging, or to assign a staging label.
---

# Staging Label Assignment

## Staging slots

The `loyal-guru-api`, `loyal-guru-api-streaming-v2` and `loyal-guru-app-owner-v2` repos have three staging environments controlled by GitHub labels:

- `deploy_staging`
- `deploy_staging2`
- `deploy_staging3`

Only one PR may hold a given label at a time. The GitHub Actions deploy workflow (`deploy.yml`) aborts if another PR already owns the label.

## Check availability (use REST API — not Search API)

**Always** use the REST Issues API — it is real-time. The GitHub Search API has up to 1-2 hours of index lag and can return false positives for labels that were recently removed:

```bash
# REST API — real-time, reliable
gh api "repos/<OWNER>/<REPO>/issues?labels=deploy_staging&state=open&per_page=10" \
  --jq '[.[] | {number, title: .title[:60]}]'

gh api "repos/<OWNER>/<REPO>/issues?labels=deploy_staging2&state=open&per_page=10" \
  --jq '[.[] | {number, title: .title[:60]}]'

gh api "repos/<OWNER>/<REPO>/issues?labels=deploy_staging3&state=open&per_page=10" \
  --jq '[.[] | {number, title: .title[:60]}]'
```

An empty array `[]` means the slot is free. Run all three to pick the lowest-numbered free slot.

> **Do NOT use** `gh pr list --search 'label:deploy_staging'` for availability checks — it uses the Search API which can be stale.

## Assign a free label

Once you identify a free slot:

```bash
gh pr edit <PR_NUMBER> --add-label <LABEL>
```

Draft PRs can receive staging labels and be deployed normally.

## Remove a label

```bash
gh pr edit <PR_NUMBER> --remove-label <LABEL>
```

## Workflow

1. Run the REST API availability check for all three slots.
2. Identify which slots are free (empty array).
3. If a free slot exists → assign it.
4. If all slots are taken → report which PRs hold each slot and tell the user to wait.
5. After assigning, confirm with the REST API check that the label is visible.

---

## Troubleshooting: deploy fails with "another PR with deploy_staging found"

The `label_check@master` action inside `deploy.yml` uses the **GitHub Search API** (`/search/issues?q=label:deploy_staging`) to detect conflicts. This search can lag up to 1-2 hours behind reality — a PR that had a label removed may still appear in results.

When this happens:
- The workflow detects `total_count > 1`, prints `"... another PR with deploy_staging found"`
- It **automatically removes** the label from the current PR
- The deploy is aborted

### Diagnose: is it a lag issue?

Compare the Search API (stale) vs the REST API (real-time) for the affected label:

```bash
REPO="loyalguru/loyal-guru-api"
LABEL="deploy_staging"

echo "=== Search API (may be stale) ==="
gh api "search/issues?q=is:pr+is:open+label:${LABEL}+repo:${REPO}" \
  --jq '{total_count, items: [.items[] | {number, labels: [.labels[].name]}]}'

echo "=== REST API (real-time) ==="
gh api "repos/${REPO}/issues?labels=${LABEL}&state=open&per_page=10" \
  --jq '[.[] | {number, title: .title[:50]}]'
```

- **REST shows 0, Search shows > 0** → confirmed lag issue. Wait for the index to catch up.
- **Both show > 1** → a real conflict exists; check which PR actually holds the label.

### Fix: monitor and retry automatically

When it's a lag issue, run this poller — it re-adds the label as soon as the search index clears:

```bash
REPO="loyalguru/loyal-guru-api"
LABEL="deploy_staging"
PR_NUMBER=<YOUR_PR>

for i in $(seq 1 24); do
  count=$(gh api "search/issues?q=is:pr+is:open+label:${LABEL}+repo:${REPO}" --jq '.total_count')
  echo "$(date -u '+%H:%M:%S') UTC - search index total_count: $count"
  if [ "$count" = "0" ]; then
    echo "Index clear. Adding label..."
    gh pr edit $PR_NUMBER --add-label $LABEL
    echo "Done. Deploy triggered."
    break
  fi
  sleep 300  # check every 5 minutes, up to 2 hours
done
```

> The workflow auto-removes the label on failure, so the poller waits for `total_count = 0` (no PRs at all in the stale index), then re-adds it. At that point, only our PR will appear (`total_count = 1`) and the workflow succeeds.

### Why the lag happens

GitHub's search index is eventually consistent. Removing a label from a PR can take 30 minutes to 2 hours to be reflected in `/search/issues`. The REST Issues API (`/repos/.../issues?labels=...`) is real-time and does not have this problem, but the `label_check@master` action uses the search endpoint.
