# AGENTS.md

## Project

`genai-capstone` — "Enterprise Knowledge Assistant": hybrid vector + knowledge-graph RAG, a
LangGraph agent with three tools, NeMo Guardrails, a Qdrant-backed semantic cache, Langfuse
tracing, a FastAPI service, and a Streamlit UI. Python only. Requires Python 3.11 exactly
(`>=3.11,<3.12`).

Read [README.md](README.md) first; [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) and
[docs/WALKTHROUGH.md](docs/WALKTHROUGH.md) cover the design. `docs/course/` explains every
file line by line.

## Setup and commands

Requires [`uv`](https://docs.astral.sh/uv/), Docker, and the credentials in `.env.example`.

```bash
make install   # uv venv --python 3.11 && uv pip install -e ".[dev]"
cp .env.example .env   # fill the five credentials
make up        # Neo4j 5 in Docker (host ports 7475 / 7688)
make check     # PHASE 0 GATE: verifies every credential live
make ingest    # rebuild Qdrant index + Neo4j graph
make serve     # FastAPI on :8000 (uvicorn app.api.main:app, --reload)
make ui        # Streamlit on :8501
```

Direct equivalents when the Makefile target is not what you want:

```bash
.venv/bin/pytest tests/ -v          # full suite (473 tests)
.venv/bin/pytest tests/test_agent.py -v
.venv/bin/ruff check app tests      # lint (line-length 100)
.venv/bin/ruff format app tests     # format
.venv/bin/python -m app.ingestion.run
```

`make eval` runs the safety scorecard against a **running** API and spends tokens (~$0.28).
Other evals live under `evals/` (promptfoo, RAGAS, load); see their own READMEs. RAGAS needs a
separate isolated venv because its pins conflict with the app's langchain range.

## Testing

- Config lives in `pyproject.toml`: `testpaths = ["tests"]`, `pythonpath = ["."]`.
- The suite never spends tokens — models, tools and the tracing client are injected/faked.
  Keep it that way; do not add tests that hit OpenRouter/Qdrant/Neo4j/Langfuse live.
- `tests/conftest.py` installs a **disabled** tracer before every test. A test that needs
  tracing installs its own.
- Several tests encode deliberate design limits and fail loudly if a dependency drifts (e.g.
  `test_an_uncued_phone_number_sits_exactly_on_the_threshold`). Do not "fix" these by relaxing
  the assertion without understanding the comment above it.

## Code conventions

- **Config is centralized.** Every knob is a field on `Settings` in `app/config.py`, read once
  via the cached `get_settings()`. Never read `os.environ` directly in a module.
- **Dependencies are pinned exactly on purpose.** `pyproject.toml` lists exact versions
  (`==`) not floors, because a wide floor already resolved across a langchain/langgraph major
  break once. Do not loosen a pin; if you introduce a package, pin it and note why in the
  comment block.
- **Comments and docstrings explain *why*, with the measurement behind the number.** The repo
  style is dense, specific module/function docstrings that record the decision and the
  evidence (often with a test name). Match that style when editing these files; a bare
  restatement of the code is not the convention here.
- **Boundaries are structural, not requested.** Examples: the agent unbinds tools at
  `agent_max_iterations` rather than asking the model to stop; citations are built from the
  retrieval ledger, never parsed from model output; free-form Cypher is unreachable from the
  model (it picks a template `Literal`, entities are bound parameters). Preserve these
  invariants.
- `from __future__ import annotations` is used throughout. Type hints are the norm.
- Logging via `logging.getLogger(__name__)`; no bare `print` in app code.

## Layout

```
app/config.py          settings; every knob with its rationale
app/ingestion/         loaders · normalize · chunker · embedding_model · vector_index · graph_builder · run
app/retrieval/         vector · graph · graph_queries · hybrid
app/agent/             graph (LangGraph loop) · tools · prompts
app/guardrails/        runner · actions · config/{config.yml,prompts.yml,rails/*.co}
app/llm/               client · tiering · cache
app/observability/     langfuse_client (never allowed to break a request)
app/api/               main · schemas · sessions · metrics · system_card
ui/streamlit_app.py    talks to the API over HTTP only
data/raw/              the ACME corpus
evals/                 promptfoo · ragas · safety · load · retrieval_probe.py
tests/                 pytest suite
docs/                  ARCHITECTURE.md · WALKTHROUGH.md · screenshots/ · course/
scripts/check_env.py   the Phase 0 credential gate
Dockerfile · start.sh · docker/   the Hugging Face Space image
```

## Before finishing a task

Run `.venv/bin/ruff check app tests` and `.venv/bin/pytest tests/` (both are offline and free)
and make sure they pass.
