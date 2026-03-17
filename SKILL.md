---
name: pr-review-loop
description: Automate GitHub PR review workflows with configurable reviewers. Use this skill whenever you need to handle PR code reviews at scale, loop through multiple reviewer feedback cycles, address all review comments, ensure tests and CI pass, and keep iterating until all reviews are resolved. Supports any number of reviewers and a coder role. Configure reviewer names, comment triggers, token names, and out-of-scope rejection threshold as inputs. Triggers on requests to "automate PR reviews", "handle code reviews", "process PR feedback loops", "CI/review loop", or similar workflows involving iterative review resolution.
---

# PR Review Loop Automation

Automate the GitHub pull request review cycle with configurable reviewers: fetch unresolved reviews, address feedback, run tests and CI, post reactions, and loop until all reviews are resolved.

## Prerequisites

- **Access to PR**: You can fetch PR metadata and comments via unauthenticated `curl` calls to the GitHub API
- **Environment Variables**: Configure the following as inputs:
  - `<REVIEWER_1_NAME>_PR_TOKEN` — For posting review requests from reviewer 1
  - `<REVIEWER_2_NAME>_PR_TOKEN` — For posting review requests from reviewer 2
  - Additional reviewer tokens as needed
  - `<CODER_NAME>_PR_TOKEN` — For posting 👍 reactions to addressed comments
  - These tokens must be set as environment variables and never exposed in logs or output
- **Reviewer Configuration**: You will specify:
  - Reviewer names (e.g., "reviewer1", "reviewer2")
  - Review comment triggers for each (e.g., "@reviewer1", "/reviewer2 review")
  - Associated token environment variable names for each
  - Coder name and its token variable name
- **Git Setup**: You have access to `git` CLI and can commit/push changes
- **GitHub PR URL**: Know the full GitHub PR URL (e.g., `https://github.com/owner/repo/pull/123`)

## Configuration Inputs

Before starting the workflow, gather the following information:

| Input | Example | Description |
|-------|---------|-------------|
| PR URL | `https://github.com/owner/repo/pull/123` | Full GitHub PR URL |
| Reviewer 1 Name | `reviewer1` | Internal name for first reviewer |
| Reviewer 1 Trigger | `@reviewer1` | Comment text to request review from reviewer 1 |
| Reviewer 1 Token Var | `REVIEWER1_PR_TOKEN` | Environment variable name for reviewer 1's token |
| Reviewer 2 Name | `reviewer2` | Internal name for second reviewer (if used) |
| Reviewer 2 Trigger | `/reviewer2 review` | Comment text to request review from reviewer 2 |
| Reviewer 2 Token Var | `REVIEWER2_PR_TOKEN` | Environment variable name for reviewer 2's token |
| Coder Name | `coder` | Internal name for the coder role |
| Coder Token Var | `CODER_PR_TOKEN` | Environment variable name for coder's token (for posting 👍/👎) |
| Max Out-of-Scope | `3` | Max rejected out-of-scope reviews before terminating the loop (default: 3) |

Add additional rows for more reviewers as needed.

## Workflow Overview

The skill runs a loop that:
1. Requests reviews from configured reviewers if previous changes have been addressed
2. Waits for and fetches unresolved review comments
3. Evaluates each comment against the PR/issue scope
4. Rejects out-of-scope comments with 👎 reactions and explanatory comments
5. Addresses in-scope feedback by making code changes
6. Verifies tests pass and CI is green
7. Posts 👍 reactions to addressed comments (using coder token)
8. Loops until no unresolved in-scope comments remain

The loop **terminates early** if:
- No unresolved comments are found
- Authentication error occurs (will stop immediately)
- Rate limit is reached
- Token limit is exceeded
- Out-of-scope rejection count exceeds `max_out_of_scope` threshold (default: 3)

## Step-by-Step Workflow

### Step 1: Request Reviews (if needed)

**Before posting any review trigger**, check for pending reviews that need to be addressed first:

