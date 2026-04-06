# Regulatory Reference Extractor

A two-stage pipeline that extracts and resolves internal cross-references from regulatory text. Given a corpus of regulatory paragraphs, the system identifies references to sections, annexes, tables, figures, and appendices, then maps them back to paragraph IDs in the corpus.

---

## How It Works

The pipeline processes each paragraph in two stages:

**Stage 1 — Regex extraction**

A set of compiled regex patterns scans the paragraph text and extracts:

- Single references: `paragraph 8.2.1`, `section 4`, `para. 8.1 and 8.2`
- Ranges: `paragraphs 8.1 to 8.5` (expanded to the full list of existing intermediate sections)
- Structured items: `Annex 3`, `Table 1`, `Figure 5`, `Annex 2 - Appendix A`
- Ambiguous items: standalone `Appendix N` (parent annex unknown)

Ranges are expanded using an algorithm that handles cross-prefix cases using `children` dict that maps partial strings to all paragraphs that start with that prefix as e.g. `'2.1.': ['2.1.1.', '2.1.2.', '2.1.3.', '2.1.4.', '2.1.5.', '2.1.6.', '2.1.7.']`.

**Stage 2 — LLM fallback (optional)**

Any references flagged as ambiguous (e.g. a bare `Appendix B` whose parent annex is unclear) are sent to `gpt-4o-mini` with the full paragraph as context. If possible, the model returns a JSON array of resolved forms (e.g. `"Annex 2 - Appendix B"`).

**Resolution**

Resolved reference strings are looked up against indexes built from the corpus. `exact` dict maps a full section string directly to a paragraph ID as e.g. `'1.4.': '659d4ec2cbbf0962d3573854'`

Each reference is classified as either `certain` meaning it is exact match or `candidate` which requires LLM to be resolved. LLM is prompted with extracted candidate reference and list of all keys from `exact` dict.

---

## Setup

**Requirements**

- Python 3.9+
- Jupyter Notebook

**Install dependencies**

```bash
pip install -r requirements.txt
```

**Set your OpenAI API key**

Create `.env` file and add place `OPENAI_API_KEY="sk-..."` into it. 

If the key is not set, the system still runs, ambiguous references are simply returned unresolved.

---

## Running the Notebook

1. `evaluation_data.json` and `test_data.json` must be in the same directory as `main.ipynb`.
2. Launch Jupyter
3. Run all cells

---

## Potential Improvements

### 1. Improve recall with richer regex coverage

There are still references that the current patterns miss so regex patterns can be further extended.

Add support for cross paragraph annex references, e.g. Annex 4 paragraph 2.1 is referenced from Paragraph 4. Split reference into two parts and use first part for scope search, second for paragraph search.

### 2. Smarter resolution

A short prefix like `"4"` or `"8"` can match hundreds of paragraphs, most of which are irrelevant. Current solution therefore only uses one dotted part afterwards to extend range.

When reference cannot be matched with keys from `exact`, all keys are sent to LLM. This includes way too many paragraphs, most of which are irrelevant, making it hard for LLM to determine the correct one. Possible fixes:

- Implement ranking mechanism based on semantic similarity (embeddings)
- Contextual narrowing: use the source paragraph's own section number to prefer siblings and children over distant relatives

### 3. Structured pipeline instead of a notebook

Refactor into a Python package with clearly separated concerns.

### 4. Caching and batching for LLM calls

Each ambiguous reference triggers a separate API call. In a large corpus this is slow and expensive. Improvements:

- Batch multiple paragraphs' ambiguous references into a single API call
- Cache results by `(text_hash, ambiguous_list)` so reprocessing the same document doesn't re-query
- Add retry logic with exponential back-off for rate-limit errors

### 5. Proper test suite

Migrate the current self-tests to e.g. pytest.

### 6. Configuration

- Move the model name (`gpt-4o-mini`), temperature, and safety cap (`10_000`) to a config file or environment variables

### 7. Output schema

Return results as a typed dataclass or Pydantic model rather than raw dicts, so downstream consumers have a stable, documented interface.

### 8. Prompts

Clarify and expand the prompts used for the LLM fallback. Add concrete examples including text, ambiguous reference and final reference to show how ambiguous references should be resolved.