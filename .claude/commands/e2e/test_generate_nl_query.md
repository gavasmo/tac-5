# E2E Test: Generate Natural Language Query Button

Test the "Generate Query" button in the Natural Language SQL Interface application.

## User Story

As a user of the Natural Language SQL Interface
I want to click a button that generates an example natural language query based on my currently loaded tables
So that I can quickly explore my data without having to compose a query from scratch

## Test Steps

1. Navigate to the `Application URL`
2. Take a screenshot of the initial state
3. **Verify** core UI elements are present:
   - Query input textbox
   - Query button
   - Upload Data button
   - Generate Query button
4. **Verify** the "Generate Query" button is visually separated (justified to the opposite side of the row) from the Query and Upload Data button group

5. Click the "Upload Data" button to open the upload modal
6. Click the "Users Data" sample data button to load a real table/schema
7. **Verify** the "users" table appears in the Available Tables section

8. Click into the query input field and type placeholder text: "placeholder text that should be overwritten"
9. Take a screenshot of the query input with the placeholder text
10. Click the "Generate Query" button
11. **Verify** the query input field no longer contains "placeholder text that should be overwritten"
12. **Verify** the query input field contains new, non-empty generated text
13. Take a screenshot of the query input after generation

## Success Criteria
- Generate Query button exists and is styled like the Upload Data secondary button
- Generate Query button is visually separated (opposite side) from the Query/Upload Data button group
- Clicking Generate Query always overwrites existing query input content with newly generated, non-empty text
- Existing Query and Upload Data functionality is unaffected
- 2 screenshots are taken
