# Feature: Generate Natural Language Query Button

## Feature Description
Add a new "Generate Query" button to the Natural Language SQL Interface that uses the LLM (via `core/llm_processor.py`) to inspect the current database schema (tables, columns, types, row counts) and generate an interesting, ready-to-run natural language question about the data. The generated question is written into the query input textarea, always overwriting any existing content, so the user can review it and press "Query" (or edit it first) to execute it. The button is visually styled like the existing "Upload Data" button (`secondary-button` class) but is placed in its own control row, separate from the primary `Query`/`Upload Data` button group, so it reads as a distinct, secondary action ("random query" / inspiration generator) rather than a core action.

## User Story
As a user exploring a new dataset
I want to click a button that generates a relevant natural language question about my uploaded tables
So that I can discover interesting queries to run without having to think of one myself or already know the schema

## Problem Statement
Today the only way to query data is to manually type a natural language question. New users who have just uploaded data (or are exploring a schema they didn't create) don't know what columns/tables exist or what an "interesting" question would look like. There's no discovery mechanism, and no way to get inspiration for a query directly in the UI.

## Solution Statement
Introduce a new backend endpoint (`POST /api/generate-query`) that fetches the current database schema (reusing `get_database_schema()`), formats it for a prompt, and asks the configured LLM provider (OpenAI or Anthropic, following the same routing/priority logic already used by `generate_sql`) to produce a single, interesting natural-language question (max two sentences) about the existing tables and their structure. A new `generate_random_query()` function is added to `core/llm_processor.py` alongside sibling `generate_random_query_with_openai` / `generate_random_query_with_anthropic` helpers that mirror the existing `generate_sql_with_openai` / `generate_sql_with_anthropic` implementations (same client setup, same markdown-fence cleanup, same env var checks). On the frontend, a new "✨ Generate Query" button (styled with the existing `.secondary-button` class, matching "Upload Data") is added in its own control row below the primary query controls. Clicking it calls the new endpoint and overwrites the `#query-input` textarea's value with the returned query text (regardless of what was previously typed there), so the user can immediately hit "Query" or tweak it further.

## Relevant Files
Use these files to implement the feature:

- `app/server/core/llm_processor.py` - Contains `generate_sql_with_openai`, `generate_sql_with_anthropic`, `format_schema_for_prompt`, and `generate_sql` routing logic. This is where the new `generate_random_query_with_openai`, `generate_random_query_with_anthropic`, and `generate_random_query` functions will be added, following the exact same patterns (client setup, prompt construction, markdown cleanup, provider routing/priority).
- `app/server/core/sql_processor.py` - Contains `get_database_schema()` which returns the `{tables: {table_name: {columns, row_count}}}` structure needed to build the prompt. Reused as-is, no changes needed.
- `app/server/core/data_models.py` - Contains all Pydantic request/response models (`QueryRequest`, `QueryResponse`, etc.). A new `GenerateQueryResponse` model needs to be added here (`query: str`, `error: Optional[str] = None`).
- `app/server/server.py` - FastAPI app with all endpoints (`/api/upload`, `/api/query`, `/api/schema`, `/api/insights`, `/api/health`). A new `POST /api/generate-query` endpoint needs to be added here, following the exact logging/error-handling pattern used by `process_natural_language_query` and `generate_insights_endpoint`.
- `app/server/tests/core/test_llm_processor.py` - Existing unit tests for `llm_processor.py` using `pytest` + `unittest.mock.patch` on `OpenAI`/`Anthropic` classes and `os.environ`. New tests for the random query generation functions should follow this exact mocking pattern.
- `app/client/index.html` - Contains the `#query-section` with `.query-controls` (Query + Upload Data buttons). A new sibling control row (e.g. `.query-generate-controls`) needs to be added inside `#query-section`, below `.query-controls`, containing the new `#generate-query-button` styled with `class="secondary-button"` (matching Upload Data's style) to keep it visually separate from the primary buttons.
- `app/client/src/main.ts` - Contains `initializeQueryInput()`, `initializeFileUpload()`, `initializeModal()`, `displayError()`, etc., all wired up in the `DOMContentLoaded` listener. A new `initializeGenerateQuery()` function needs to be added and called from the `DOMContentLoaded` listener, implementing the click handler that calls `api.generateQuery()` and overwrites `#query-input`'s value.
- `app/client/src/api/client.ts` - Contains the `api` object with `uploadFile`, `processQuery`, `getSchema`, `generateInsights`, `healthCheck` methods, all using the shared `apiRequest<T>()` helper. A new `generateQuery(): Promise<GenerateQueryResponse>` method needs to be added here, calling `POST /generate-query`.
- `app/client/src/types.d.ts` - Contains TypeScript interfaces that must mirror the Pydantic models exactly. A new `GenerateQueryResponse` interface needs to be added here (`query: string`, `error?: string`), matching the new backend model.
- `app/client/src/style.css` - Contains `.query-section`, `.query-controls`, `.primary-button`, `.secondary-button` rules. A new `.query-generate-controls` rule needs to be added (simple flex row with top margin) to visually separate the new button from `.query-controls`.
- `.claude/commands/test_e2e.md` - Read this to understand how E2E tests are run (setup, screenshot directory conventions, output format) before writing the new E2E test file.
- `.claude/commands/e2e/test_basic_query.md` - Read this as the reference example for E2E test file structure (User Story, Test Steps, Success Criteria) to model the new E2E test file after.
- `README.md` - Update the "Usage" and "API Endpoints" sections to document the new button and endpoint.

### New Files
- `.claude/commands/e2e/test_generate_query_button.md` - New E2E test file validating the "Generate Query" button: clicking it populates the query input, overwrites existing text, and the generated query can be executed successfully.

## Implementation Plan
### Phase 1: Foundation
Add the new Pydantic response model (`GenerateQueryResponse`) to `core/data_models.py` and the mirrored TypeScript interface to `types.d.ts`. These are pure data-shape additions with no behavior, and everything else depends on them.

### Phase 2: Core Implementation
Implement the schema-aware random query generation logic in `core/llm_processor.py` (`generate_random_query_with_openai`, `generate_random_query_with_anthropic`, `generate_random_query`), reusing `format_schema_for_prompt` and the same provider-routing priority as `generate_sql`. Add the new `POST /api/generate-query` FastAPI endpoint in `server.py` that calls `get_database_schema()` then `generate_random_query()`, following the existing try/except/logging conventions. Add unit tests for the new llm_processor functions mirroring the existing test patterns in `test_llm_processor.py`.

### Phase 3: Integration
Wire up the frontend: add the `generateQuery()` method to `api/client.ts`, add the new button markup to `index.html` in its own control row (separate from `.query-controls`), style it with `.secondary-button` (matching Upload Data) plus a small `.query-generate-controls` layout rule in `style.css`, and implement `initializeGenerateQuery()` in `main.ts` to call the endpoint and overwrite `#query-input`'s value on click (always replacing existing text, disabling/re-enabling the button and showing a loading state, and surfacing errors via `displayError()` — including the "no tables" case). Update `README.md` usage/API docs. Create and run the new E2E test to validate the full user flow.

## Step by Step Tasks
IMPORTANT: Execute every step in order, top to bottom.

### 1. Add backend response model
- In `app/server/core/data_models.py`, add a `GenerateQueryResponse` model near the other Query Models: `query: str`, `error: Optional[str] = None`.

### 2. Add frontend TypeScript interface
- In `app/client/src/types.d.ts`, add a `GenerateQueryResponse` interface mirroring the Pydantic model exactly: `query: string`, `error?: string`.

### 3. Implement random query generation in llm_processor.py
- In `app/server/core/llm_processor.py`, add `generate_random_query_with_openai(schema_info: Dict[str, Any]) -> str` mirroring `generate_sql_with_openai`'s client setup and markdown-fence cleanup, but with a prompt that asks the model to: describe the schema via `format_schema_for_prompt`, then generate ONE interesting, specific natural language question a user could ask about this data, using table/column names that actually exist, limited to a maximum of two sentences, returning ONLY the question text (no explanations, no quotes, no markdown).
- Add `generate_random_query_with_anthropic(schema_info: Dict[str, Any]) -> str` as the Anthropic-provider mirror of the above (same prompt, same cleanup).
- Add a small shared helper (e.g. `clean_generated_query_text(text: str) -> str`) to strip markdown code fences and surrounding quote characters from the generated text, reused by both provider functions.
- Add `generate_random_query(schema_info: Dict[str, Any]) -> str` that: raises a clear exception if `schema_info.get('tables')` is empty (no tables to query), otherwise routes to the OpenAI or Anthropic implementation using the same API-key-priority logic as `generate_sql` (check `OPENAI_API_KEY` first, then `ANTHROPIC_API_KEY`).

### 4. Add the `/api/generate-query` endpoint
- In `app/server/server.py`, import `generate_random_query` from `core.llm_processor` and `GenerateQueryResponse` from `core.data_models`.
- Add `POST /api/generate-query` (`generate_query_endpoint`) that calls `get_database_schema()`, then `generate_random_query(schema_info)`, returns `GenerateQueryResponse(query=query)` on success, and on exception returns `GenerateQueryResponse(query="", error=str(e))`, following the exact logging conventions (`logger.info("[SUCCESS] ...")` / `logger.error("[ERROR] ...")` + traceback) used by the other endpoints.

### 5. Add backend unit tests
- In `app/server/tests/core/test_llm_processor.py`, add a `TestGenerateRandomQuery`-style set of tests (mirroring the existing `TestLLMProcessor` mocking patterns with `@patch('core.llm_processor.OpenAI')` / `@patch('core.llm_processor.Anthropic')` and `patch.dict(os.environ, ...)`) covering: successful generation with OpenAI, successful generation with Anthropic, markdown-fence cleanup, provider routing priority (OpenAI over Anthropic), and the "no tables" error case.
- Run `cd app/server && uv run pytest tests/core/test_llm_processor.py -v` and fix any failures.

### 6. Add frontend API client method
- In `app/client/src/api/client.ts`, add `generateQuery(): Promise<GenerateQueryResponse>` to the `api` object, calling `apiRequest<GenerateQueryResponse>('/generate-query', { method: 'POST' })`.

### 7. Add the button markup and styling
- In `app/client/index.html`, inside `#query-section`, add a new sibling element after `.query-controls` (e.g. `<div class="query-generate-controls"><button id="generate-query-button" class="secondary-button">✨ Generate Query</button></div>`) so it is visually separated from the primary `Query`/`Upload Data` row.
- In `app/client/src/style.css`, add a `.query-generate-controls` rule near `.query-controls` (flex row, `margin-top` to create visual separation) so the new button appears just below, but clearly apart from, the primary button group. Do not modify `.secondary-button` itself, since it must keep the same look as "Upload Data".

### 8. Wire up the click handler in main.ts
- In `app/client/src/main.ts`, add `initializeGenerateQuery()` that: gets `#generate-query-button` and `#query-input`, adds a click listener that disables the button and shows a loading state (mirroring `initializeQueryInput()`'s pattern), calls `api.generateQuery()`, and on success always overwrites `queryInput.value` with the returned `query` (regardless of current contents) and focuses the input; on error (or when `response.error` is set, e.g. no tables available) calls `displayError(...)`; finally re-enables the button and restores its original label.
- Call `initializeGenerateQuery()` from the `DOMContentLoaded` listener alongside the other `initialize*()` calls.

### 9. Create the E2E test file
- Read `.claude/commands/test_e2e.md` and `.claude/commands/e2e/test_basic_query.md` to understand the required structure and conventions.
- Create `.claude/commands/e2e/test_generate_query_button.md` with a User Story, Test Steps, and Success Criteria that: loads sample data (e.g. Users), types placeholder text into the query input, clicks "Generate Query", verifies the button shows a loading state then the query input's value changes to a non-empty generated question (overwriting the placeholder text typed earlier), takes a screenshot, then clicks "Query" to confirm the generated query executes successfully and results/SQL are displayed, taking a final screenshot.

### 10. Update documentation
- Update `README.md`'s "Usage" section to mention the new "Generate Query" button and its behavior (overwrites the input field).
- Update `README.md`'s "API Endpoints" list to include `POST /api/generate-query`.

### 11. Run full validation
- Execute every command in the `Validation Commands` section below and fix any failures before considering the feature complete.

## Testing Strategy
### Unit Tests
- `generate_random_query_with_openai` returns cleaned text and calls the OpenAI client with the expected model/temperature/max_tokens, using a mocked schema.
- `generate_random_query_with_anthropic` returns cleaned text and calls the Anthropic client correctly, using a mocked schema.
- `generate_random_query` raises when `schema_info['tables']` is empty.
- `generate_random_query` routes to OpenAI when `OPENAI_API_KEY` is set, and to Anthropic when only `ANTHROPIC_API_KEY` is set (mirrors existing `generate_sql` routing tests).
- Markdown code-fence and quote stripping behaves the same as the existing `generate_sql_with_*` cleanup.

### Edge Cases
- No tables exist in the database (empty schema) — endpoint should return a clear `error` message instead of crashing, and the frontend should surface it via `displayError()` without touching the query input.
- Neither `OPENAI_API_KEY` nor `ANTHROPIC_API_KEY` is set — should raise the same `ValueError` behavior as `generate_sql_with_*` today.
- LLM returns a response longer than two sentences or wrapped in markdown/quotes — cleanup logic should still leave usable plain text (note: enforcing the "two sentences max" constraint is done via prompt instruction, not strict post-processing truncation, matching how `generate_sql_with_*` relies on prompt instructions rather than hard validation).
- User has existing text in the query input when clicking "Generate Query" — must be fully overwritten, never appended.
- Rapid repeated clicks on "Generate Query" — button is disabled while the request is in flight, mirroring the existing Query button behavior.

## Acceptance Criteria
- A new "✨ Generate Query" button exists, styled identically to "Upload Data" (`secondary-button`), and is placed in a visually distinct row separate from the primary `Query`/`Upload Data` controls.
- Clicking the button calls `POST /api/generate-query`, which uses `core/llm_processor.py` and the current database schema to generate an interesting natural language question limited to two sentences.
- The generated query always overwrites the current contents of the query input field, regardless of what was there before.
- If no tables exist, the button surfaces a clear error instead of crashing or silently doing nothing.
- All existing functionality (Query, Upload Data, schema display, insights) continues to work with zero regressions.
- New unit tests for the generation logic pass, and a new E2E test validates the button's behavior end-to-end with screenshots.

## Validation Commands
Execute every command to validate the feature works correctly with zero regressions.

- `cd app/server && uv run pytest` - Run server tests to validate the feature works with zero regressions
- `cd app/server && uv run pytest tests/core/test_llm_processor.py -v` - Run the new/updated llm_processor tests specifically
- `cd app/client && bun tsc --noEmit` - Run frontend type checking to validate the feature works with zero regressions
- `cd app/client && bun run build` - Run frontend build to validate the feature works with zero regressions
- Read `.claude/commands/test_e2e.md`, then read and execute the new `.claude/commands/e2e/test_generate_query_button.md` E2E test file to validate this functionality works end-to-end with screenshots.

## Notes
- No new third-party libraries are required; the feature reuses the existing `openai` and `anthropic` SDK clients already used by `generate_sql_with_openai`/`generate_sql_with_anthropic`.
- The "limit the query to two sentences maximum" requirement is enforced via the prompt instructions given to the LLM (consistent with how existing SQL-generation prompts rely on instruction-following rather than backend truncation); if in practice the model occasionally exceeds this, a future follow-up could add a lightweight sentence-count safeguard, but this is out of scope for the initial implementation.
- "Justify apart from our current primary buttons" (from the issue) is interpreted as: the new button must be visually/structurally separated from the primary `Query`/`Upload Data` button row, which is why it gets its own `.query-generate-controls` container rather than being appended into `.query-controls`.