```bash
curl -s "https://api.github.com/repos/<owner>/<repo>/pulls/<pr_number>/comments"
```

If there are any comments that are **all of the following**:
- Not outdated
- Not resolved
- Have **no 👍 reaction** from the coder
- Have **no 👎 reaction** from the coder

Then **skip posting review triggers** and go directly to **Step 2** to address the existing pending reviews first. Do not request new reviews while actionable reviews remain unhandled.

Only if **no such pending reviews exist**, proceed to post review triggers:

For each configured reviewer:

**If you've posted changes to the previous review from that reviewer** and there are no unresolved review comments:
- Post a comment using the configured review trigger (e.g., `@reviewer1`, `/reviewer2 review`) using the reviewer's `<REVIEWER_NAME>_PR_TOKEN`
- Wait briefly for the reviewer to respond

Use the configuration table above to determine which trigger text and token variable to use for each reviewer.

### Step 2: Fetch Unresolved Comments

Before proceeding, wait a few seconds (reviews take time to arrive).

Fetch all comments on the PR using unauthenticated `curl`:
```bash
curl -s "https://api.github.com/repos/<owner>/<repo>/pulls/<pr_number>/comments"
```

Filter for:
- Comments that are **unresolved** (not marked as resolved)
- Comments that are **not outdated** (created/updated after the last commit that addressed them)
- Comments with **no 👍 reaction** posted yet (by the coder)

### Step 3: Check for Termination Conditions

**Stop the loop if:**
- No unresolved comments found → Success, exit gracefully
- An API call returned HTTP 401 (Unauthorized) → Auth error, stop immediately and report
- An API call returned HTTP 429 (Rate Limit Exceeded) → Stop and report rate limit reached
- Token limit exceeded in your context → Stop and report
- Out-of-scope rejection count ≥ `max_out_of_scope` (default: 3) → Reviewer misalignment detected, stop and report

### Step 4: Evaluate Comment Scope

Before addressing any comment, evaluate whether it falls within the original PR/issue scope:

1. Fetch the PR description and linked issue (if any) to understand the intended scope:
   ```bash
   curl -s "https://api.github.com/repos/<owner>/<repo>/pulls/<pr_number>"
   ```
2. For each unresolved comment, compare the requested change against the PR/issue scope:
   - **In scope**: Directly related to the PR's stated goal, bug fix, or feature
   - **Out of scope**: Security hardening unrelated to the PR, refactoring beyond the PR goal, unrelated feature additions, style changes not required by the PR, infrastructure changes not mentioned in the PR

3. **For out-of-scope comments**:
   a. Post a 👎 reaction using the coder's token:
      ```bash
      TOKEN=$(python3 -c "import os; print(os.environ.get('<CODER_TOKEN_VAR>',''))")
      RESULT=$(curl -s -o /dev/null -w "%{http_code}" -X POST \
        -H "Authorization: Bearer $TOKEN" \
        -H "Content-Type: application/json" \
        -d '{"content": "-1"}' \
        "https://api.github.com/repos/<owner>/<repo>/pulls/comments/<comment_id>/reactions")
      echo "HTTP $RESULT"
      ```
   b. Post an inline reply **directly to the review comment** (not the main PR thread) using the coder's token:
      ```bash
      TOKEN=$(python3 -c "import os; print(os.environ.get('<CODER_TOKEN_VAR>',''))")
      curl -s -X POST \
        -H "Authorization: Bearer $TOKEN" \
        -H "Content-Type: application/json" \
        -d '{"body": "Out of scope for this PR. Tracked separately.", "in_reply_to": <comment_id>}' \
        "https://api.github.com/repos/<owner>/<repo>/pulls/<pr_number>/comments"
      ```
      > **Important**: Reply inline to the specific review comment (using `in_reply_to`) rather than posting to the main PR issue thread. This preserves context and keeps rejection messages attached to the comment they respond to.
   c. Increment the out-of-scope rejection counter
   d. Record the out-of-scope item for future reference (note the comment ID, reviewer, and description)
   e. **Skip** implementing the change — do not modify the code for this comment

