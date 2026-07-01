# Feature: Generate Natural Language Query Button

## Feature Description
Add a "Generate Query" button to the query input section of the Natural Language SQL Interface. When clicked, it calls the backend to generate an interesting, schema-grounded natural language question (using the LLM provider already configured for the app) and overwrites the contents of the query textarea with that generated question. This lets users discover interesting things to ask about their data without having to think of a question themselves, especially useful right after uploading a new dataset.

## User Story
As a user of the Natural Language SQL Interface
I want to click a button that generates an example natural language query based on the tables currently loaded in the database
So that I can quickly explore my data without having to compose a query from scratch

## Problem Statement
Once a user uploads data, they are presented with an empty query textarea and must come up with their own natural language question from scratch. Many users don't know what questions are interesting or possible to ask against the specific schema they just uploaded, which creates friction and reduces engagement with the app's core natural-language-to-SQL feature.

## Solution Statement
Add a new "Generate Query" button, styled like the existing "Upload Data" secondary button, placed visually apart (justified to the opposite side of the row) from the primary Query/Upload Data button group. Clicking it calls a new `POST /api/generate-query` backend endpoint that reads the current database schema (via the existing `get_database_schema` schema-introspection function) and asks the already-configured LLM provider (OpenAI or Anthropic, reusing the same provider-selection convention as `generate_sql`) to produce one interesting, schema-grounded natural language question, capped at two sentences. The response always overwrites whatever is currently in the query input field, matching the explicit requirement in the issue ("Always overwrite what's in the field").

## Relevant Files
Use these files to implement the feature:

- `README.md` - Project overview; documents existing API endpoints list (needs to be updated with the new endpoint) and describes the Usage flow the new button fits into.
- `app/server/server.py` - FastAPI app; defines all `/api/*` routes (`process_natural_language_query`, `get_database_schema_endpoint`, etc.). The new `POST /api/generate-query` endpoint must be added here, following the same try/except + logging pattern as `process_natural_language_query`.
- `app/server/core/llm_processor.py` - Contains `generate_sql_with_openai`, `generate_sql_with_anthropic`, `generate_sql` (provider-routing dispatcher), and `format_schema_for_prompt`. The new query-generation logic (`generate_nl_query_with_openai`, `generate_nl_query_with_anthropic`, `generate_natural_language_query`) belongs here, reusing `format_schema_for_prompt` and following the exact same OpenAI/Anthropic client-call conventions as the existing `generate_sql_with_*` functions.
- `app/server/core/data_models.py` - Pydantic request/response models (`QueryRequest`, `QueryResponse`, `DatabaseSchemaResponse`, etc.). Add a new `GenerateQueryResponse` model here (`query: str`, `error: Optional[str] = None`), matching the shape of `QueryResponse`.
- `app/server/core/sql_processor.py` - Already exposes `get_database_schema()`, which is used elsewhere in `server.py` (`get_database_schema_endpoint`) to introspect table/column structure. Reuse this directly to source the schema used for query generation — no new schema-introspection code needed.
- `app/server/tests/core/test_llm_processor.py` - Existing pytest suite for `llm_processor.py` (uses `unittest.mock.patch` on the `OpenAI`/`Anthropic` classes). Add a new `TestNaturalLanguageQueryGeneration` test class here covering success, markdown/quote stripping, two-sentence truncation, missing API key, and API error cases for both providers, plus the provider-routing dispatcher.
- `app/client/index.html` - Contains the `#query-section` markup with `#query-input`, `.query-controls` div, `#query-button`, and `#upload-data-button`. Add a new `#generate-query-button` here, grouping the existing Query/Upload Data buttons into a `.query-controls-group` div and placing the new button as a sibling so CSS can justify it to the opposite side of the row.
- `app/client/src/style.css` - Defines `.query-controls`, `.primary-button`, `.secondary-button` styles. Update `.query-controls` to `justify-content: space-between` and add a `.query-controls-group` rule so the new button visually separates from the existing pair while still using the existing `.secondary-button` styling (same as Upload Data).
- `app/client/src/main.ts` - Client entry point; contains `initializeQueryInput()`, `initializeFileUpload()`, `initializeModal()`, and the `DOMContentLoaded` bootstrap that calls them. Add `initializeGenerateQueryButton()` following the same pattern as the other `initialize*` functions (grab elements by id, attach a click listener, disable/show loading state during the async call, restore state in `finally`), and call it from the bootstrap.
- `app/client/src/api/client.ts` - Defines the `api` object with methods like `getSchema()`, `processQuery()` that wrap `apiRequest<T>()`. Add a `generateQuery(): Promise<GenerateQueryResponse>` method calling `POST /generate-query`.
- `app/client/src/types.d.ts` - Client-side TypeScript interfaces mirroring the server's Pydantic models (`QueryResponse`, `DatabaseSchemaResponse`, etc.). Add a `GenerateQueryResponse` interface (`query: string; error?: string;`) matching the new server-side model.
- `.claude/commands/test_e2e.md` - Read this to understand how the E2E test runner executes an E2E test file (Playwright automation, screenshot conventions, success/failure criteria format).
- `.claude/commands/e2e/test_basic_query.md` - Read this as the reference example for how an E2E test file should be structured (User Story, numbered Test Steps, **Verify** assertions, Success Criteria).

