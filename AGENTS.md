# AGENTS.md: Loan Application Agent (portfolio project)

This file tells every coding agent in this repo how to work here. Codex reads it directly, and Claude Code reads it through `CLAUDE.md`. Read it fully, together with the constitution, before you plan or write code. If something is unclear or conflicts with a request, stop and ask the humans. Do not guess.

Agents must not edit this file or the constitution unless a human asks in the current session.

## 1. Project in one paragraph

A demo **AI loan application agent** for a fictional Indonesian lender, **"Demo Lending"**. Applicants submit small personal loan applications. An AI agent gathers facts through tools (identity check, a mock credit bureau served over MCP, installment and debt-service calculations, policy rules) and recommends approve, reject, or refer to a human. A human loan officer decides, and only approved loans can be disbursed, exactly once. Two junior engineers build it as a **portfolio project**: production-style code at a small scale, with synthetic data only. It is not financial advice and does not claim regulatory compliance.

Skills the project must visibly demonstrate:
- LLM tool calling and our own agent loop;
- Model Context Protocol (our own MCP server and client adapter);
- FastAPI + PostgreSQL;
- structured output;
- guardrails;
- LLM evaluation;
- pytest;
- Docker and Google Cloud Run;
- observability.

Never pick a design that hides one of these, for example a hosted agent service or an agent framework that replaces our loop. Ask first.

## 2. How we work: Spec-Driven Development with GitHub Spec Kit

- **Constitution**: `.specify/memory/constitution.md` holds the non-negotiable rules (money logic, human approval, PII, synthetic data, testing, honest numbers). Every plan and review is checked against it.
- **Features**: each feature lives in `specs/<NNN-name>/` (`spec.md`, `plan.md`, `tasks.md`, and other design files).
- **Command flow per feature:**
  `specify → clarify → plan → tasks → analyze → implement → converge`.
  - Run each step separately and let a human review the result before the next one.
  - `spec.md` holds the **what and why** only. Tech choices belong in `plan.md`.
  - Run `analyze` before `implement`. Fix issues at the step that owns them.
  - Implement in stages (Setup + Foundational first, then one user story at a time).
  - Repeat `implement` and `converge` until converge reports **Converged**.
- **Invocation syntax:**

  | Agent | Syntax | Example |
  |---|---|---|
  | Claude Code | `/speckit-<command>` | `/speckit-plan` |
  | Codex | `$speckit-<command>` | `$speckit-plan` |

- **Shared Spec Kit setup.** Claude is the default integration and Codex is installed alongside it. Do **not** do any of these unless both humans agree:
  - run `specify integration use`, `switch`, or `upgrade`;
  - add Spec Kit extensions or presets (they register only for the default integration);
  - add the agent-context extension.
- **Active feature.** Spec Kit finds the active feature from `.specify/feature.json` or the `SPECIFY_FEATURE_DIRECTORY` environment variable, not from the git branch.
  - Each human sets `SPECIFY_FEATURE_DIRECTORY` before starting a session.
  - `.specify/feature.json` is gitignored and never committed.
  - If the active feature is not the one named in the task, stop and ask.

## 3. Feature roadmap

The specs for all features are written first on one branch and merged, so feature numbers never collide. After that, each feature is planned and built on its own branch.

| Feature | Depends on | Notes |
|---|---|---|
| 001 Loan application intake and policy evaluation | none | Also sets up the repo, tooling, and CI |
| 002 Credit bureau over MCP | none | Can be built in parallel with 001 |
| 003 AI recommendation agent | 001, 002 | Our own agent loop with tool calling |
| 004 Officer approval and safe disbursement | 001 | State machine and idempotency |
| 005 Guardrails and evaluation | 003 | **001–005 = MVP** |
| 006 Officer console | 003, 004 | Server-rendered web UI |
| 007 Cloud deployment and observability | 005, 006 | Public demo with a cost cap |

## 4. Collaboration rules (two humans, two agents)

- One feature has one owner at a time and one branch: `feat/<NNN>-<short-name>`. Pull `main` before starting. `main` is protected and changes only through pull requests.
- Never edit `spec.md`, `plan.md`, or `tasks.md` of a feature owned by the other person. Open an issue or ask them.
- Before opening a PR:
  - converge reports Converged;
  - `ruff check`, `ruff format --check`, and `pytest` pass, and you report their real output.
- The PR description has: What, Why, How it was tested, Spec Kit status (analyze and converge results), and Notes for the reviewer.
- **Cross-review**: a PR built with Codex is reviewed with Claude Code, and the other way round. Only a human merges.
- Progress lives in `tasks.md` checkboxes and the PR description. Keep both accurate so the other person's agent can continue.