4. **If HTTP 401 or 429** when posting the reaction/comment: Stop loop immediately

5. Check termination: if rejection counter ≥ `max_out_of_scope`, stop the loop and report all collected out-of-scope items

6. Continue to Step 5 only with **in-scope** comments

### Step 5: Address In-Scope Review Feedback

For each unresolved **in-scope** comment:
1. Read the review comment text
2. Understand what needs to be fixed
3. Make the necessary code changes to the repo
4. Verify any linting errors using the CI config if specified
5. Commit changes with a descriptive message (reference the PR and comment if possible)
6. Push changes to the PR branch

### Step 6: Verify CI Status

After pushing:
1. Wait for CI to run
2. Check the CI status by fetching the PR details:
   ```bash
   curl -s "https://api.github.com/repos/<owner>/<repo>/pulls/<pr_number>"
   ```
   Look for the `statuses_url` and check status
3. If CI is **failing**: Diagnose the failure and make additional commits to fix it
4. If CI is **passing**: Proceed to next step

### Step 7: Post Reactions

For each in-scope comment you've addressed:
1. Extract the comment ID
2. Post a 👍 reaction using the coder's token (`<CODER_NAME>_PR_TOKEN`):
   ```bash
   TOKEN=$(python3 -c "import os; print(os.environ.get('<CODER_TOKEN_VAR>',''))")
   RESULT=$(curl -s -o /dev/null -w "%{http_code}" -X POST \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"content": "+1"}' \
     "https://api.github.com/repos/<owner>/<repo>/pulls/comments/<comment_id>/reactions")
   echo "HTTP $RESULT"
   ```
   (Replace `<CODER_TOKEN_VAR>` with the actual token variable name from your configuration)

3. If HTTP 401: Stop loop immediately (auth error)
4. If HTTP 429: Stop loop immediately (rate limit reached)

### Step 8: Loop Back

If you've successfully posted 👍 reactions and there are no more unresolved in-scope comments, go back to **Step 1** and request new reviews from the configured reviewers.

Otherwise, return to **Step 2** to fetch updated comment status.

If the loop terminates due to exceeding `max_out_of_scope`, report a summary of all collected out-of-scope items so they can be tracked as separate issues or PRs.

### Step 9: Final Sweep After Termination

When the loop ends for **any reason** (no reviews remaining, out-of-scope limit reached, or other termination condition), perform a final verification before declaring completion:

1. Wait briefly (3–5 seconds) to allow any in-flight reviews to arrive
2. Re-fetch all PR comments:
   ```bash
   curl -s "https://api.github.com/repos/<owner>/<repo>/pulls/<pr_number>/comments"
   ```
3. Check for any comments that are:
   - Not outdated
   - Not resolved
   - Have **no 👍 reaction** from the coder
   - Have **no 👎 reaction** from the coder

4. If such comments exist:
   - Evaluate their scope (Step 4)
   - Address any in-scope comments (Steps 5–7) — **without** posting new review triggers
   - Reject out-of-scope comments with 👎 and inline replies (Step 4) — **without** posting new review triggers
   - **Do not loop back to Step 1 or trigger new reviews during the final sweep**

5. Once no unhandled comments remain (or all remaining are handled), report the final status and exit.

> **Note**: No review triggers should occur during the final sweep. The goal is only to catch and handle any comments that arrived after the loop's last check.

## Important Notes

### Token Security
- **Never** log, echo, or display token values
- Use the pattern: `TOKEN=$(python3 -c "import os; print(os.environ.get('<TOKEN_VAR>',''))")`
- Check tokens silently; if missing, report that the token is not set and stop

### Error Handling
- **Auth errors (HTTP 401)**: Stop immediately. Do not retry. Do not proceed to next steps.
- **Rate limits (HTTP 429)**: Stop immediately. Report rate limit reached.
- **Token exhaustion**: If context token limit is reached, stop and report.
- **Transient failures**: Retry up to 2 times for network errors (HTTP 5xx), then stop

