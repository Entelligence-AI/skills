---
name: entloop
description: >
  Iteratively improves a GitHub pull request until EntelligenceAI gives it a 5/5 confidence score
  ("Safe to Merge") with zero unresolved review threads. Triggers an Entelligence review, fixes all
  actionable comments using each comment's "Prompt to fix with AI" and committable suggestions,
  resolves the threads, pushes, re-triggers the review, and repeats. Use when the user wants to fully
  optimize a PR against EntelligenceAI's code review standards before merging.
license: MIT
compatibility: >
  Requires git and gh (GitHub CLI) authenticated, and the EntelligenceAI PR review app installed on
  the repo (https://github.com/apps/entelligence-ai-pr-reviews). GitHub only.
metadata:
  author: Entelligence AI
  version: "1.0"
allowed-tools: Bash(gh:*) Bash(git:*)
---

# Entloop

Iteratively fix a GitHub PR until EntelligenceAI gives a perfect review: 5/5 confidence
("Safe to Merge"), zero unresolved review threads.

EntelligenceAI posts two things on a PR:

1. A single **summary issue comment** titled `## EntelligenceAI PR Summary` that it edits in place
   on every review. Its confidence-score block is delimited by an `<!-- CONFIDENCE_SCORE -->` marker
   and renders as a line like `## Confidence Score: 3/5 - Review Recommended`.
2. **Inline review comments** (resolvable review threads) authored by the bot, each carrying a
   severity badge (`CRITICAL` / `MAJOR` / `NIT`), an optional committable `suggestion` block, and a
   collapsible "Prompt to fix with AI" block you can apply directly.

The confidence score is computed from the inline findings (severity counts and how many are
unresolved). Drive the score to 5/5 by fixing and resolving every actionable thread, then
re-triggering.

## Inputs

- **PR number** (optional): if not provided, detect the PR for the current branch.

## Score labels

| Score | Label              | Meaning                         |
| ----- | ------------------ | ------------------------------- |
| 5/5   | Safe to Merge      | done                            |
| 4/5   | Mostly Safe        | minor issues remain             |
| 3/5   | Review Recommended | meaningful issues to fix        |
| 2/5   | Changes Needed     | several issues                  |
| 1/5   | Blocking Issues    | must not merge                  |

## Progress output

Keep the loop legible. Print short, human-readable lines; never paste raw check-run or comment JSON
into the conversation. At the start of each iteration, print a compact status block:

```
Entloop iteration 2/5
  Review:     completed (Entelligence Review check)
  Confidence: 4/5 - Mostly Safe
  Threads:    3 unresolved (2 MAJOR, 1 NIT)
  Fixing:     pr_reviewer/utils.py:244, ai_chat/handler.py:88
```

While waiting, one "Waiting for ..." line per poll is enough. The snippets below route command
errors to `/dev/null` on purpose - do not surface a transient API hiccup as output. Save the full
per-finding breakdown for the final report.

## Instructions

### 1. Identify the PR

```bash
gh pr view --json number,headRefName,headRefOid \
  -q '{number: .number, branch: .headRefName, sha: .headRefOid}'
```

Switch to the PR branch if you are not already on it. All `gh api` calls below use the `{owner}` and
`{repo}` placeholders, which `gh` resolves from the current repo's `origin` remote.

The EntelligenceAI bot logins differ between environments. Match the author login
case-insensitively against `entelligence` so the skill works against either:

- `entelligence-ai-pr-reviews[bot]` (production)
- `entelligenceai-pr-review-alpha[bot]` (alpha/staging)

### 2. Loop

Repeat the following cycle. **Max 5 iterations** to avoid runaway loops.

#### A. Trigger an Entelligence review

First push any pending local commits:

```bash
git push
```

Capture a baseline so you can tell when the review has refreshed. Record the current HEAD SHA and
the newest timestamp on any existing Entelligence comment:

```bash
HEAD_SHA=$(gh pr view <PR_NUMBER> --json headRefOid -q .headRefOid)

BASELINE_TS=$(gh api --paginate "repos/{owner}/{repo}/issues/<PR_NUMBER>/comments?per_page=100" \
  | jq -s 'add | [ .[] | select(.user.login | test("entelligence"; "i")) | .updated_at ] | max // ""')
```

EntelligenceAI auto-reviews new commits when `review_on_every_push` is enabled (the default), but
that can be turned off per repo. To guarantee a fresh review, check whether one is already running,
and if not, post the manual trigger comment. The "Entelligence Review" check run shows the live
state:

```bash
ENT_STATE=$(gh api "repos/{owner}/{repo}/commits/$HEAD_SHA/check-runs" \
  --jq '.check_runs[] | select(.name | test("entelligence"; "i")) | .status' 2>/dev/null | head -1)

if [ "$ENT_STATE" != "queued" ] && [ "$ENT_STATE" != "in_progress" ]; then
  gh pr comment <PR_NUMBER> --body "@entelligence review"
fi
```

`@entelligence review` is the canonical trigger (`@entelligenceai review` and `@entelligence-ai review`
also work). The bot acknowledges with a 👀 reaction.

Now poll until the review has finished. This is two-phase: the **`Entelligence Review` check run**
completes first, and the **confidence score** is written to the summary comment a few seconds later
by an async job. Wait for both:

```bash
# Phase 1: wait for the "Entelligence Review" check run on this SHA to complete.
# Pull status/conclusion as scalars inside the gh --jq call. Do NOT capture the whole
# check-run object and re-parse it with a second jq: its output.text holds the full
# review markdown (badges, the score block, control characters), and re-feeding that
# through jq fails with "control characters ... must be escaped" on every tick.
# A review usually takes a few minutes; poll with an elapsed-time heartbeat and bail
# after 15 min so it can never hang silently.
SECONDS=0
while true; do
  STATE=$(gh api "repos/{owner}/{repo}/commits/$HEAD_SHA/check-runs" \
    --jq '[(.check_runs // [])[] | select(.name | test("entelligence"; "i"))]
          | "\(.[0].status // "absent") \(.[0].conclusion // "")"' 2>/dev/null)
  STATUS=${STATE%% *}; CONCLUSION=${STATE#* }

  if [ "$STATUS" = "completed" ]; then
    echo "Entelligence Review check completed (${CONCLUSION:-unknown}) after ${SECONDS}s."
    break
  fi
  if [ "$SECONDS" -ge 900 ]; then
    echo "Timed out waiting for the Entelligence Review check (${SECONDS}s). Inspect the PR manually."
    break
  fi
  echo "Waiting for Entelligence Review... (${STATUS:-absent}, ${SECONDS}s elapsed)"
  sleep 10
done

# Phase 2: wait for the summary comment's confidence score to refresh past the baseline.
# The score posts a few seconds after the check completes; bail after 3 min just in case.
SECONDS=0
while true; do
  CUR_TS=$(gh api --paginate "repos/{owner}/{repo}/issues/<PR_NUMBER>/comments?per_page=100" 2>/dev/null \
    | jq -rs 'add
      | [ .[] | select(.user.login | test("entelligence"; "i"))
                | select(.body | test("Confidence Score")) ]
      | sort_by(.updated_at) | last | .updated_at // ""')

  # A changed updated_at means the summary comment was re-edited with the new
  # review's score. (Use !=, not >: the [ ] / test builtin has no string > operator,
  # and \> breaks under zsh, the default macOS shell.)
  if [ -n "$CUR_TS" ] && [ "$CUR_TS" != "$BASELINE_TS" ]; then
    break
  fi
  if [ "$SECONDS" -ge 180 ]; then
    echo "Confidence score did not refresh within ${SECONDS}s; using the latest available."
    break
  fi
  echo "Waiting for confidence score to update... (${SECONDS}s elapsed)"
  sleep 5
done
```

#### B. Fetch the review results

**Confidence score** lives in the summary comment (the bot comment whose body contains
`Confidence Score`, selected by most recent `updated_at`). Pull `.body` out in the jq call itself
rather than capturing the JSON object and re-parsing it, so a multi-line body never trips the shell:

```bash
SUMMARY_BODY=$(gh api --paginate "repos/{owner}/{repo}/issues/<PR_NUMBER>/comments?per_page=100" \
  | jq -rs 'add
    | [ .[] | select(.user.login | test("entelligence"; "i"))
              | select(.body | test("Confidence Score")) ]
    | sort_by(.updated_at) | last | .body')

SCORE=$(printf '%s\n' "$SUMMARY_BODY" | grep -oE 'Confidence Score: [0-9]+/5' | head -1 | grep -oE '[0-9]+' | head -1)
LABEL=$(printf '%s\n' "$SUMMARY_BODY" | grep -oE 'Confidence Score: [0-9]+/5 - .*' | head -1 | sed 's/.* - //')
echo "Confidence: $SCORE/5 - $LABEL"
```

The summary also contains a `**Key Findings:**` section and a "Files requiring special attention"
block. Read them for high-level context.

**Unresolved inline threads** are the actionable work. Fetch the bot's unresolved review threads via
GraphQL (see [GitHub API reference](references/github-api.md)):

```bash
gh api graphql -f query='
query($cursor: String) {
  repository(owner: "OWNER", name: "REPO") {
    pullRequest(number: PR_NUMBER) {
      reviewThreads(first: 100, after: $cursor) {
        pageInfo { hasNextPage endCursor }
        nodes {
          id
          isResolved
          isOutdated
          comments(first: 5) {
            nodes { body path line author { login } }
          }
        }
      }
    }
  }
}'
```

Keep threads where `isResolved == false` and the first comment's `author.login` matches
`entelligence` (case-insensitive). Each comment body contains:

- A severity badge near the top: `![CRITICAL]`, `![MAJOR]`, or `![NIT]` (shields.io image).
- Often a GitHub committable suggestion fenced as a ` ```suggestion ` block.
- A collapsible block titled **Prompt to fix with AI** containing a ready-to-use fix prompt.

#### C. Check exit conditions

Stop the loop if **either** is true:

- Confidence score is **5/5** ("Safe to Merge") **and** there are **zero unresolved bot threads**.
- Max iterations reached (report the current state).

#### D. Fix actionable comments

For each unresolved bot thread, in priority order `CRITICAL` -> `MAJOR` -> `NIT`:

1. Read the file at the referenced `path`/`line` and understand the finding in context.
2. If the comment includes a ` ```suggestion ` block and it is correct, apply that exact change.
3. Otherwise follow the "Prompt to fix with AI" block, or implement the fix yourself.
4. If it is a false positive or intentional, leave the code as-is but still resolve the thread (a
   brief reply explaining why is optional).

Entelligence does not delete or mark old comments as outdated on a re-review; it runs cross-push
dedup so it only posts genuinely new findings next round. Resolving addressed threads yourself
(step E) is what keeps the unresolved count accurate.

#### E. Resolve threads

Resolve every thread you have addressed. Batch the GraphQL mutation (see
[GitHub API reference](references/github-api.md)):

```bash
gh api graphql -f query='
mutation {
  t1: resolveReviewThread(input: {threadId: "ID1"}) { thread { isResolved } }
  t2: resolveReviewThread(input: {threadId: "ID2"}) { thread { isResolved } }
}'
```

Shortcut: posting the PR comment `@entelligence resolve_all` resolves all bot threads at once. Use
it only when you have genuinely addressed (or accepted) every finding, since it does not
discriminate between fixed and unfixed threads.

#### F. Commit and push

```bash
git add -A
git commit -m "address entelligence review feedback (entloop iteration N)"
git push
```

Then go back to step **A**.

### 3. Report

After exiting the loop, summarize:

| Field              | Value         |
| ------------------ | ------------- |
| PR                 | #N            |
| Iterations         | N             |
| Final confidence   | X/5 - <label> |
| Threads resolved   | N             |
| Remaining threads  | N (if any)    |

If the loop exited on max iterations, list the remaining unresolved threads (file:line and the
finding title) and suggest next steps.

## Output format

```
Entloop complete.
  PR:            #128
  Iterations:    2
  Confidence:    5/5 - Safe to Merge
  Resolved:      6 threads
  Remaining:     0
```

If not fully resolved:

```
Entloop stopped after 5 iterations.
  PR:            #128
  Confidence:    4/5 - Mostly Safe
  Resolved:      9 threads
  Remaining:     2

Remaining issues:
  - src/auth.ts:45  [MAJOR] Rate-limit this endpoint
  - src/db.ts:112   [NIT]   Add an index on user_id
```
