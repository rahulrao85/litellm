# Upstream contributions: status and next steps

Owner: Rahul Rao (GitHub `rahulrao85`)

Last updated: 7 October 2026

Fork: https://github.com/rahulrao85/litellm

Upstream: https://github.com/BerriAI/litellm (default branch `main`)

This one document has everything needed to finish the work in a new session. Read sections 1 to 4 yourself first. Section 6 is a prompt you can paste into a new session

---

## 1. Status at a glance

| Item | Status |
|---|---|
| Issue picked | BerriAI/litellm#44575, "latency-based-routing: LowestLatencyLoggingHandler callback not reliably registered" |
| Issue still free? | Yes, as of 7 October 2026: open, no assignee, no linked or competing PR |
| Failing test written first | Done. It failed on the original code |
| Fix | Done. One line in `litellm/router_strategy/lowest_latency.py` |
| Unit tests | Pass. 4,743 related tests pass. 18 unrelated failures happen on the original code too (optional `semantic_router` package missing in the sandbox) |
| Lint and gates | Pass, except the full basedpyright gate, which could not install one blocked package in the sandbox. Direct basedpyright run on the changed source shows no new errors |
| Pushed to fork | Done. Branch `fix/44575-latency-routing-second-router` |
| PR opened upstream | **No.** Waiting for you, see section 4 |
| CLA signed | Unknown. **You must check**, see section 4 |
| Real-provider proof | **Not done.** Needs an OpenAI key, see section 5 step 3 |

---

## 2. Where everything is

| What | Where |
|---|---|
| Fix branch on your fork | `fix/44575-latency-routing-second-router` |
| Fix commit | `28b0827c` fix(router): register each router's latency selector so a second Router records latency |
| PR draft commit | `ed860a34` adds `PR_DRAFT.md` on the fix branch (title, body, proof, explanation) |
| This document | Branch `claude/quirky-wright-g02l4h`, file `upstream-contributions/NEXT_STEPS.md` |
| Changed source file | `litellm/router_strategy/lowest_latency.py` (one added line) |
| New test | `tests/unit/router_strategy/test_lowest_latency.py`, test `test_second_router_in_one_process_records_its_own_latency` |

Base commit for everything: `68c0a972` (fork `main` and upstream `main` were identical on 7 October 2026)

---

## 3. The bug and the fix in plain words

**What was wrong.** When two `Router` objects using `routing_strategy="latency-based-routing"` exist in one Python process, only the first one learns latency. The second router's latency cache stays empty, so it cannot prefer the faster deployment

**Root cause.** Each router creates its own latency listener and adds it to LiteLLM's global callback list. That list rejects a new listener if an existing one has the same class name and the same simple text and number fields. The latency listener had no such field that differs between routers, so every latency listener looked identical and the second one was silently dropped

**Fix.** Add `self.router_cache_id = str(id(router_cache))` to the latency listener. Each router's cache is a different object, so each router's listener now looks different and gets registered. The least-busy listener already uses exactly this line for the same bug, so the fix follows an existing pattern in the codebase

**Proof (mocked, no provider cost).** A script that builds N routers and sends three calls through the last one:

| Run | Original code | With fix |
|---|---|---|
| 1 router | latency recorded | latency recorded |
| 2 routers | `None` (nothing recorded) | latency recorded |

**Things I checked after the first summary**

- The non-async `router.completion()` path also works with one router. My earlier remark about a "possible separate bug" was wrong: the first cold run simply took longer than the 1-second wait in my script. With a 3-second wait it records latency every time. Nothing more to do there
- The same duplicate-dropping bug also affects three other strategies: `usage-based-routing`, `usage-based-routing-v2` and `cost-based-routing`. Least-busy and (now) latency are fine. This is a scope decision for you, see decision D4

**Explain it to Govu (5 lines)**

1. LiteLLM keeps one global list of "listeners" that hear about every finished LLM call
2. Latency routing adds a listener per router that writes call times into that router's own notebook (cache)
3. The list refuses duplicates, and it judged "duplicate" only by class name plus simple text and number fields
4. Every latency listener looked identical, so a second router's listener was turned away and its notebook stayed empty
5. I gave each listener a field that is unique per router (the id of its cache), exactly like the least-busy router already does, so each router now gets its own listener

---

## 4. Decisions that need you

Each decision has my recommendation. None of them is urgent; the issue was still unclaimed on 7 October 2026, but popular issues in this repository often get a competing PR within a day or two, so check again before you start (section 5 step 1)

