# Code Generation Prompts

Prompts used with GitHub Copilot (Agent Mode, Model = Claude Opus 4.6) to generate the Python functions for each step of the pipeline. Each prompt was sent independently with the relevant context attached.

---

## Step 1 — Document indexing

```
Write a Python function build_section_index(paragraphs) -> tuple[dict, dict]
where each paragraph dict has "id" and "text" fields, as e.g.
{
    "text": "8.7.6.2.1. Requirements for Group 1",
    "id": "659d4ec2cbbf0962d3573996",
    "targetIds": []
    }

The function should:
1. Parse the section number prefix from the start of each paragraph's text using regex. 
Section numbers look like "8.24.5.1.10." or "1.2." or "Annex 3", "A.", "Figure 2" or "5." at the very start of the text.
2. Build an exact match index: dict mapping section number string -> paragraph id.
3. Build a prefix index: dict mapping each prefix of the section number -> list of paragraph ids. For example "8.24.5.1.10." should appear under "8.24.5.1." only whereas "8.24.5" under "8.24." only.

Return both dicts.

Write clean and professional code.
```

### Prompt used for fixing issue with indexes

```
ParagraphLinks array has different scopes (main body + annexes/appendices), each restarting numbering at "1.", "2.", etc. 
Implement scope-aware indexing into existing code. Infer each paragraph's scope (main body, Annex N, Annex N - Appendix M) from its position in the array, then build per-scope indexes. When resolving, merge the source paragraph's scope with the root scope so same-scope sections take priority.
```

---


## Step 2 — Reference extraction

```
Write a Python function extract_references(text) -> list[str]that extracts internal cross-references from a regulatory paragraph.

First try regex. The function should cover these patterns:
- Single section references: "paragraph 8.2.1.", "section 4.3.", "para. 8.2.", "paragraph 5."
- Multiple references in one phrase: "paragraphs 8.24.5.1.5. and 8.24.5.1.8."
- Range references: "paragraphs 8.1. to 8.5."
- Annex references: "Annex 3"
- Table and figure references: "Table 1", "Figure 2"
- Appendix references: "Annex X - Appendix N", "Appendix N"

If there is a range reference, all references within range should be covered as for example "paragraphs 2.2.4.1.3. to 2.2.4.2.2." should reference ['2.2.4.1.3.', '2.2.4.1.4.', '2.2.4.2.1.', '2.2.4.2.2.']. 
To do this use children dict where references are assigned based on prefix, e.g. 
{
'2.2.4.1.': ['2.2.4.1.1.', '2.2.4.1.2.', '2.2.4.1.3.', '2.2.4.1.4.'], 
'2.2.4.2.': ['2.2.4.2.1.', '2.2.4.2.2.'],
}

Return 2 lists, one list of normalized reference strings (e.g. ["8.24.5.1.5", "8.24.5.1.8", "Annex 3", "Annex X - Appendix N"]) and another list with ambiguous references e.g. when it is not clear to what annex appendix belongs ["Appendix N"].

If there are ambiguous references, use LLM. The LLM prompt should ask the model to extract and normalize ambiguous references based on text and examples provided and return a JSON array of strings only, no markdown, no explanation.

Read the API key from the .env as OPENAI_API_KEY variable. Handle JSON parsing errors gracefully and return an empty list on failure.

Write clean and professional code. 
```

---

## Step 3 — Reference resolution

```
Write a Python function resolve_reference(
    ref,
    exact_index
) -> tuple[list[str], bool]that resolves a single extracted reference string to paragraph ids.

Logic:
1. Try exact match in exact_index first. If found, return ([id], True). The True flag means this is a certain match and no LLM validation is needed.
2. LLM Validation includes OPENAI API call to try to match extracted reference with one of the existing references. Include existing references into prompt which are stored as keys in exact_index.
Make sure only results with high confidence are returned, otherwise return [].
If non empty result is returned find corresponding id from exact_index.
3. If neither matches, return ([], False).

Also write a wrapper function resolve_all_references(
    references,
    exact_index
) -> tuple[list[str], list[str]]that calls resolve_reference for each extracted reference and returns list with ids.

Write clean and professional code.
```

---

## Step 4 — Evaluation and output

```
Write a Python function evaluate_reference_extraction(
    paragraphs,
    scoped_exact,
    scoped_children,
    scope_of_id
) -> dictthat runs the full extraction and resolution on every paragraph that has ground-truth targetIds and computes  metrics.

Logic:
1. Filter paragraphs to only those where "targetIds" is non-empty.
2. For each paragraph:
   - Look up its scope, then get the merged exact and children indexes for that scope.
   - Call extract_references to get refs and ambiguous lists.
   - Call resolve_all_references to get resolved paragraph ids.
   - Compare predicted id set against expected targetIds set, compute TP, FP, FN...
   - Collect per-paragraph detail dicts with metric to determine overall quality of prediction.
3. Compute overall precision, recall, etc. 
4. Return a dict with keys: paragraphs_evaluated, total_tp, total_fp, total_fn, precision, recall, f1, details.

Write clean and professional code.
```

---

## Step 5 - Test with test_data.json

```
Write a Python function run_on_test_data(test_path = "test_data.json") -> dict that loads the test data and runs the existing reference-extraction/resolution pipeline.

Logic:
1. Load JSON from test_path. The file structure: top-level paragraphLinksis a list of paragraph dicts with fields "text", "id", and "targetIds". "targetIds" are in this case empty and need to be determined.
2. Reuse existing functions in the codebase:
   - build_section_index(paragraphs) to produce document indexing dicts scoped_exact, scoped_children, scope_of_id.
   - extract_references(text) to extract references from the each paragraphs text.
   - resolve_all_references(references, exact_index) to resolve extracted references based on document indexing dict from step 1.

3. Output:
   - Return the full results dict and save as JSON dict into test_result = "test_results.json" path.
```