### Timing
- Wait 3-5 seconds between checking for new reviews (reviews take time to arrive)
- Wait for CI to complete before checking status (typically 2-10 minutes depending on test suite)
- Use exponential backoff for polling (wait longer as you check more times)

### Comment Filtering
Only address comments that are:
- Unresolved (not closed/resolved in the PR UI)
- Not outdated (created/updated after the last applicable commit)
- Do not have a 👍 reaction from the coder yet
- Evaluated as **in scope** for the current PR (see Step 4)

Comments marked as resolved, outdated, or out-of-scope should be skipped.

### Out-of-Scope Detection
When evaluating scope, consider the PR title, description, and linked issue as the source of truth:
- A comment is **in scope** if it directly relates to the bug fix, feature, or change described in the PR
- A comment is **out of scope** if it requests unrelated security hardening, refactoring, style changes, or new features not mentioned in the PR
- When in doubt, err on the side of addressing the comment (bias toward in-scope)
- Out-of-scope items are collected and can be used to create follow-up issues after the loop ends
- The `max_out_of_scope` threshold (default: 3) prevents infinite loops caused by persistent reviewer misalignment; adjust it upward if your workflow expects many advisory comments

### Reviewer Configuration Notes
- Reviewer names and triggers are flexible and can be customized for your workflow
- Each reviewer should have its own token variable
- The coder token is separate and used only for posting 👍 reactions
- You can have as many reviewers as needed; just follow the same pattern

## API Reference

### Fetch PR Comments (Unauthenticated)
```bash
curl -s "https://api.github.com/repos/OWNER/REPO/pulls/PR_NUMBER/comments"
```

### Fetch PR Details
```bash
curl -s "https://api.github.com/repos/OWNER/REPO/pulls/PR_NUMBER"
```

### Post Comment
```bash
TOKEN=$(python3 -c "import os; print(os.environ.get('TOKEN_VAR',''))")
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"content": "Your comment here"}' \
  "https://api.github.com/repos/OWNER/REPO/pulls/PR_NUMBER/comments"
```

### Post 👍 Reaction to Comment (in-scope, addressed)
```bash
TOKEN=$(python3 -c "import os; print(os.environ.get('TOKEN_VAR',''))")
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"content": "+1"}' \
  "https://api.github.com/repos/OWNER/REPO/pulls/comments/COMMENT_ID/reactions"
```

### Post 👎 Reaction to Comment (out-of-scope, rejected)
```bash
TOKEN=$(python3 -c "import os; print(os.environ.get('TOKEN_VAR',''))")
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"content": "-1"}' \
  "https://api.github.com/repos/OWNER/REPO/pulls/comments/COMMENT_ID/reactions"
```

### Post Inline Reply to Review Comment (out-of-scope rejection)
```bash
TOKEN=$(python3 -c "import os; print(os.environ.get('TOKEN_VAR',''))")
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"body": "Out of scope for this PR. Tracked separately.", "in_reply_to": COMMENT_ID}' \
  "https://api.github.com/repos/OWNER/REPO/pulls/PR_NUMBER/comments"
```
> Replies inline to the specific review comment rather than posting to the main PR thread, preserving context.

## Debugging

If something goes wrong:
1. Check the HTTP response codes for auth/rate limit errors
2. Verify all environment tokens are set (without logging their values)
3. Review PR comments manually at the GitHub web UI if needed
4. Check CI logs if tests fail
5. Ensure commit messages are meaningful for audit trail
6. Verify reviewer names and triggers match your configuration
7. Confirm coder token variable name is correct when posting reactions
8. If the loop terminates due to `max_out_of_scope` threshold: review the collected out-of-scope items and consider creating separate issues for them; increase the threshold if the reviewer's suggestions are broadly acceptable
9. If a comment was incorrectly classified as out-of-scope: re-run the loop after adjusting the scope evaluation logic or increase `max_out_of_scope`