| # | Decision | Options | My recommendation |
|---|---|---|---|
| D1 | Sign the CLA | Sign at https://cla-assistant.io/BerriAI/litellm with your GitHub account, or do not contribute | **Sign it before opening the PR.** It is a hard merge requirement. Read it first: it is a legal agreement in your personal name, separate from CloudRock |
| D2 | Proof of fix with a real provider | (a) Run the script with a real OpenAI key, costing well under USD 0.01. (b) Open the PR with only the mocked proof | **(a).** The repository's AGENTS.md explicitly asks for real-provider proof, and reviewers check for it |
| D3 | `PR_DRAFT.md` on the fix branch | (a) Delete it before opening the PR. (b) Keep it | **(a) Delete it.** Otherwise it appears in the PR diff. Its content is also saved in this document |
| D4 | Scope: the three other strategies with the same bug | (a) Keep this PR to latency only, mention the others, offer a follow-up PR. (b) Widen this PR to fix all four. (c) Fix the root cause once in the router registration code instead | **(a).** The issue is about latency, the repo rules say "one problem per PR", and a one-line fix is easy to review and merge. A second small PR for the other three strategies is a good next contribution once the first is merged |
| D5 | Branch name | Keep `fix/44575-latency-routing-second-router`, or rename | **Keep it.** CONTRIBUTING.md asks for Conventional Branches, and `fix/...` matches. The "no slash" rule in AGENTS.md is aimed at internal maintainers |
| D6 | AI disclosure | Keep the voluntary disclosure line in the PR body, or remove it | **Keep it.** The repository has no formal AI policy, but honest disclosure protects your reputation, and the repository itself encourages AI agents |
| D7 | Ask the reporter whether they use one router or two | Post a short comment on the issue, or skip | **Optional, low priority.** If you want certainty that the fix covers their case, the text is ready in section 7. You said not to touch upstream, so this is only if you choose to |

---

## 5. Step-by-step plan to open the PR

Run these on your own machine or in a new cloud session that has the fork checked out. Expected total time: about 30 to 45 minutes

### Step 1: confirm the issue is still free (2 minutes)

Open https://github.com/BerriAI/litellm/issues/44575 and check that it has no assignee and no linked PR. Also search https://github.com/BerriAI/litellm/pulls?q=is%3Apr+latency+router+callback for a new competing PR. If someone opened one, stop and pick a new issue (section 8)

### Step 2: update the branch with the latest upstream main (5 minutes)

```bash
git clone https://github.com/rahulrao85/litellm && cd litellm
git remote add upstream https://github.com/BerriAI/litellm
git fetch upstream main
git checkout fix/44575-latency-routing-second-router
git merge upstream/main
uv run pytest tests/unit/router_strategy/test_lowest_latency.py -v
```

If the merge has a conflict in `lowest_latency.py`, check whether upstream already fixed the bug. If it did, stop: the work is no longer needed

### Step 3: real-provider proof (10 minutes, under USD 0.01)

Save this as `repro.py` outside the repository (do not commit it):

```python
import asyncio
import sys

from litellm import Router


def build():
    return Router(
        model_list=[
            {
                "model_name": "gpt",
                "litellm_params": {"model": "openai/<cheapest current OpenAI chat model>"},
                "model_info": {"id": "d1"},
            }
        ],
        routing_strategy="latency-based-routing",
    )


async def main(router_count: int):
    routers = [build() for _ in range(router_count)]
    last = routers[-1]
    for _ in range(3):
        await last.acompletion(model="gpt", messages=[{"role": "user", "content": "Say hi"}], max_tokens=5)
    await asyncio.sleep(2)
    print(f"routers={router_count} last router latency cache:", last.cache.get_cache(key="gpt_map"))


asyncio.run(main(int(sys.argv[1])))
```

Pick the model: the repo rule says to use the latest model in the family, so choose the cheapest current OpenAI chat model listed in `model_prices_and_context_window.json`

Run it on the original code and on the fix:

```bash
export OPENAI_API_KEY=...   # never commit or paste this anywhere
git checkout 68c0a972 -- litellm/router_strategy/lowest_latency.py
for n in 1 2; do uv run python repro.py $n; done      # Before: routers=2 shows None
git checkout HEAD -- litellm/router_strategy/lowest_latency.py
for n in 1 2; do uv run python repro.py $n; done      # After: routers=2 shows latencies
```

Copy both outputs into the "Screenshots / Proof of Fix" section of the PR body, replacing the mocked runs, and put the commit hashes in the Before and After headings

### Step 4: remove the draft file and push (2 minutes)

```bash
git rm PR_DRAFT.md
git commit -m "chore: drop PR draft notes"
git push origin fix/44575-latency-routing-second-router
```

Before deleting it, copy the title and body from `PR_DRAFT.md` (also available in section 3 of this document)

### Step 5: run the checks once more (10 to 20 minutes)

```bash
make install-dev
make lint
uv run pytest tests/unit/router_strategy/ tests/unit/test_router_get_settings.py -q
```

On your own machine `make lint` should be able to install every package, so the full basedpyright gate will run. Do not edit any `*-budget.json` file even if a gate complains; ask the next session to fix the code instead

