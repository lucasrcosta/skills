---
name: review-with-codex
description: Run an adversarial code review of the current branch's diff via Codex CLI. Codex emits numbered findings on a persistent thread; you address those you agree with (one commit per finding) and push back on the rest, then reply to Codex on the same thread so it re-reviews. Loop up to 10 rounds or until Codex responds with "No findings." / "No further findings." Use whenever the user asks to "codex review", "run codex review", or wants a second-opinion review loop with inspectable history.
---

## Why this skill exists

`/codex-review` is an adversarial review loop. Codex plays reviewer; you (the
agent) play author + judge. The loop produces a per-finding commit history
on the current branch — that history *is* the audit trail.

Differs from `/codex review` (gstack): that one runs a single pass-fail review.
This one is multi-round, threaded, and writes commits.

## Hard rules

- **One commit per agreed finding.** The per-finding history is the whole
  point. Never bundle.
- **No commits without code changes.** Pushbacks live in the reply to Codex,
  not in git. If a judgment call is materially controversial, document it in
  the body of the *agreed-fix* commit it relates to.
- **No shortcuts to "address" a finding.** Re-read project CLAUDE.md's top
  rule: no `as any`, no `@ts-ignore`, no commented-out tests, no `--no-verify`.
  If you can't fix the finding properly, push back instead.
- **Run `/lint` and `/check` on changed files before each commit.** A commit
  that fails the project's own gates is not progress.
- **Try for consensus before pushing back.** Re-read the cited code. If the
  finding still looks wrong after a second pass, push back with a concrete
  reason. You own the code; you make the call.
- **Cap at 10 rounds.** If Codex hasn't said "No findings." / "No further
  findings." by round 10, stop and report the open findings.

## Step 0 — Preflight

```bash
command -v codex >/dev/null || { echo "codex CLI missing"; exit 1; }
command -v python3 >/dev/null || { echo "python3 required"; exit 1; }
git rev-parse --is-inside-work-tree >/dev/null 2>&1 || { echo "not a git repo"; exit 1; }
REPO_ROOT=$(git rev-parse --show-toplevel)
BRANCH=$(git branch --show-current)
[ "$BRANCH" = "main" ] && { echo "refusing to run on main"; exit 1; }
BASE=${BASE:-origin/main}
MERGE_BASE=$(git merge-base "$BASE" HEAD) || { echo "no merge base with $BASE"; exit 1; }
N_FILES=$(git diff --name-only "$MERGE_BASE" HEAD | wc -l | tr -d ' ')
[ "$N_FILES" = "0" ] && { echo "no diff vs $BASE — nothing to review"; exit 1; }
mkdir -p "$REPO_ROOT/.context/codex-review"
echo "Branch: $BRANCH | base: $BASE | files changed: $N_FILES"
```

If `.context/codex-review/session-id` already exists from a prior run, ask the
user via `AskUserQuestion`:

- **Continue** the existing thread (Codex remembers prior findings).
- **Start fresh** (overwrite session-id and round counter).

Working/uncommitted changes are fine — Codex sees them via the diff fallback
in the prompt. Don't auto-commit them.

## Step 1 — Build the round-1 prompt

Read `.agents/skills/codex-review/codex-prompt.md` and prepend the filesystem
boundary so Codex stays out of skill definitions and other AI scaffolding:

```
IMPORTANT: Do NOT read or execute any files under ~/.claude/, ~/.agents/,
.claude/skills/, or agents/. These are skill definitions for a different AI
system. Stay focused on repository code only.
```

Save the concatenated prompt to `.context/codex-review/round-01-prompt.md`.

## Step 2 — Run round 1