## 5. Looking up documentation: Context7 MCP (mandatory)

Never write library or SDK code from memory. Before using any library API:
1. Call Context7 `resolve-library-id`.
2. Call Context7 `query-docs` with one focused question per concept, and pass the version from `pyproject.toml` / `uv.lock` when there is one.

Known IDs:

| Library | Context7 ID |
|---|---|
| Anthropic Python SDK | `/anthropics/anthropic-sdk-python` |
| MCP Python SDK | `/modelcontextprotocol/python-sdk` |
| GitHub Spec Kit | `/github/spec-kit` |

Resolve the IDs for FastAPI, SQLAlchemy, Alembic, Pydantic, and HTMX when you need them. If Context7 is unavailable, use the official docs site and say so in your reply. Never invent an API.

Never put API keys, secrets, personal data, or private code into a Context7 query.

## 6. Claude API rules (current as of November 2026, verify with Context7)

- Model from env `LLM_MODEL`, default `claude-opus-5-5`. Use the exact ID with no date suffix. Changing the model is a human decision.
- Set `output_config: {"effort": ...}` explicitly from env `LLM_EFFORT` (default `medium`). Use adaptive thinking (`thinking: {"type": "adaptive"}`) or omit `thinking`. Do **not** send `budget_tokens`, `thinking: {"type": "disabled"}`, `temperature`, `top_p`, or `top_k` (400 errors).
- Forced `tool_choice` (`any` or `tool`) returns a 400 on this model. Use `tool_choice: auto` with a clear instruction and `strict: true` on tools (schemas need `required` and `additionalProperties: false`).
- No assistant prefill (400). Use strict tools or `output_config.format` / `client.messages.parse()` for structured data. The `output_format` parameter is deprecated.
- Write our own loop on the Messages API (`while stop_reason == "tool_use"`):
  - Return all `tool_result` blocks of one assistant turn in a single user message.
  - Use `is_error: true` for a failed tool instead of dropping it.
  - Parse tool inputs as JSON and validate them with Pydantic before running them.
- Check `stop_reason` before reading content, and handle `refusal` (read `stop_details`) and `max_tokens`. Enable server-side refusal fallbacks (beta `server-side-fallback-2026-07-01`, `fallbacks: "default"`) after checking the exact syntax.
- Non-streaming `max_tokens` is around 16000. Cache the stable prefix (tools + system prompt) and check `usage.cache_read_input_tokens`.
- Catch the SDK's typed exceptions from most to least specific. Never match on error strings. Retry only retryable errors.
- MCP: the agent connects to `mcp_bureau` with the MCP Python SDK client (stdio locally, streamable HTTP when deployed), and an adapter converts MCP tools into Claude tool definitions. Claude's hosted MCP connector is optional, not the main path.
- Live LLM calls cost money. Unit tests use a scripted fake LLM client and need no API key. Live evals run only on purpose (`pytest -m live` / `make eval`).

## 7. Tech stack and conventions (plans must follow this)

- Python 3.12, `uv`, `ruff` (lint + format), type hints, `pytest`.
- FastAPI, Pydantic v2, SQLAlchemy 2 + Alembic, PostgreSQL. Use `Decimal` for money, never `float`.
- Anthropic Python SDK, MCP Python SDK. Officer console: Jinja2 + HTMX inside the FastAPI app, with no frontend build step.
- Docker; Google Cloud Run for deployment.
- Code, comments, commits (Conventional Commits), specs, and README are in English. Agent messages to applicants are in Indonesian.

Repository layout (ask before changing):

```
app/        api/ agent/ domain/ db/ web/
mcp_bureau/ MCP credit bureau server + synthetic dataset
config/     policy.yaml, settings
tests/      unit and API tests (no real LLM calls)
evals/      scenario cases, eval runner, report.md
deploy/     Cloud Run config and scripts
specs/      Spec Kit feature folders
docs/       design-notes.md, architecture diagram
notes/      gitignored personal study notes (Indonesian)
```

## 8. What you (the agent) deliver for every feature

1. The Spec Kit artifacts for the current step, each reviewed by a human before the next step.
2. Code with tests and real `ruff` / `pytest` output. Never claim tests pass without running them.
3. `docs/design-notes.md`: one entry per real design decision (the decision, the options considered, and why).
4. `notes/<NNN>-penjelasan.md` in **Indonesian** (gitignored), so the humans can explain the work in an interview:
   - what was built and how the parts connect;
   - the key files and functions;
   - trade-offs and what would change in production;
   - 5 likely interview questions with short answers;
   - one small change the humans should make by hand to check they understand it.
5. A PR with the template in section 4. Never merge it yourself.
