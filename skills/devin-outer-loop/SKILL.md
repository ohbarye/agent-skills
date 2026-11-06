---
name: devin-outer-loop
description: Outer loop that hands a design to Devin, has it open a PR, then repeats inline review and fix verification until no findings remain. Use when you want Devin to implement a design settled in the conversation, or to resume an interrupted outer loop.
license: MIT
---

# devin-outer-loop

An outer loop with cloud Devin as the implementer. It assumes the design is already settled in this conversation. Roles are split as follows. **The user** means the person running this loop.

- **Devin**: implements, gets CI green, fixes review findings and replies on the threads
- **You (the agent running this skill)**: review, judge the fixes, resolve threads, decide when the loop ends
- **The user**: makes spec decisions. Approval and merge are left to human reviewers

Drive Devin through the tools of the Devin MCP server (`https://mcp.devin.ai/mcp`), such as `devin_session_create`. Some agents add a prefix like `mcp__devin__`. If the session tools are unavailable, ask the user to add the Devin MCP server to their agent and stop.

Drive GitHub with `gh`, run from inside a checkout of the target repository. `{owner}` / `{repo}` in a `gh api` path or in a `-F` value are filled in by `gh` from the repository in the current directory. Writes (posting reviews, replies, resolving threads) appear under the name of the user authenticated in `gh`.

## 0. Learn the repository's rules

These shape the brief and the review, so first read the repository's agent docs (`AGENTS.md`, `CLAUDE.md`, and so on) and its skills, then record the following in the state file. Write "none" for anything you cannot find.

- Repository name (`gh repo view --json nameWithOwner -q .nameWithOwner`)
- Commit and PR conventions (a PR-creation skill or template, language, required labels, how to close issues)
- Lint and test commands
- A code review skill (used in step 5): one that takes a PR number and reviews that PR's diff. Exclude skills that assume the local branch or a comparison base. If several qualify, let the user choose
- Where intermediate files go (the state file lives there)
- **Other automation that picks up the same issues or PRs**, such as scheduled Devin runs. Search the docs and skills for `devin` and find what they pick up (labels, Project status, and so on). So that this loop and that automation do not both start or fix the same work, decide which labels Devin must not add, and which label to put on the issue to take it out of their candidates. For the issue label, search the docs and skills for its name and pick one that does nothing beyond removing the issue from candidates (no skipping other jobs, no nudging people)

## State file

Keep appending to `devin-outer-loop/<slug>/state.md` under the intermediate-file location (`tmp/` if there is none): what you learned in step 0, plus the items below. The loop takes hours, and this file lets you resume after the agent session ends or loses its context. When asked to resume, read this file first.

- Devin session ID and URL, PR URL and head branch
- Round number (+1 each time you hand off to Devin in step 6)
- Per round: the head SHA you reviewed, the findings, and your judgments
- **Loop threads**: the ID (`databaseId`) of the first comment of each thread you opened in step 6
- Consecutive CI bounce count (per check name; reset to 0 when that check turns green)
- Times you re-asked about an unanswered thread (per thread)
- The user's decisions, and when you last messaged Devin
- Labels you put on the issue

## 1. Write the brief and get the user's OK

Put the following in the brief. Do not compress the design: include everything decided in the conversation (and URLs if it lives in issue comments or docs).

- Goal, and the target issue (number and title, if any)
- In scope / out of scope
- Base branch (for a stack: which PR's branch to branch from, and not to push to that PR's branch)
- Follow the repository's agent docs. Make the smallest change test-first, and get the lint and related tests from step 0 green
- Open the PR following the conventions from step 0. Name the labels step 0 said Devin must not add, and ask it not to add them
- Keep fixing until CI is green, then report
- When a spec decision is needed, stop and ask instead of guessing
- Fix review findings with additional commits on the existing PR branch (no force push, no new PR). Reply on each thread after fixing. **Reviewers resolve threads, so Devin does not**

Show the brief to the user together with what you decided in step 0 (review skill, labels Devin must not add, label for the issue) and the fact that reviews, replies, and resolutions during the loop will be posted under the user's name. Create the session only after the user's OK. If there is an issue and step 0 found a label that removes it from other automation's candidates, add that label before creating the session.

## 2. Create the session

