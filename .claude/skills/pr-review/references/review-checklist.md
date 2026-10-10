# Review checklist — genai-capstone conventions

This distils `AGENTS.md` into questions to ask of a diff. These are not style opinions:
in this repo, breaking them is a defect. Cite the line and the reason when flagging.

## Configuration

- Is every new knob a field on `Settings` in `app/config.py`, read via the cached
  `get_settings()`?
- Does any module read `os.environ` directly? That is a violation — flag it.
- Are defaults and rationale recorded alongside the field, matching the existing style?

## Dependencies

- Does `pyproject.toml` pin any new package with `==`? Floors (`>=`) are forbidden here
  because a wide floor once resolved across a langchain/langgraph major break.
- Is there a comment explaining *why* the package was added, in the existing comment block?
- Did an existing pin get loosened? Treat that as blocking unless justified.

## Structural boundaries (must stay structural)

The agent enforces limits in code, not by asking the model. Flag any change that moves a
boundary from structure to prompt:

- Tools are unbound at `agent_max_iterations` — is that still enforced in code?
- Citations are built from the retrieval ledger, never parsed from model output.
- Free-form Cypher is unreachable from the model: it picks a template `Literal`, entities
  are bound parameters. Any change that lets model text reach a query template is blocking.

## Tests

- Config lives in `pyproject.toml`: `testpaths = ["tests"]`, `pythonpath = ["."]`.
- Do new tests avoid spending tokens? Models, tools and the tracing client must be
  injected or faked. A test that hits OpenRouter/Qdrant/Neo4j/Langfuse live is blocking.
- `tests/conftest.py` installs a disabled tracer before every test; a test that needs
  tracing installs its own.
- Some tests encode deliberate design limits and fail loudly if a dependency drifts
  (e.g. `test_an_uncued_phone_number_sits_exactly_on_the_threshold`). A PR that "fixes"
  such a test by relaxing the assertion is suspect — check the comment above it first.

## Style

- `from __future__ import annotations` is used throughout; new modules should have it.
- Type hints are the norm.
- Logging via `logging.getLogger(__name__)`; no bare `print` in `app/` code.
- Line length 100 (`ruff`).
- Comments and docstrings explain **why**, with the measurement behind a number — often
  citing a test name. A bare restatement of the code is against convention.

## Gates to run before merge

Both are offline and free; a PR is not ready if either fails:

```bash
.venv/bin/ruff check app tests
.venv/bin/pytest tests/
```

`make check` verifies live credentials and is part of the Phase 0 gate — only relevant
when the PR touches credentials/config wiring.

## CI

`gh pr checks <n>` gives per-check status. For a failing check, pull the first real error:

```bash
gh run view <run-id> --log-failed
```
