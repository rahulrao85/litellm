# PR draft (delete this file before opening the PR)

Target repository: BerriAI/litellm, base branch `main` (checked with `python3 scripts/default_branch.py --branch`)

Head branch: `rahulrao85:fix/44575-latency-routing-second-router`

## Title

fix(router): record latency for every Router in a process under latency-based routing

## Body

## TLDR

Problem this solves:

- A second latency-based Router in one process never records latency
- Its routing never learns, so it looks like the strategy does nothing

How it solves it:

- Give the latency selector a per-router cache id, like least-busy already has
- The callback manager now keeps one latency selector per router

## User Flow

Before: a developer who creates a second `Router(routing_strategy="latency-based-routing")` in the same process gets no latency data on it

1. They build a Router with latency-based routing, then build a second one (a notebook re-run, a test, or an app holding two routers)
2. They send several `acompletion` calls through the second router
3. `router.cache.get_cache("<model_group>_map")` on the second router returns `None`, so every deployment looks equally fast

After: the same second router records latency for each call and routes to the faster deployment

1. They build the same two routers
2. They send the same `acompletion` calls through the second router
3. `router.cache.get_cache("<model_group>_map")` returns the recorded latencies for each deployment

## Relevant issues

Fixes #44575

## Pre-Submission checklist

**Please complete all items before asking a LiteLLM maintainer to review your PR**

- [x] I have added meaningful tests
- [x] The handful of test files covering my change pass locally, e.g. `uv run pytest tests/unit/<your_test_file>.py -v`. Leave the suites (`make test-unit-*`, `make test-unit`) to CI: it finishes in ~15 minutes where a laptop takes an hour or more
- [ ] My PR passes all required CI/CD checks (e.g., lint, schema.d.ts sync check, etc.)
- [x] My PR's scope is as isolated as possible; it only solves 1 specific problem
- [ ] I have received a Greptile **Confidence Score of at least 4/5** before requesting a maintainer review (Greptile reviews automatically once the PR is opened; only comment `@greptileai` to re-request a review after pushing changes)

## Screenshots / Proof of Fix

Shared setup: a script that builds N latency-based routers with one mocked deployment, sends three `acompletion` calls through the last one, and prints that router's latency cache

```python
import asyncio
import sys

from litellm import Router


def build():
    return Router(
        model_list=[
            {
                "model_name": "gpt",
                "litellm_params": {"model": "openai/gpt-4o-mini", "api_key": "sk-fake", "mock_response": "hi"},
                "model_info": {"id": "d1"},
            }
        ],
        routing_strategy="latency-based-routing",
    )


async def main(router_count: int):
    routers = [build() for _ in range(router_count)]
    last = routers[-1]
    for _ in range(3):
        await last.acompletion(model="gpt", messages=[{"role": "user", "content": "hi"}])
    await asyncio.sleep(1)
    print(f"routers={router_count} last router latency cache:", last.cache.get_cache(key="gpt_map"))


asyncio.run(main(int(sys.argv[1])))
```

### Before (68c0a972)

1. `for n in 1 2; do uv run python repro.py $n; done`
2. Output:

```
routers=1 last router latency cache: {'d1': {'latency': [0.008041900000000001, 0.0492502, 0.0002559], '2026-10-07-06-48': {'tpm': 90, 'rpm': 3}}}
routers=2 last router latency cache: None
```

### After (28b0827c)

1. `for n in 1 2; do uv run python repro.py $n; done`
2. Output:

```
routers=1 last router latency cache: {'d1': {'latency': [0.010795599999999999, 0.05632805, 0.00032845], '2026-10-07-06-48': {'tpm': 90, 'rpm': 3}}}
routers=2 last router latency cache: {'d1': {'latency': [0.00824965, 0.0461887, 0.0002473], '2026-10-07-06-48': {'tpm': 90, 'rpm': 3}}}
```

This run uses `mock_response`, so it costs nothing and does not hit a real provider. The repo's AGENTS.md asks for a live proxy run against a real provider, so redo this section with a real OpenAI key (drop `mock_response`) before asking for review

This PR was prepared with help from an AI coding assistant (Claude Code). I reviewed the diff, the test and the before and after runs myself

---

## What was wrong, the root cause and the fix (notes for you, not part of the PR body)

What was wrong: when two Router objects with `routing_strategy="latency-based-routing"` exist in one Python process, only the first one learns latency. The second router's latency cache stays empty forever, so it cannot prefer the faster deployment. That matches the issue: manually appending `router.lowestlatency_logger` to `litellm.callbacks` "fixes" it

Root cause: each router builds its own `LowestLatencyLoggingHandler` and registers it with `litellm.logging_callback_manager.add_litellm_callback`. That manager drops a logger when another logger in the list has the same class name and the same plain public attributes (strings, bools, ints). The latency handler only had `router_cache` and `routing_args`, which are objects, so every latency handler looked identical and the second router's handler was silently dropped. The least-busy handler hit the same bug earlier and was fixed by adding `self.router_cache_id = str(id(router_cache))`

The fix: one line in `litellm/router_strategy/lowest_latency.py` that adds the same `router_cache_id` attribute. Each router's handler now has a different key, so each one gets registered and records into its own router's cache. Re-initialising one router (for example `update_settings`) still does not pile up handlers, because the router removes its old handler by identity before adding the new one

How it was tested:

- New regression test `test_second_router_in_one_process_records_its_own_latency` in `tests/unit/router_strategy/test_lowest_latency.py`. It failed before the fix (second router cache had no latency) and passes after it
- Related suites pass: `tests/unit/router_strategy/` (minus `test_litellm_encoder.py`, which needs the optional `semantic_router` package), `tests/unit/test_router/test_router.py`, `tests/unit/test_router_get_settings.py`, `tests/unit/test_router_weighted_failover.py`, `tests/unit/test_lowest_latency_zero_tokens.py`, `tests/unit/types/test_litellm_params.py`, and the proxy router settings and config tests. 4743 passed. The 18 failures are all semantic router tests that fail identically on the unchanged base commit because `semantic_router` is not installed here
- Lint: `ruff format --check`, `ruff check` (code and tests config), the ruff strict gate, the type discipline gate, the test quality gate, basedpyright on `tests/e2e`, the circular import check and the import safety check all pass
- Type check: the full basedpyright budget gate could not build its own environment here, because the sandbox proxy blocks one private archive (`litellm-admin-mcp`). I ran basedpyright directly on the changed source file at base and at head: no new errors in `litellm/`. CI will run the full gate

Rules from the upstream repo that apply here:

- Sign the CLA at https://cla-assistant.io/BerriAI/litellm before or right after opening the PR, or it cannot be merged
- Conventional Commits for the title and commits, Conventional Branches for the branch name (`fix/...` fits)
- At least one meaningful unit test under `tests/unit/`, mirroring the source path, mocked only
- Run `make lint` and the covering test files; leave the full unit suite to CI. Do not touch the `*-budget.json` files
- Fill the PR template, drop empty sections, aim for a Greptile score of 4/5 or more before asking for review. The repo has no formal AI policy or required label; the disclosure line above is voluntary

## Explain it to Govu

1. LiteLLM keeps one global list of "listeners" that hear about every finished LLM call
2. Latency routing adds a listener per router that writes call times into that router's own notebook (cache)
3. The list refuses duplicates, and it judged "duplicate" only by class name plus simple text and number fields
4. Every latency listener looked identical, so a second router's listener was turned away and its notebook stayed empty
5. I gave each listener a field that is unique per router (the id of its cache), exactly like the least-busy router already does, so each router now gets its own listener