Call `devin_session_create` for one session: `repos=["<owner>/<repo>"]` (the name from step 0), a `title`, and the brief as `prompt`. Pass `devin_mode` only if the user asked for one (picker name Ultra → `ultra`, and so on). Prefix the returned `session_id` with `devin-`, record it in the state file, and give the session URL to the user.

`notify_on_response` is not delivered to local MCP clients, so waiting happens in step 3.

## 3. Wait for Devin to settle

Call `devin_session_gather` (`timeout_seconds=590`) repeatedly until the session is settled. Then read `status_detail` and `pull_requests` with `devin_session_interact` action `get`, and the latest messages with `get_messages`. Treat `suspended` (sleeping) as settled, like `finished`. Decide only from Devin messages newer than the last message you sent (a gather right after sending can return a stale `finished` from before Devin picked up the message).

If there is no new message, look at the PR. Devin watches GitHub on its own and sometimes replies only on the threads without writing to the session. If the head has moved past the SHA you last reviewed and every unresolved loop thread has a reply from Devin, treat it as "finished fixing". Otherwise Devin is idle: send one `message` asking "How far did you get with the previous request?" and wait again. If there is still neither a new message nor a PR change, tell the user and stop.

| State | Next |
| --- | --- |
| Reports it opened the PR for the first time | Step 5 |
| Finished fixing (including CI fixes) | Step 4 if round ≥ 1, step 5 if round is 0 |
| Asks a question | If the design or the user's decisions answer it, answer and wait again. If it needs a spec decision, ask the user with options and pass the decision to Devin verbatim (`message`). Record the decision in the state file |
| `waiting_for_approval` (network, secret, or other permission) | Pass it to the user, saying what is being requested |
| Only offers a recording or a UI check | Reply that it is not needed and wait again |
| `error` / `exit` | Report to the user with the latest message, then follow "When to stop" in step 7 |

A `suspended` session wakes up when you send it a `message`.

If it stays working for more than two hours, tell the user how it is going, with the latest message, and ask whether to keep waiting.

## 4. Verify the fixes

1. Look at the diff since the head SHA reviewed in the previous round. After `git fetch origin <head branch>`, if `git merge-base --is-ancestor <previous SHA> origin/<head branch>` is true, use `git diff <previous SHA>..origin/<head branch>`; if false (force push or rewritten history), use the whole PR diff
2. Fetch the threads:

   ```bash
   gh api graphql -F o='{owner}' -F r='{repo}' -F n=<number> -f query='query($o:String!,$r:String!,$n:Int!){repository(owner:$o,name:$r){pullRequest(number:$n){reviewThreads(first:100){nodes{id isResolved isOutdated path comments(first:20){nodes{databaseId author{login __typename} body url}}}}}}}'
   ```

3. Judge the **loop threads** (those whose first comment's `databaseId` is in the state file, resolved or not) and unresolved threads opened by bots (first comment's `author.__typename` is `Bot`). Do not resolve threads opened by other people; show them to the user

   | Verdict | Condition | Action |
   | --- | --- | --- |
   | Sound | The failure you pointed out is gone in the diff, and no new breakage came in | Resolve it if unresolved |
   | Insufficient | The failure remains, or the fix introduced a new failure | Record what is missing in the state file and reply in step 6. If Devin resolved it, unresolve it |
   | Disputed | Devin replied with a reason not to fix | Resolve it if the reason holds. If it needs a spec decision, ask the user |
   | Unanswered | No reply from Devin | Ask Devin to continue and go back to step 3. If the same thread is still unanswered after two re-asks, treat it as insufficient |

   ```bash
   gh api graphql -F id=<thread id> -f query='mutation($id:ID!){resolveReviewThread(input:{threadId:$id}){thread{isResolved}}}'
   gh api graphql -F id=<thread id> -f query='mutation($id:ID!){unresolveReviewThread(input:{threadId:$id}){thread{isResolved}}}'
   ```

4. Record the verdicts in the state file and go to step 5

## 5. Review

Check `gh pr checks <number>`. If anything is pending, wait with `gh pr checks <number> --watch` (Devin watches CI itself, so do not wake it). If anything failed, send Devin "Please keep fixing until CI is green, then report", go back to step 3, and count the consecutive bounces per check name.

When the round is 0 (step 4 has not run yet), run step 4.3 on bot threads only before reviewing, so that findings from automated reviewers that ran when the PR opened get judged here.