### Step 6: open the PR (5 minutes)

1. Go to https://github.com/BerriAI/litellm/compare/main...rahulrao85:litellm:fix/44575-latency-routing-second-router
2. Title: `fix(router): record latency for every Router in a process under latency-based routing`
3. Body: the body from `PR_DRAFT.md`, with the real-provider proof from step 3. Remove the line that says "redo this section with a real OpenAI key"
4. If you chose D4 (a), add a short line under TLDR: "The usage-based (v1, v2) and cost-based selectors have the same dedup problem. Happy to send a follow-up PR"
5. Submit, then sign the CLA if the CLA bot asks

### Step 7: after the PR is open (over the following days)

| Check | What to do |
|---|---|
| CLA bot | Sign if it asks |
| CI | All required checks must be green. If one fails, open a new session with the prompt in section 6 and paste the failing check name |
| Greptile review | Needs a confidence score of at least 4/5 before you ask a maintainer. If it is lower, fix what it flags, push, and comment `@greptileai` once to re-review |
| Other bots (Veria, Bugbot) | Read their findings. Fix real ones; reply briefly to false positives |
| Maintainer questions | Use section 3 ("Explain it to Govu") to answer in your own words |
| Slow review | After about a week, ping politely in the LiteLLM Slack channel `#pr-review` |

---

## 6. Prompt to paste into a new session

```text
I am rahulrao85. My fork is rahulrao85/litellm, upstream is BerriAI/litellm.
Read upstream-contributions/NEXT_STEPS.md on branch claude/quirky-wright-g02l4h of my fork first.
Work only on branch fix/44575-latency-routing-second-router.
Tasks: (1) merge upstream/main into the branch and re-run tests/unit/router_strategy/test_lowest_latency.py,
(2) run make lint and fix anything my change caused, never edit *-budget.json files,
(3) remove PR_DRAFT.md in its own commit, (4) push to origin.
Do NOT open a PR, comment on issues, or touch upstream. Stop and report.
```

If the PR is already open and CI or a reviewer asks for changes, use this instead:

```text
My PR from rahulrao85:fix/44575-latency-routing-second-router to BerriAI/litellm is open.
Read upstream-contributions/NEXT_STEPS.md on branch claude/quirky-wright-g02l4h of my fork.
Here is the failing check or review comment: <paste it>.
Reproduce it, make the smallest fix, run the related tests and make lint, push to the same branch.
Do not post any GitHub comment without showing me the text first.
```

---

## 7. Ready-to-paste texts

**Optional comment on the issue (D7).** Only if you decide to ask the reporter:

> Thanks for the report. Were two Router instances created in one process when this happened? A second latency router's callback gets deduplicated away

**PR follow-up line for D4 (a):**

> The usage-based (v1, v2) and cost-based selectors have the same dedup problem. Happy to send a follow-up PR

---

## 8. If this issue is taken: other candidates (checked 7 October 2026)

| Issue | Status when checked | Notes |
|---|---|---|
| BerriAI/litellm#44852 Headroom guardrail skips tool-output-only Responses requests | No PR | The expected fix is a design change; likely over 150 lines |
| BerriAI/litellm#44030 Langfuse pass-through logged as an LLM generation | Related PR #44480 exists | Check that PR's state first |
| Follow-up to #44575 | No issue yet | Same dedup bug in usage-based (v1, v2) and cost-based routing. Natural second PR after this one merges |

Issues that already had competing PRs on 7 October 2026 (skip): #44743, #44694, #44792, #44535, #32332

---

## 9. Upstream rules that apply

- **CLA:** you must sign it before the PR can be merged
- **Naming:** Conventional Commits for commits and the PR title, Conventional Branches for branch names
- **Tests:** at least one meaningful mocked unit test, in the `tests/unit/` file that mirrors the source path. For bug fixes, extend the existing test file
- **Checks:** run `make lint` plus the covering test files; CI runs the full suite. Never edit the `*-budget.json` files
- **PR body:** fill the PR template and remove empty sections. The upstream maintainers expect a Greptile score of at least 4/5 before review. There is no formal AI policy or required label

---

## 10. Sandbox limits seen in the cloud session (for the next session)

- The GitHub API tools could not read upstream issues (the session was scoped to the fork only). Public issue pages could still be read with a web fetch
- The proxy blocked the private package archive `litellm-admin-mcp`. Workaround: `uv sync --inexact --frozen --extra proxy --group proxy-dev --group e2e-dev --no-install-package litellm-admin-mcp`, then `uv run --no-sync ...`
- Because of that block, the full basedpyright gate could not build its own environment there. On your own machine it should work
- The optional `semantic_router` package is not installed, so 18 semantic router tests fail there on the original code too. They are unrelated