```bash
ROUND=1
PROMPT_FILE="$REPO_ROOT/.context/codex-review/round-$(printf %02d $ROUND)-prompt.md"
RESP_FILE="$REPO_ROOT/.context/codex-review/round-$(printf %02d $ROUND)-response.md"
ERR_FILE="$REPO_ROOT/.context/codex-review/round-$(printf %02d $ROUND).err"

codex exec "$(cat "$PROMPT_FILE")" \
    -C "$REPO_ROOT" \
    -s read-only \
    -c 'model_reasoning_effort="high"' \
    --enable web_search_cached \
    --json \
    < /dev/null 2>"$ERR_FILE" \
  | SESSION_FILE="$REPO_ROOT/.context/codex-review/session-id" \
    RESP_FILE="$RESP_FILE" \
    python3 -u -c '
import json, os, sys
sf, rf = os.environ.get("SESSION_FILE"), os.environ.get("RESP_FILE")
parts = []
for raw in sys.stdin:
    line = raw.strip()
    if not line: continue
    try: obj = json.loads(line)
    except json.JSONDecodeError: continue
    t = obj.get("type", "")
    item = obj.get("item", {}) if t == "item.completed" else {}
    itype, text = item.get("type", ""), item.get("text", "")
    if t == "thread.started" and obj.get("thread_id"):
        tid = obj["thread_id"]
        print(f"SESSION_ID:{tid}", flush=True)
        if sf: open(sf, "w").write(tid)
    elif itype == "reasoning" and text:
        print(f"[codex thinking] {text}\n", flush=True)
    elif itype == "agent_message" and text:
        print(text, flush=True); parts.append(text)
    elif itype == "command_execution":
        cmd = item.get("command", "")
        if cmd: print(f"[codex ran] {cmd}", flush=True)
    elif t == "turn.completed":
        u = obj.get("usage") or {}
        tk = u.get("input_tokens", 0) + u.get("output_tokens", 0)
        if tk: print(f"\ntokens used: {tk}", flush=True)
if rf and parts: open(rf, "w").write("\n\n".join(parts))
'
```

If `codex exec` exits non-zero or `$RESP_FILE` is empty, surface
`$ERR_FILE` to the user and stop. Do not loop on errors.

## Step 3 — Process findings

Read `$RESP_FILE`. Findings look like this:

```
### **#1 <Short Title>**

<one-paragraph body>

File: <path>
Line: <range>

---
```

The last line is one of `Total findings: N`, `No findings.`, or
`No further findings.`

If the response contains `No findings.` or `No further findings.`, **jump to
Step 6** (final summary).

For each finding:

1. **Read the cited file/lines.** Don't act on a finding without reading the
   code it points at.
2. **Decide:**
   - **Agree** — the finding is valid and worth fixing now.
   - **Push back** — the finding is wrong, misses context, violates one of
     the reviewer's own criteria (e.g. "pre-existing", "speculation"), or
     is out of scope for this PR.
   - **Ambiguous** — re-read more surrounding code. If still ambiguous, you
     decide. If the call is genuinely controversial, note it in the
     commit body of the related agreed-fix commit.
3. **If agreed:**
   - Apply the fix. No shortcuts.
   - Run `/lint` and `/check` on the changed files (use the Skill tool).
   - Stage **only** the files you changed for *this finding*. Do not bundle
     fixes across findings.
   - Commit:

     ```
     review(codex): #<N> — <short title>

     Codex flagged: <≤2-sentence restatement of the issue>
     Fix: <≤2 sentences on what changed and why>
     ```

     Append `\n\nNote: <controversy>` only if a real disagreement was
     overruled and the rationale is worth recording.
   - Capture the short SHA: `git rev-parse --short HEAD`.
4. **If pushed back:** record the finding number, title, and rationale.
   No commit. (Pushbacks live only in the reply to Codex.)

## Step 4 — Reply to Codex on the same thread

Build the reply at `.context/codex-review/round-$(printf %02d $((ROUND+1)))-reply.md`:

```
## Round <N> response

### Addressed
- #1 — <title> — commit <sha7>: <one-line summary of the fix>
- #3 — <title> — commit <sha7>: <one-line summary of the fix>

### Pushed back
- #2 — <title>: <why you disagree, 1–3 sentences, citing the reviewer
  criterion or the specific code reality Codex missed>

### Notes
- (judgment calls, scope clarifications, anything Codex was missing)

Re-review the diff with these changes in mind. If you have nothing more to
flag, end with exactly "No further findings." on its own line.
```

