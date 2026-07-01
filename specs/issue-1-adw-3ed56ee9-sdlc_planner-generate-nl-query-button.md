# Feature: Generate Natural Language Query Button

## Feature Description
Add a new "Generate Query" button to the Natural Language SQL Interface that uses an LLM to inspect the current database schema (tables and their columns) and generate an interesting, ready-to-run natural language query. Clicking the button always overwrites the contents of the query input field with the freshly generated query (max two sentences), giving users a fast way to discover what kinds of questions they can ask about their uploaded data without having to think one up themselves.

## User Story
As a user of the Natural Language SQL Interface
I want to click a button that generates an example natural language query based on my currently loaded tables
So that I can quickly explore my data without having to compose a query from scratch

## Problem Statement
Users who have just uploaded data (or are exploring an unfamiliar dataset) don't necessarily know what interesting questions they can ask. There's no guided or inspirational entry point into the query box — the textarea starts empty and users must invent a query themselves, which is a barrier for first-time users or when tables/columns are unfamiliar.

## Solution Statement
Add a "Generate Query" button, styled like the existing "Upload Data" secondary button, placed apart from the primary Query/Upload Data button group (justified to the opposite side of the query controls row). Clicking it calls a new backend endpoint that fetches the current database schema, sends it to the LLM processor (reusing the existing OpenAI/Anthropic routing pattern in `core/llm_processor.py`), and asks the model to produce one interesting, concise (max two sentences) natural language query grounded in the real tables/columns available. The returned query text always overwrites whatever is currently in the query input field.

## Relevant Files
Use these files to implement the feature:

