---
name: staging-redeploy
description: Re-trigger a staging deployment for a Loyal Guru API pull request by removing and re-adding the deploy label. Use when the user says "re-deploy to staging", "trigger staging again", "redeploy staging1/2/3", or when a deploy needs to be repeated after a code push or failed deploy.
---

# Staging Re-Deploy

Re-deploying to staging means **removing the deploy label and re-adding it**. That is the only trigger for the deploy workflow.

## Prerequisites — branch must be up to date with master

> **Warning**: If the branch is behind master, the deploy workflow fails and **automatically removes the label from the PR**. Once removed, another developer can claim that staging slot for their own PR. Always sync with master before touching the label.

```bash
git fetch origin
git merge origin/master
git push
```

## Re-deploy steps

1. Find which label the PR currently holds:

```bash
gh pr view <PR_NUMBER> --json labels --jq '[.labels[].name]'
```

2. Remove the label:

```bash
gh pr edit <PR_NUMBER> --remove-label <LABEL>
```

3. Re-add the label (triggers the deploy):

```bash
gh pr edit <PR_NUMBER> --add-label <LABEL>
```

That's it. The deploy workflow fires as soon as the label is added.

## After re-deploying

Monitor the deploy via the PR checks page or:

```bash
gh pr checks <PR_NUMBER> --watch
```

The `deploy / deploy (push)` job must succeed. If the workflow removes the label again, check the job logs — the most common cause is the branch being behind master.

## Related skill

For first-time label assignment or when all slots are taken, see the `staging-label-assignment` skill.