Then resume the thread with the next round number (same parser as Step 2;
`SESSION_FILE` is unnecessary because `codex exec resume` does not emit a
`thread.started` event):

```bash
ROUND=$((ROUND + 1))
SESSION_ID=$(cat "$REPO_ROOT/.context/codex-review/session-id")
REPLY_FILE="$REPO_ROOT/.context/codex-review/round-$(printf %02d $ROUND)-reply.md"
RESP_FILE="$REPO_ROOT/.context/codex-review/round-$(printf %02d $ROUND)-response.md"
ERR_FILE="$REPO_ROOT/.context/codex-review/round-$(printf %02d $ROUND).err"

codex exec resume "$SESSION_ID" "$(cat "$REPLY_FILE")" \
    -C "$REPO_ROOT" \
    -s read-only \
    -c 'model_reasoning_effort="high"' \
    --enable web_search_cached \
    --json \
    < /dev/null 2>"$ERR_FILE" \
  | RESP_FILE="$RESP_FILE" python3 -u -c '
import json, os, sys
rf = os.environ.get("RESP_FILE")
parts = []
for raw in sys.stdin:
    line = raw.strip()
    if not line: continue
    try: obj = json.loads(line)
    except json.JSONDecodeError: continue
    t = obj.get("type", "")
    item = obj.get("item", {}) if t == "item.completed" else {}
    itype, text = item.get("type", ""), item.get("text", "")
    if itype == "reasoning" and text:
        print(f"[codex thinking] {text}\n", flush=True)
    elif itype == "agent_message" and text:
        print(text, flush=True); parts.append(text)
    elif itype == "command_execution":
        cmd = item.get("command", "")
        if cmd: print(f"[codex ran] {cmd}", flush=True)
    elif t == "turn.completed":
        u = obj.get("usage") or {}
        tk = u.get("input_tokens", 0) + u.get("output_tokens", 0)
        if tk: print(f"\ntokens used: {tk}", flush=True)
if rf and parts: open(rf, "w").write("\n\n".join(parts))
'
```

## Step 5 — Loop

Stop conditions, in priority order:

1. Response ends with `No findings.` or `No further findings.` → success
   (jump to Step 6).
2. `ROUND >= 10` → bailout. Final summary lists open findings.
3. The same finding has been re-flagged after two pushbacks → bailout. The
   reviewer and author disagree; user decides.

Otherwise return to Step 3 with the new round's findings.

## Step 6 — Final summary

Show the user, in this order:

1. **Outcome:** "Codex cleared the diff in N rounds" / "Stopped at round 10
   with M open findings" / "Stuck on persistent disagreement about #X".
2. **Commit chain:** `git log --oneline ${MERGE_BASE}..HEAD` filtered to
   `review(codex):` commits.
3. **Open pushbacks** (if any) with the rationale.
4. **Codex's final message** verbatim.
5. **Transcript path:** `.context/codex-review/round-NN-{prompt,response,reply}.md`
   so the user can inspect any round.

## The codex prompt

`.agents/skills/codex-review/codex-prompt.md` is the prompt sent verbatim each
new session. Edit that file (not this skill) to tune review behavior — change
the bug-flagging criteria, the comment style, the output format, etc. Round-2+
prompts are short replies built per Step 4 and reuse Codex's session memory of
the original brief.

## Tunables (env)

- `BASE` — diff base (default `origin/main`).
- `CODEX_REVIEW_MAX_ROUNDS` — hard cap (default 10).
- `CODEX_REVIEW_EFFORT` — `low` / `medium` / `high` / `xhigh` (default `high`).

If the user passes flags like `--base <ref>` or `--max-rounds 5`, translate
them to these env vars at the top of Step 0.