### New Files
- `.claude/commands/e2e/test_generate_nl_query.md` - New E2E test file validating the Generate Query button: button presence and visual separation from the primary button group, clicking it overwrites existing textarea content with new non-empty generated text, and existing Query/Upload Data functionality is unaffected. Follow the structure of `test_basic_query.md`.

## Implementation Plan
### Phase 1: Foundation
Add the server-side data model (`GenerateQueryResponse`) and the LLM query-generation functions in `llm_processor.py` (mirroring the existing `generate_sql_with_openai` / `generate_sql_with_anthropic` / `generate_sql` provider-routing pattern), including a defensive two-sentence truncation helper since the LLM prompt instruction alone cannot be trusted to always produce exactly two sentences.

### Phase 2: Core Implementation
Wire a new `POST /api/generate-query` FastAPI endpoint in `server.py` that fetches the current schema via `get_database_schema()`, calls `generate_natural_language_query(schema_info)`, and returns a `GenerateQueryResponse`, following the existing error-handling/logging conventions used by `process_natural_language_query`. Add corresponding unit tests in `test_llm_processor.py`.

### Phase 3: Integration
Add the "Generate Query" button to the client: markup in `index.html`, styling in `style.css` (secondary-button style, justified apart from the Query/Upload Data group), the `generateQuery()` API client method, the `GenerateQueryResponse` TypeScript type, and the `initializeGenerateQueryButton()` wiring in `main.ts` that always overwrites the query input value on success and shows an error via the existing `displayError` helper on failure. Finish with an E2E test validating the full flow and update the README API endpoint list.

## Step by Step Tasks
IMPORTANT: Execute every step in order, top to bottom.

### 1. Add `GenerateQueryResponse` data model
- In `app/server/core/data_models.py`, add a `GenerateQueryResponse(BaseModel)` class with `query: str` and `error: Optional[str] = None`, placed near `QueryResponse` for locality.

