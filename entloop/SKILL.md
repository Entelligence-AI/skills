---
name: entloop
description: >
  Iteratively improves a GitHub pull request until it is clean: zero unresolved review threads and a
  confidence score of 4/5 ("Mostly Safe") or 5/5 ("Safe to Merge"). Triggers an Entelligence review, fixes all
  actionable comments using each comment's "Prompt to fix with AI" and committable suggestions,
  resolves the threads, pushes, re-triggers the review, and repeats. Use when the user wants to fully
  optimize a PR against EntelligenceAI's code review standards before merging.
license: MIT
compatibility: >
  Requires git and gh (GitHub CLI) authenticated, and the EntelligenceAI PR review app installed on
  the repo (https://github.com/apps/entelligence-ai-pr-reviews). GitHub only.
metadata:
  author: Entelligence AI
  version: "1.1"
allowed-tools: Bash(gh:*) Bash(git:*)
---

# Entloop

Iteratively fix a GitHub PR until it is clean: zero unresolved review threads and a confidence score
of 4/5 ("Mostly Safe") or 5/5 ("Safe to Merge"). A fully-resolved 4/5 is a valid stop - no need to
chase the last point.

EntelligenceAI posts two things on a PR:

1. A single **summary issue comment** titled `## EntelligenceAI PR Summary` that it edits in place
   on every review. Its confidence-score block is delimited by an `<!-- CONFIDENCE_SCORE -->` marker
   and renders (current bots) as a **Merge Confidence** table row like
   `| **Merge Confidence** | ●●●○○ **3/5** — Review Recommended | … |`; older deployments
   emit a `## Confidence Score: 3/5 - Review Recommended` line instead. Select the comment
   by the stable `<!-- CONFIDENCE_SCORE -->` marker (or either phrase) and parse BOTH forms.
2. **Inline review comments** (resolvable review threads) authored by the bot, each carrying a
   severity badge (`CRITICAL` / `MAJOR` / `NIT`), an optional committable `suggestion` block, and a
   collapsible "Prompt to fix with AI" block you can apply directly.

The confidence score is computed from the inline findings (severity counts and how many are
unresolved). Drive the score to 4/5 or better by fixing and resolving every actionable thread, then
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

Keep the conversation small: every reply re-reads the whole session, so each line a poll prints or
each comment body you dump is paid for again on every later turn. The wait loops below print only
when the state changes, and step B writes the threads to a file and prints one line per thread;
read a full comment body only for the thread you are fixing. Start entloop in a fresh session (or
after `/clear`) when the current one already holds unrelated work. The snippets route command
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

Pick a run id for this loop and re-type it into every later block exactly as you re-type `PR`
(shell state does not survive between separately executed blocks):

```bash
RUN_ID="$(date +%Y%m%dT%H%M%S)-$$"; echo "RUN_ID=$RUN_ID"
```

Loop state lives in `/tmp/entloop.<PR>.<RUN_ID>.*`, so two entloop sessions working the same PR
never read each other's files.

The EntelligenceAI bot logins differ between environments. Match the author login
against a case-insensitive **exact allowlist** of the two bot logins — never a bare
`entelligence` substring, which a similarly-named non-bot account could satisfy and
inject a fake score or a trusted thread:

- `entelligence-ai-pr-reviews[bot]` (production)
- `entelligenceai-pr-review-alpha[bot]` (alpha/staging)

See [GitHub API reference](references/github-api.md) for every call this skill makes.

In jq: `select(.user.login | ascii_downcase | IN("entelligence-ai-pr-reviews[bot]",
"entelligenceai-pr-review-alpha[bot]"))` — use the same allowlist everywhere an author
is checked (the score comment, the review threads). The REST API reports bot logins WITH
the `[bot]` suffix, but GraphQL (`author { login }` on review threads) reports them WITHOUT
it (`entelligence-ai-pr-reviews`), so strip the suffix before comparing a GraphQL login, or
every bot thread is silently dropped and the PR looks clean. The check run is named exactly
`Entelligence Review` in both environments; match it with `.name == "Entelligence Review"`,
never a substring.

### 2. Loop

Repeat the following cycle. **Max 5 iterations** to avoid runaway loops.

#### A. Wait for the review of the current commit (never trigger one after a push)

EntelligenceAI reviews on open and on **every push** (`review_on_every_push`, on by default), so a
push already starts the review: wait for its `Entelligence Review` check run, never post
`@entelligence review` after pushing. The check run appears 30-60 s after the push, so a check made
right after `git push` sees nothing; posting then starts a second, whole-PR review of the same commit
(measured: about half of the manual triggers on entloop PRs came within 3 minutes of a commit). Post
`@entelligence review` only when no check run has appeared after the wait below (auto-review is off
for the repo, or the webhook was lost).

Run the whole step as one script so the variables persist. It can take up to ~20 minutes, so give
it the longest Bash timeout you have; if it is cut off, run it again: it resumes on the same head
and does not post a second trigger.

```bash
PR=<PR_NUMBER>
RUN_ID=<RUN_ID>
BOTS='["entelligence-ai-pr-reviews[bot]","entelligenceai-pr-review-alpha[bot]"]'

# updated_at of the current confidence-score comment (lets us tell when a NEW score lands).
score_ts() { gh api --paginate "repos/{owner}/{repo}/issues/$PR/comments?per_page=100" 2>/dev/null \
  | jq -rs --argjson bots "$BOTS" \
       'add | [ .[] | select(.user.login | ascii_downcase | IN($bots[]))
                      | select(.body | test("CONFIDENCE_SCORE|Confidence Score|Merge Confidence")) ]
        | sort_by(.updated_at) | last | .updated_at // ""'; }
BASELINE_TS=$(score_ts)   # taken BEFORE the push, so a fast review cannot land unnoticed

BEFORE=$(gh pr view "$PR" --json headRefOid -q .headRefOid)
git push --quiet || { echo "git push failed - resolve it before continuing the loop."; exit 1; }
HEAD_SHA=$(git rev-parse HEAD)
if [ "$HEAD_SHA" != "$BEFORE" ]; then PUSHED=1; APPEAR_WAIT=300; else PUSHED=0; APPEAR_WAIT=90; fi

# Pull scalars inside gh --jq; never capture the whole check-run object and re-parse it - its
# output.text holds the review markdown (badges, score block, control chars) and re-feeding that
# through jq fails with "control characters ... must be escaped". On an API error gh prints the
# error JSON to stdout, so anything that is not a known status reads as "absent" (poll again).
check_state() {
  local s
  s=$(gh api "repos/{owner}/{repo}/commits/$HEAD_SHA/check-runs" \
    --jq '[(.check_runs // [])[] | select(.name == "Entelligence Review")]
          | "\(.[0].status // "absent") \(.[0].conclusion // "")"' 2>/dev/null)
  case "${s%% *}" in
    queued|in_progress|completed|waiting|requested|pending) printf '%s\n' "$s" ;;
    *) echo "absent " ;;
  esac
}

# Phase 0: wait for the auto-review check run to APPEAR. Trigger by comment only if none shows up.
STATE=$(check_state); STATUS=${STATE%% *}; FIRST_STATUS=$STATUS
SECONDS=0; TRIGGERED=0
while [ -z "$STATUS" ] || [ "$STATUS" = "absent" ]; do
  if [ "$SECONDS" -ge "$APPEAR_WAIT" ]; then
    if [ "$TRIGGERED" = 1 ]; then
      echo "No Entelligence Review check even after requesting one. Inspect the PR manually."; exit 1
    fi
    echo "No auto-review appeared within ${SECONDS}s - requesting one."
    gh pr comment "$PR" --body "@entelligence review" >/dev/null   # also: @entelligenceai review
    TRIGGERED=1; SECONDS=0; APPEAR_WAIT=300
  fi
  sleep 10
  STATE=$(check_state); STATUS=${STATE%% *}
done
# The existing score is current only when nothing was pushed and the review had already finished.
if [ "$PUSHED" = 0 ] && [ "$FIRST_STATUS" = "completed" ]; then REVIEW_IS_FRESH=1
  echo "Entelligence already reviewed this commit - using the existing review."
else REVIEW_IS_FRESH=0; echo "Entelligence Review: $STATUS - waiting for it to finish."; fi

# Phase 1: wait for the check run to complete. Quiet: no line per poll. Bail after 15 min, and do
# not continue on a missing review - stale results would drive the next fix.
SECONDS=0
while [ "$STATUS" != "completed" ]; do
  if [ "$SECONDS" -ge 900 ]; then
    echo "Timed out waiting for the Entelligence Review check (${SECONDS}s). Inspect the PR manually."; exit 1
  fi
  sleep 15
  STATE=$(check_state); STATUS=${STATE%% *}
done
echo "Entelligence Review check completed (${STATE#* })."

# Phase 2: wait for the summary comment's score to reflect this commit. If the review already
# existed it is current already (take it once present); otherwise wait for updated_at to move past
# the baseline. (!=, not >: [ ] / test has no string > operator and \> breaks under zsh.)
SECONDS=0
while true; do
  CUR_TS=$(score_ts)
  if [ -n "$CUR_TS" ] && { [ "$REVIEW_IS_FRESH" = "1" ] || [ "$CUR_TS" != "$BASELINE_TS" ]; }; then break; fi
  if [ "$SECONDS" -ge 180 ]; then
    echo "Confidence score did not refresh within ${SECONDS}s; using the latest available."; break
  fi
  sleep 5
done
```

#### B. Fetch the review results

**Confidence score** lives in the summary comment (the bot comment whose body contains
the `<!-- CONFIDENCE_SCORE -->` marker — or either score phrase — selected by most recent
`updated_at`). Pull `.body` out in the jq call itself
rather than capturing the JSON object and re-parsing it, so a multi-line body never trips the shell:

```bash
BOTS='["entelligence-ai-pr-reviews[bot]","entelligenceai-pr-review-alpha[bot]"]'
SUMMARY_BODY=$(gh api --paginate "repos/{owner}/{repo}/issues/<PR_NUMBER>/comments?per_page=100" \
  | jq -rs --argjson bots "$BOTS" 'add
    | [ .[] | select(.user.login | ascii_downcase | IN($bots[]))
              | select(.body | test("CONFIDENCE_SCORE|Confidence Score|Merge Confidence")) ]
    | sort_by(.updated_at) | last | .body')

# Current format: the Merge Confidence table row ("| **Merge Confidence** | ●●●○○ **3/5** — Label | …").
# Legacy format: "Confidence Score: 3/5 - Label". Parse both; table row wins when present.
ROW=$(printf '%s\n' "$SUMMARY_BODY" | grep -E 'Merge Confidence' | head -1)
if [ -n "$ROW" ]; then
  SCORE=$(printf '%s\n' "$ROW" | grep -oE '\*\*[0-9]+/5\*\*' | head -1 | grep -oE '[0-9]+' | head -1)
  LABEL=$(printf '%s\n' "$ROW" | sed -E 's/.*\*\*[0-9]+\/5\*\* *[—-] *([^|]*)\|.*/\1/' | sed 's/ *$//')
else
  SCORE=$(printf '%s\n' "$SUMMARY_BODY" | grep -oE 'Confidence Score: [0-9]+/5' | head -1 | grep -oE '[0-9]+' | head -1)
  LABEL=$(printf '%s\n' "$SUMMARY_BODY" | grep -oE 'Confidence Score: [0-9]+/5 - .*' | head -1 | sed 's/.* - //')
fi
# Format drift guard: if neither form matched, take the first "N/5" after the score marker.
# An empty SCORE would make the exit check below unsatisfiable and burn all 5 iterations.
if [ -z "$SCORE" ]; then
  SCORE=$(printf '%s\n' "$SUMMARY_BODY" | sed -n '/CONFIDENCE_SCORE/,$p' | grep -oE '[0-5]/5' | head -1 | cut -d/ -f1)
fi
[ -n "$SCORE" ] || echo "Could not read the confidence score - read the summary comment yourself before deciding to stop."
echo "Confidence: $SCORE/5 - $LABEL"
```

The summary also contains a `**Key Findings:**` section and a "Files requiring special attention"
block. Read them for high-level context.

**Unresolved inline threads** are the actionable work. Fetch the bot's unresolved review threads via
GraphQL:

Paginate to the end — a single `first: 100` page can silently truncate a large PR's
thread list and report it clean while findings remain:

```bash
PR=<PR_NUMBER>
RUN_ID=<RUN_ID>
T="/tmp/entloop.$PR.$RUN_ID.threads.json"   # per run, so parallel loops never mix thread lists
CURSOR=; : > "$T"   # empty, NOT the literal string "null" — gh -f would send "null" as a real cursor value and GitHub rejects it
while :; do
  PAGE=$(gh api graphql -f query='
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
  }' ${CURSOR:+-f cursor="$CURSOR"})
  jq '.data.repository.pullRequest.reviewThreads.nodes[]' <<<"$PAGE" >> "$T"
  jq -e '.data.repository.pullRequest.reviewThreads.pageInfo.hasNextPage' <<<"$PAGE" >/dev/null || break
  CURSOR=$(jq -r '.data.repository.pullRequest.reviewThreads.pageInfo.endCursor' <<<"$PAGE")
done

# One line per unresolved bot thread; the full bodies stay in $T. GraphQL logins carry no
# "[bot]" suffix, so compare against the allowlist with the suffix stripped.
jq -rs '[ .[] | select(.isResolved | not)
          | select((.comments.nodes[0].author.login // "") | ascii_downcase
                   | IN("entelligence-ai-pr-reviews", "entelligenceai-pr-review-alpha")) ]
        | .[] | .comments.nodes[0] as $c
        | ($c.body | (capture("badge/-(?<s>CRITICAL|MAJOR|NIT)-") // {s: "?"}).s) as $sev
        | ($c.body | (capture("\\*\\*(?<t>[^*]+)\\*\\*") // {t: "(no title)"}).t) as $title
        | "\(.id)  [\($sev)]  \($c.path):\($c.line // "-")  \($title)"' "$T"
```

To read one finding in full: `jq -rs '.[] | select(.id == "<THREAD_ID>") | .comments.nodes[0].body' "$T"`.

Keep threads where `isResolved == false` and the first comment's `author.login` is in the
exact bot allowlist from step 1 (case-insensitive, `[bot]` suffix stripped). Each comment body contains:

- A severity badge near the top: `![CRITICAL]`, `![MAJOR]`, or `![NIT]` (shields.io image).
- Often a GitHub committable suggestion fenced as a ` ```suggestion ` block.
- A collapsible block titled **Prompt to fix with AI** containing a ready-to-use fix prompt.

#### C. Check exit conditions

The goal is a clean PR, not a perfect score. Stop the loop (success) when there are **zero unresolved
bot review threads** and the confidence score is **4/5 or 5/5** - a fully-resolved PR at 4/5
("Mostly Safe") is just as valid a stop as 5/5, so do not grind for the last point once every inline
comment is cleared.

Stop if **either**:

- `unresolved_threads == 0` **and** `score >= 4` (4/5 or 5/5), or
- max iterations reached (report the current state).

Keep looping only while threads are still unresolved or the score is 3/5 or lower.

**Every push is a full review round** (the main review plus any deep-review pass), so never push
just for NITs. When the score is 4/5 or 5/5 and only NIT threads remain, do not start another
iteration: fix the cheap ones and commit them locally (unpushed, listed in the report so the user
ships them with their next change), reply on the rest, resolve them all, and stop. NIT fixes that
ride along with a CRITICAL or MAJOR fix in the same push are fine.

#### D. Fix actionable comments

For each unresolved bot thread, in priority order `CRITICAL` -> `MAJOR` -> `NIT`:

1. Read the file at the referenced `path`/`line` and understand the finding in context.
2. If the comment includes a ` ```suggestion ` block and it is correct, apply that exact change.
3. Otherwise follow the "Prompt to fix with AI" block, or implement the fix yourself.
4. If it is a false positive or intentional, leave the code as-is but still resolve the thread (a
   brief reply explaining why is optional).
5. **Watch for fix chains.** From iteration 3 on, a new finding in the same function as a thread you
   already fixed is often a narrower edge case of your own fix (a race that needs unusual timing,
   an adversarial input size). Fix it only when it is a realistic failure for this code's callers;
   otherwise reply with that reasoning and resolve it. Chasing each narrower case is what keeps a
   loop from converging.

Entelligence does not delete or mark old comments as outdated on a re-review; it runs cross-push
dedup so it only posts genuinely new findings next round. Resolving addressed threads yourself
(step E) is what keeps the unresolved count accurate.

#### E. Resolve threads

Resolve every thread you have addressed. Batch the GraphQL mutation:

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

#### F. Commit (step A pushes)

Stage **only the files you edited for this iteration's findings** — never `git add -A`,
which would sweep unrelated working-tree changes (a user's in-progress edit, a local
secret) into the PR. Check what else is dirty first, and leave it alone:

```bash
git status --short             # anything here you did NOT touch for a finding stays unstaged
git add path/to/fixed_file1 path/to/fixed_file2   # explicit paths only
git commit -m "address entelligence review feedback (entloop iteration N)"
```

Do not push here: step A pushes, and it needs to see the head move to know a review is coming
(pushing here first makes step A think nothing was pushed and shortens its wait).

If `git status` shows unrelated modifications you cannot attribute to your own fixes,
do not commit them — mention them in the final report instead.

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