Once CI is green, record the PR head SHA (`gh pr view <number> --json headRefOid`) in the state file, then review the whole PR. If step 0 found a code review skill, pass it the PR number and follow it. Otherwise review the `gh pr diff <number>` diff yourself (the PR's diff, not the local branch's). Adjustments for this loop:

- Take the results as a report in the conversation. Only step 6 posts to GitHub
- If the skill has a step where it fixes findings itself, skip it. Devin does the fixing
- Review the whole PR in later rounds too (a fix can break something elsewhere). Pass the resolved findings, the findings you set aside, and the user's decisions from the state file to the review so they are not raised again

Also check the PR **against the design** yourself: is everything in scope present, and has nothing out of scope been touched?

Split findings into **must** and **optional**. A must is one where you can state, in one sentence, a failure the current diff causes if left unfixed (bug, security, data inconsistency, spec violation, repository rule violation). Divergence from the design counts as a spec violation, so it is a must. Everything else (readability, future-proofing, taste) is optional. If the review skill has its own severity labels, map them onto must/optional by this rule.

If the end conditions in step 7 are met, go to step 7. Otherwise go to step 6.

## 6. Post and hand off to Devin

Post only musts inline. Keep optional findings in the state file and show them to the user at the end; post only the ones the user wants adopted, in the next round (put them on the PR and Devin will fix them all, which leads to over-engineering).

Fetch the head SHA again right before posting. If it differs from the SHA recorded in step 5, Devin pushed during your review: post nothing, go back to step 3, and once Devin settles, go through steps 4 → 5 (without incrementing the round).

If there are no musts, post no review; only reply to the insufficient threads below and send Devin a `message`. If there are musts, post them as one review. In each comment, say what input or state breaks what, and give the smallest fix. Start each comment with the review skill's severity label if it has one, otherwise `[must]`. For a finding outside the diff, attach it to the nearest diff line and say it is outside the diff. If even one comment points at a line not in the diff, the whole review fails with 422 and nothing is posted; move that comment and post the whole review again.

```json
{
  "commit_id": "<head SHA>",
  "event": "COMMENT",
  "body": "outer loop round <k>: <N> must-fix findings",
  "comments": [
    { "path": "path/to/file", "line": 42, "side": "RIGHT", "body": "[must] ... (failure scenario and fix)" }
  ]
}
```

```bash
gh api 'repos/{owner}/{repo}/pulls/<number>/reviews' --method POST --input <state file directory>/review_r<k>.json
gh api 'repos/{owner}/{repo}/pulls/<number>/reviews/<review id>/comments' --jq '.[].id'
```

Record the comment IDs from the second command in the state file as loop threads. Keep `event` as `COMMENT`: approving and requesting changes stay with human reviewers.

Reply to the threads judged insufficient in step 4 now. Reply to the first comment of the thread (`comments.nodes[0].databaseId`); the API does not accept replies to replies.

```bash
gh api 'repos/{owner}/{repo}/pulls/<number>/comments/<first comment databaseId>/replies' -f body='...'
```

Finally, increment the round and send Devin a `message` (with the review URL if you posted one): "I posted a review on PR #<number> (<URL>). Please fix the unresolved threads with additional commits on the existing branch, push, and reply on each thread (please do not resolve them). Report back once CI is green." Devin may already have started from the GitHub notification; send it anyway. Record when you sent it in the state file and go back to step 3.

## 7. Finish

**End conditions**: all three hold.

- The step 5 review found no musts
- Every thread judged in step 4 is resolved
- CI is green

**When to stop**: if any of these happens, stop there and report to the user. If you stop before a PR exists, remove the label you put on the issue in step 1 (left in place, other automation keeps skipping the issue).

- The end conditions are still not met after 5 rounds. Findings that do not converge signal a gap in the design or the brief; include the remaining findings and where things keep going back and forth
- The same CI check still fails after two consecutive bounces (it may be flaky, or the base branch may be broken)
- Devin ended in `error` / `exit` in step 3, or did not move after a nudge

When you finish or stop, report the following to the user. Do not approve or merge the PR.

- PR URL and Devin session URL
- Number of rounds, and the number of musts per round
- The optional findings (the user decides whether to adopt them)
- Unresolved threads opened by other people
- Decisions the user made along the way
- Remaining human work (approve, merge, and for a stack, checking the base retarget)