### 2. Implement query-generation logic in `llm_processor.py`
- Add a private `_limit_to_two_sentences(text: str) -> str` helper that splits on sentence-ending punctuation and joins the first two sentences, as a defensive backstop to the "two sentences maximum" requirement.
- Add `generate_nl_query_with_openai(schema_info: Dict[str, Any]) -> str`: reads `OPENAI_API_KEY`, builds an OpenAI client, formats the schema with `format_schema_for_prompt`, prompts the model to produce exactly ONE interesting, schema-grounded natural language question (no SQL, no preamble, max two sentences), strips markdown/quote wrapping from the response, and passes it through `_limit_to_two_sentences`.
- Add `generate_nl_query_with_anthropic(schema_info: Dict[str, Any]) -> str`: same behavior using the Anthropic client/model, matching the conventions of `generate_sql_with_anthropic`.
- Add `generate_natural_language_query(schema_info: Dict[str, Any]) -> str`: routes to `generate_nl_query_with_openai` if `OPENAI_API_KEY` is set, else `generate_nl_query_with_anthropic` if `ANTHROPIC_API_KEY` is set, else falls back to the OpenAI path so a clear "API key not set" error surfaces (mirroring `generate_sql`'s provider-selection convention).

### 3. Add the `/api/generate-query` endpoint
- In `app/server/server.py`, import `GenerateQueryResponse` from `core.data_models` and `generate_natural_language_query` from `core.llm_processor`.
- Add `POST /api/generate-query` returning `GenerateQueryResponse`: call `get_database_schema()` to obtain `schema_info`, call `generate_natural_language_query(schema_info)`, log success, and return `GenerateQueryResponse(query=query)`. On exception, log the error and traceback and return `GenerateQueryResponse(query="", error=str(e))`, matching the pattern used in `process_natural_language_query`.

### 4. Add server-side unit tests
- In `app/server/tests/core/test_llm_processor.py`, import the three new functions and add a `TestNaturalLanguageQueryGeneration` class covering:
  - OpenAI success path returns the model's question.
  - OpenAI response with markdown fences/quotes gets stripped correctly.
  - A three-sentence OpenAI response gets truncated to two sentences.
  - Missing `OPENAI_API_KEY` raises an exception with a clear message.
  - OpenAI API errors are wrapped in a clear exception message.
  - Anthropic success path returns the model's question (mirroring the OpenAI cases).
  - `generate_natural_language_query` provider routing: OpenAI key present → OpenAI path called; only Anthropic key present → Anthropic path called; neither key present → OpenAI path invoked (surfacing the missing-key error).
- Run `cd app/server && uv run pytest tests/core/test_llm_processor.py -v` to confirm all new and existing tests pass.

### 5. Add the `GenerateQueryResponse` TypeScript type
- In `app/client/src/types.d.ts`, add `interface GenerateQueryResponse { query: string; error?: string; }` near `QueryResponse`.

### 6. Add the `generateQuery` API client method
- In `app/client/src/api/client.ts`, add `async generateQuery(): Promise<GenerateQueryResponse> { return apiRequest<GenerateQueryResponse>('/generate-query', { method: 'POST' }); }` alongside the existing `getSchema`/`processQuery` methods.

### 7. Add the "Generate Query" button markup
- In `app/client/index.html`, inside `.query-controls`, wrap the existing `#query-button` and `#upload-data-button` in a new `<div class="query-controls-group">`, and add `<button id="generate-query-button" class="secondary-button">Generate Query</button>` as a sibling of that group so it lands on the opposite side of the row.

### 8. Style the new button layout
- In `app/client/src/style.css`, update `.query-controls` to include `justify-content: space-between` so the group and the new button sit apart, and add a `.query-controls-group { display: flex; gap: 1rem; align-items: center; }` rule to keep the Query/Upload Data buttons visually grouped together. Do not introduce a new button class — `#generate-query-button` reuses `.secondary-button` so it matches the Upload Data button style exactly, per the issue's requirement.

### 9. Wire up the button behavior in `main.ts`
- Add `initializeGenerateQueryButton()`: on click, disable the button and show a loading indicator (reusing the existing `.loading` spinner convention used elsewhere), call `api.generateQuery()`, and on success always overwrite `queryInput.value` with `response.query` regardless of prior content (per the issue's explicit "Always overwrite" requirement); on `response.error` or a thrown error, call the existing `displayError` helper; in `finally`, re-enable the button and restore its label.
- Call `initializeGenerateQueryButton()` from the `DOMContentLoaded` bootstrap alongside `initializeQueryInput()`, `initializeFileUpload()`, and `initializeModal()`.

### 10. Create the E2E test file
- Create `.claude/commands/e2e/test_generate_nl_query.md` following the structure of `.claude/commands/e2e/test_basic_query.md`: include a User Story, numbered Test Steps that (a) verify the Generate Query button exists and is visually separated from the Query/Upload Data group, (b) load sample data via Upload Data, (c) type placeholder text into the query input, (d) click Generate Query, (e) verify the placeholder text has been overwritten with new non-empty text, and screenshots before/after generation. Include a Success Criteria section.

### 11. Update README documentation
- In `README.md`, add `POST /api/generate-query` to the "API Endpoints" list, and optionally mention the "Generate Query" button in the "Usage" section as a way to get a starting-point question.

### 12. Run full validation
- Execute every command listed in `Validation Commands` below, including the new E2E test, and confirm zero regressions across the full test suite.

## Testing Strategy
### Unit Tests
- `llm_processor.py`: OpenAI success, Anthropic success, markdown/quote stripping, two-sentence truncation (defensive helper), missing-API-key errors, upstream API errors, and provider-routing precedence (OpenAI > Anthropic > default-to-OpenAI-error), all following the existing `unittest.mock.patch('core.llm_processor.OpenAI'/'Anthropic')` conventions already used in `test_llm_processor.py`.
- `server.py`: covered indirectly via the E2E test (no existing dedicated server-route test file to extend for this endpoint pattern); the endpoint's own logic is a thin wrapper around already-tested `get_database_schema` and `generate_natural_language_query`.

### Edge Cases
- No tables uploaded yet (empty schema) — `generate_natural_language_query` should still return a valid response or a clear error rather than crashing; verify `format_schema_for_prompt` handles an empty `tables` dict gracefully (it already does, per existing behavior used by `generate_sql`).
- Neither `OPENAI_API_KEY` nor `ANTHROPIC_API_KEY` set — endpoint returns `GenerateQueryResponse(query="", error=...)` with a clear message instead of a 500.
- LLM returns more than two sentences — defensive truncation ensures the client never receives more than two.
- LLM wraps the response in markdown code fences or quotes — stripped before returning.
- Clicking Generate Query when the query input already has user-typed text — must always be overwritten, never appended or merged.
- Rapid repeated clicks — button is disabled while the request is in flight to prevent duplicate concurrent requests.

## Acceptance Criteria
- A "Generate Query" button is present in the query controls area, styled identically to the "Upload Data" secondary button.
- The button is visually justified apart from the Query/Upload Data button group (opposite side of the row).
- Clicking the button always overwrites the current contents of the query input field with a newly generated natural language question, regardless of prior content.
- The generated question is grounded in the actual tables/columns present in the current database schema.
- The generated question is limited to a maximum of two sentences.
- Existing Query and Upload Data functionality is unaffected.
- All new and existing server-side unit tests pass.
- The new E2E test passes with screenshots demonstrating the before/after state of the query input.
- No regressions in existing client build/typecheck or server test suite.

## Validation Commands
Execute every command to validate the feature works correctly with zero regressions.

- Read `.claude/commands/test_e2e.md`, then read and execute the new E2E test file `.claude/commands/e2e/test_generate_nl_query.md` to validate this functionality works end-to-end with screenshots.
- `cd app/server && uv run pytest` - Run server tests to validate the feature works with zero regressions
- `cd app/client && bun tsc --noEmit` - Run frontend type checking to validate the feature works with zero regressions
- `cd app/client && bun run build` - Run frontend build to validate the feature works with zero regressions

## Notes
- No new third-party libraries are required — the feature reuses the already-installed `openai` and `anthropic` Python SDKs and the existing `apiRequest` client helper.
- Provider selection intentionally mirrors the existing `generate_sql` convention (OpenAI preferred if its key is present, else Anthropic) for consistency, rather than introducing a separate/independent provider-selection strategy for this one endpoint.
- The two-sentence limit is enforced both via prompt instruction and a defensive server-side truncation helper (`_limit_to_two_sentences`), since LLM outputs cannot be trusted to always follow instruction-based constraints exactly.
- Placement/styling ("justify apart", "Use Upload Data button style") is achieved purely via a CSS layout change (`justify-content: space-between` plus a new `.query-controls-group` wrapper) and reusing the existing `.secondary-button` class — no new button variant is introduced.