- `app/server/server.py` - FastAPI app with all API route definitions; add the new `POST /api/generate-query` endpoint here following the existing route patterns (e.g. `process_natural_language_query`).
- `app/server/core/llm_processor.py` - Contains `generate_sql_with_openai`, `generate_sql_with_anthropic`, `format_schema_for_prompt`, and the `generate_sql` routing function. Add a parallel `generate_natural_language_query_with_openai`, `generate_natural_language_query_with_anthropic`, and a `generate_natural_language_query(schema_info)` routing function that mirrors the existing OpenAI-first/Anthropic-fallback priority logic, but which returns a natural language question (not SQL) grounded in the schema.
- `app/server/core/sql_processor.py` - Contains `get_database_schema()` which returns the `{'tables': {table_name: {'columns': {...}, 'row_count': ...}}}` structure already consumed by `format_schema_for_prompt`. Reuse this to build the prompt context; no changes needed here.
- `app/server/core/data_models.py` - Pydantic request/response models. Add a `GenerateQueryResponse` model (`query: str`, `error: Optional[str] = None`). No request body is needed (mirrors `DatabaseSchemaRequest`'s no-input pattern).
- `app/server/tests/core/test_llm_processor.py` - Existing unit test conventions (mocking `OpenAI`/`Anthropic` clients via `@patch`, asserting call args) to follow when adding tests for the new NL-query generation functions.
- `app/client/index.html` - Contains the `.query-controls` div with the `#query-button` (primary) and `#upload-data-button` (secondary) buttons. Add the new `#generate-query-button` here, using the same `secondary-button` class as Upload Data, in a layout that visually separates it from the existing button group (justified to the opposite side of the row).
- `app/client/src/style.css` - Contains `.query-controls` (currently `display:flex; gap:1rem;`) and the `.primary-button`/`.secondary-button` styles. Update `.query-controls` layout (e.g. `justify-content: space-between` with a wrapper group for the existing two buttons) so the new button sits apart from the primary buttons.
- `app/client/src/api/client.ts` - Contains the `api` object with methods like `processQuery`, `getSchema`. Add a `generateQuery()` method that calls the new `/generate-query` endpoint.
- `app/client/src/types.d.ts` - TypeScript interfaces mirroring the Pydantic models. Add `GenerateQueryResponse` matching the new backend model.
- `app/client/src/main.ts` - Contains `initializeQueryInput()` and other init functions wired up in the `DOMContentLoaded` handler. Add a new `initializeGenerateQueryButton()` function that calls `api.generateQuery()` on click and overwrites `queryInput.value` with the result, plus a loading state matching the existing query button's loading spinner pattern.
- `.claude/commands/test_e2e.md` - Read this to understand how the E2E test runner executes test files (Playwright automation, screenshot conventions, output format).
- `.claude/commands/e2e/test_basic_query.md` - Read this as the template/example for structuring the new E2E test file (User Story, Test Steps, Success Criteria format).

### New Files
- `.claude/commands/e2e/test_generate_nl_query.md` - New E2E test file validating the Generate Query button: verifies it exists, is visually separated from the primary buttons, generates a non-empty query grounded in the loaded schema, and overwrites existing input field content.

## Implementation Plan
### Phase 1: Foundation
Add the new Pydantic response model (`GenerateQueryResponse`) and the LLM processor functions that generate a natural language query from schema info, following the exact routing/priority pattern already used by `generate_sql` (OpenAI key present → OpenAI; else Anthropic key present → Anthropic; else fall back to request-less default since there's no `llm_provider` preference input for this feature — default to OpenAI if neither key present, matching `generate_sql`'s final fallback shape but without a request object).

### Phase 2: Core Implementation
Wire the new `/api/generate-query` FastAPI endpoint that pulls the current schema via `get_database_schema()` and calls the new `generate_natural_language_query()` function, returning a `GenerateQueryResponse`. Add corresponding unit tests for the LLM processor functions and the endpoint.

### Phase 3: Integration
Update the frontend: add the button to `index.html`, style it and adjust `.query-controls` layout in `style.css`, add the `generateQuery()` API client method and `GenerateQueryResponse` type, and wire up the click handler in `main.ts` that always overwrites the query input field. Create the E2E test file and run it to validate the full flow.

## Step by Step Tasks

### Task 1: Add `GenerateQueryResponse` model
- In `app/server/core/data_models.py`, add a `GenerateQueryResponse` model with `query: str` and `error: Optional[str] = None`, placed near the other Query Models.

### Task 2: Add NL query generation functions to `llm_processor.py`
- Add `generate_nl_query_with_openai(schema_info: Dict[str, Any]) -> str` that builds a prompt from `format_schema_for_prompt(schema_info)` asking the model to produce ONE interesting natural language question a user could ask about this data, grounded in the real table/column names, limited to a maximum of two sentences, with no SQL and no explanations/preamble — return only the query text.
- Add `generate_nl_query_with_anthropic(schema_info: Dict[str, Any]) -> str` mirroring the OpenAI version but calling the Anthropic client (same model/params as `generate_sql_with_anthropic`).
- Add `generate_natural_language_query(schema_info: Dict[str, Any]) -> str` that routes between the two using the same OpenAI-key-first / Anthropic-key-fallback priority as `generate_sql`, defaulting to OpenAI if neither key is present (since there is no `QueryRequest.llm_provider` input for this feature).
- Add a small helper/trim step to enforce the two-sentence limit defensively (e.g. split on sentence-ending punctuation and keep at most the first two, in case the model ignores the instruction).

### Task 3: Add `/api/generate-query` endpoint
- In `app/server/server.py`, add `POST /api/generate-query` (no request body) that calls `get_database_schema()`, then `generate_natural_language_query(schema_info)`, and returns a `GenerateQueryResponse`. Follow the existing try/except + logging pattern used by the other endpoints (log `[SUCCESS]`/`[ERROR]`, return an error-populated response instead of raising on failure).

### Task 4: Backend unit tests
- In `app/server/tests/core/test_llm_processor.py`, add tests for `generate_nl_query_with_openai`, `generate_nl_query_with_anthropic`, and `generate_natural_language_query` routing (mirroring the existing `TestLLMProcessor` mocking patterns: success case, no-API-key error case, provider priority case).
- Add a test in `app/server/tests/` (or extend an existing server-level test file if one exists) for the `/api/generate-query` endpoint's success and error paths if a server-level test file/pattern exists; otherwise cover this via the unit tests above plus the E2E test.

### Task 5: Add frontend types and API client method
- In `app/client/src/types.d.ts`, add `interface GenerateQueryResponse { query: string; error?: string; }`.
- In `app/client/src/api/client.ts`, add `async generateQuery(): Promise<GenerateQueryResponse>` calling `apiRequest<GenerateQueryResponse>('/generate-query', { method: 'POST' })`.

### Task 6: Add the button to the UI
- In `app/client/index.html`, inside `.query-controls`, restructure so the existing `#query-button` and `#upload-data-button` are grouped together, and add a new `<button id="generate-query-button" class="secondary-button">Generate Query</button>` placed in a way that is justified apart (opposite side) from the primary button group.
- In `app/client/src/style.css`, update `.query-controls` to use `justify-content: space-between` (or equivalent) with a wrapper class for the primary button group so the new button visually separates from Query/Upload Data.

### Task 7: Wire up the click handler
- In `app/client/src/main.ts`, add `initializeGenerateQueryButton()` that: on click, disables the button and shows a loading state (matching the `queryButton` loading spinner pattern), calls `api.generateQuery()`, and on success always overwrites `queryInput.value` with `response.query` regardless of current content; on error calls `displayError(...)`; always re-enables the button and restores its label in a `finally` block.
- Call `initializeGenerateQueryButton()` from the `DOMContentLoaded` handler alongside the other `initialize*` calls.

### Task 8: Create the E2E test file
- Create `.claude/commands/e2e/test_generate_nl_query.md` following the structure of `.claude/commands/e2e/test_basic_query.md`, with steps to: navigate to the app, load sample data (e.g. users) so a real table/schema exists, type placeholder text into the query input, click "Generate Query", verify the input field is no longer the placeholder text and now contains non-empty generated text (overwritten), verify the button is visually separated from the Query/Upload Data buttons, and take screenshots of the before/after state of the query input.

### Task 9: Run validation
- Run all `Validation Commands` below (backend tests, frontend type-check/build, and the new E2E test) to confirm the feature works end-to-end with zero regressions.

## Testing Strategy
### Unit Tests
- `generate_nl_query_with_openai` / `generate_nl_query_with_anthropic`: success case returns the model's text, markdown/quote stripping if applicable, error propagation when the API call fails, error when API key missing.
- `generate_natural_language_query` routing: OpenAI key present → uses OpenAI regardless of nothing else being set; only Anthropic key present → uses Anthropic; neither key present → defaults to OpenAI path (and surfaces the "API key not set" error naturally).
- `/api/generate-query` endpoint: returns a `GenerateQueryResponse` with a populated `query` and no `error` on success; returns an `error` field populated (HTTP 200 body, not raised exception) when schema retrieval or LLM call fails, matching the existing endpoint error-handling convention.

### Edge Cases
- No tables uploaded yet (`schema['tables']` is empty) — the generated query prompt should still return gracefully (e.g. a generic query or a clear message) rather than erroring, and the frontend should still overwrite the input field with whatever text is returned.
- LLM returns more than two sentences despite instructions — defensive trimming should cap it at two.
- Query input already has unsaved user-typed text — clicking Generate Query must overwrite it unconditionally (per the issue's explicit "Always overwrite whats in the field" requirement).
- Neither OPENAI_API_KEY nor ANTHROPIC_API_KEY set — endpoint should return a clear `error` in the response body (consistent with how `/api/query` already handles this) rather than a 500.

## Acceptance Criteria
- A "Generate Query" button exists on the main page, styled with the same `secondary-button` class as "Upload Data".
- The button is visually placed apart (justified to the opposite side) from the existing Query/Upload Data button group.
- Clicking the button calls a new backend endpoint that generates a natural language query grounded in the current database schema (real table and column names).
- The generated query is at most two sentences.
- Clicking the button always overwrites the current contents of the query input field, regardless of what was there before.
- Existing Query and Upload Data functionality is unaffected.
- All backend unit tests pass; frontend type-checks and builds cleanly; the new E2E test passes.

## Validation Commands
Execute every command to validate the feature works correctly with zero regressions.

- `cd app/server && uv run pytest` - Run server tests to validate the feature works with zero regressions
- `cd app/client && bun tsc --noEmit` - Run frontend tests to validate the feature works with zero regressions
- `cd app/client && bun run build` - Run frontend build to validate the feature works with zero regressions
- Read `.claude/commands/test_e2e.md`, then read and execute the new `.claude/commands/e2e/test_generate_nl_query.md` E2E test file to validate this functionality works end-to-end (button visible, separated from primary buttons, generates a schema-grounded query, and overwrites the input field).

## Notes
- No new third-party libraries are required — the feature reuses the existing `openai` and `anthropic` SDK clients already used by `generate_sql_with_openai` / `generate_sql_with_anthropic`.
- The issue text references "lm_proncessor.py" (typo) — this refers to the existing `app/server/core/llm_processor.py` module, which is extended rather than replaced.
- Because `generate_sql` takes a `QueryRequest` (which carries `llm_provider`), but this feature has no user-supplied provider preference, the new `generate_natural_language_query` routing function only depends on which API keys are present in the environment (OpenAI first, then Anthropic), consistent with the "key availability" branch of the existing routing logic.
