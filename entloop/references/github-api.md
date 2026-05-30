# GitHub API Reference

GitHub REST and GraphQL calls used by the entloop workflow, via `gh`. All REST calls use the
`{owner}` and `{repo}` placeholders, which `gh api` resolves from the current repo's `origin` remote.
For GraphQL, substitute `OWNER`, `REPO`, and `PR_NUMBER` literally.

## EntelligenceAI identifiers

- **Bot logins** (match case-insensitively on `entelligence`):
  - `entelligence-ai-pr-reviews[bot]` (production)
  - `entelligenceai-pr-review-alpha[bot]` (alpha/staging)
- **Check run name**: `Entelligence Review`
- **Summary comment**: a single issue comment whose body starts with `## EntelligenceAI PR Summary`
  and contains the confidence-score block delimited by `<!-- CONFIDENCE_SCORE -->`. It is edited in
  place each review, so always select the one with the latest `updated_at`.
- **Score line format**: `## Confidence Score: <n>/5 - <label>`
- **Manual triggers** (PR comments): `@entelligence review`, `@entelligenceai review`,
  `@entelligence-ai review`. Related commands: `@entelligence resolve_all`, `@entelligence summary`,
  `@entelligence config`.

## Identify the PR for the current branch

```bash
gh pr view --json number,headRefName,headRefOid \
  -q '{number: .number, branch: .headRefName, sha: .headRefOid}'
```

## Trigger a review

```bash
gh pr comment <PR_NUMBER> --body "@entelligence review"
```

## Poll the "Entelligence Review" check run

```bash
gh api "repos/{owner}/{repo}/commits/<HEAD_SHA>/check-runs" \
  --jq '.check_runs[] | select(.name | test("entelligence"; "i")) | {status, conclusion}'
```

Terminal `status` is `completed`; `conclusion` is `success` or `failure`. The check run can be
disabled per repo or owned by an ECS worker, so do not block forever on it; if it never appears,
fall back to waiting on the summary comment's `updated_at` (below).

## Fetch the summary comment + confidence score (REST)

The summary comment is an issue comment that the bot edits in place, so select by `updated_at`, not
`created_at`:

```bash
gh api --paginate "repos/{owner}/{repo}/issues/<PR_NUMBER>/comments?per_page=100" \
  | jq -s 'add
    | [ .[] | select(.user.login | test("entelligence"; "i"))
              | select(.body | test("Confidence Score")) ]
    | sort_by(.updated_at) | last
    | {author: .user.login, updated_at, body}'
```

Parse the score and label from the body (use `printf`, not `echo`, when re-feeding a multi-line body
to grep):

```bash
printf '%s\n' "$BODY" | grep -oE 'Confidence Score: [0-9]+/5 - .*' | head -1
# -> "Confidence Score: 3/5 - Review Recommended"
```

## Fetch unresolved review threads (GraphQL, paginated)

```graphql
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
            nodes {
              body
              path
              line
              author { login }
              createdAt
            }
          }
        }
      }
    }
  }
}
```

Keep nodes where `isResolved == false` and `comments.nodes[0].author.login` matches `entelligence`
(case-insensitive). Page through with the returned `endCursor` while `hasNextPage` is true.

### Inline comment body anatomy

Each bot review comment body contains, in order:

- A severity badge image: `![CRITICAL](...)`, `![MAJOR](...)`, or `![NIT](...)`, optionally followed
  by a category badge (e.g. `![BUG]`, `![SECURITY]`) and a bold **title**.
- A description of the issue.
- Frequently a GitHub committable suggestion:

  ````
  ```suggestion
  <replacement code for the commented line(s)>
  ```
  ````

  Applying the suggestion (via the GitHub UI "Commit suggestion", or by editing the file to match)
  is often the fastest correct fix.
- A collapsible block:

  ```
  <details>
  <summary><strong>Prompt to fix with AI</strong></summary>

  > Copy this prompt into your AI coding assistant to fix this issue.

  ```
  <ready-to-use fix prompt>
  ```

  </details>
  ```

## Batch-resolve threads (GraphQL)

```graphql
mutation {
  t1: resolveReviewThread(input: {threadId: "ID1"}) { thread { isResolved } }
  t2: resolveReviewThread(input: {threadId: "ID2"}) { thread { isResolved } }
}
```

Or resolve all bot threads at once with a PR comment (only after addressing every finding):

```bash
gh pr comment <PR_NUMBER> --body "@entelligence resolve_all"
```

## Notes on re-review behavior

- EntelligenceAI does **not** delete prior comments or mark them outdated when you push new commits.
- It runs cross-push dedup, so on each re-review it posts only genuinely new findings; previously
  posted threads stay until you resolve them.
- Therefore the unresolved-thread count is only accurate if you resolve threads as you address them
  (step E of the loop).
