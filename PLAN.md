# Plan — Issue #153: Faithfulness checker crashes on `text: None`

**Issue:** https://github.com/ascherj/pathreview/issues/153
**Branch:** tree/fix/153-faithfulness

## What needs to change

The bug is in `FaithfulnessChecker.check()`: when a context chunk has its
`text` key present but set to `None` (e.g. `{"text": None}`), the checker raises
a `TypeError` instead of scoring the feedback. The fix makes `check()` treat a
`None` (or missing) chunk text as empty text, so it returns a valid faithfulness
score instead of crashing.

## Root cause

In [`rag/evaluator/faithfulness_checker.py:34-36`](rag/evaluator/faithfulness_checker.py#L34-L36):

```python
context_text = " ".join([
    chunk.get("text", "") for chunk in context_chunks
])
```

`dict.get("text", "")` only substitutes the default `""` when the key is
**missing**. When the key is present with value `None`, `.get()` returns `None`,
and `" ".join([...])` raises:

```
TypeError: sequence item 0: expected str instance, NoneType found
```

### Reproduction

```python
from rag.evaluator.faithfulness_checker import FaithfulnessChecker
FaithfulnessChecker().check('Knows Python.', [{'text': None}])
# TypeError: sequence item 0: expected str instance, NoneType found
```

Failing test: `test_none_context_chunk_text` in
[`tests/unit/test_faithfulness_checker.py:231`](tests/unit/test_faithfulness_checker.py#L231).

## Files likely involved

- [`rag/evaluator/faithfulness_checker.py`](rag/evaluator/faithfulness_checker.py) — the `check()` method (the join at line 34) is the only production code that changes.
- [`tests/unit/test_faithfulness_checker.py`](tests/unit/test_faithfulness_checker.py) — `test_none_context_chunk_text` (line 231) and `test_missing_text_key_in_chunk` (line 244) define the expected graceful behavior; no test changes should be needed.

## Sub-tasks (step by step)

1. **Reproduce** the crash locally with the issue snippet above and confirm
   `test_none_context_chunk_text` fails with the `TypeError`.
2. **Edit `check()`** in `faithfulness_checker.py`: change the list
   comprehension from `chunk.get("text", "")` to `chunk.get("text") or ""` so a
   `None` or missing value coerces to an empty string before the join.
3. **Guard against non-dict / non-str chunks defensively** (optional hardening):
   confirm the comprehension still produces only strings; if a value could be a
   non-string, wrap with `str(...)`. Decide based on how callers build chunks.
4. **Run the targeted tests** for the checker and confirm the two graceful-handling
   tests pass and the original repro snippet now returns a float.
5. **Run the full checker test module** to confirm no regression in the other
   (unrelated) tests.
6. **Commit** the fix with a message referencing issue #153.

## Inputs and outputs

- **`check(feedback: str, context_chunks: list[dict]) -> float`** — the public
  signature does not change.
  - *Inputs:* `feedback` (a string of generated feedback) and `context_chunks`
    (a list of dicts, each expected to carry a `"text"` string).
  - *Output:* a faithfulness score as a `float` in `0.0`–`1.0` (the ratio of
    supported claims).
- **Behavior that changes:** the value produced by the list comprehension for a
  chunk whose `text` is `None` or missing goes from `None`/`TypeError` to `""`.
  The `context_text` string, and therefore the returned score, is now
  well-defined for such chunks (they simply contribute no tokens).
- **No change** to the score for well-formed chunks — a chunk with a valid
  `text` string produces identical output to before.

## Risks and unknowns

- **Silent masking of upstream bugs** — coercing `None` to `""` hides the fact
  that a retrieval step produced a null chunk. Unknown: *should a `None` chunk be
  logged?* Investigate the retriever that builds `context_chunks` (search the RAG
  pipeline for where chunks are assembled) to decide whether to add a
  `logger.info`/`warning` when a null text is encountered in `check()`.
- **Empty-context edge** — if *every* chunk is `None`/empty, `context_text`
  becomes `""` and every claim scores unsupported (score `0.0`). Confirm in
  `_is_supported` ([line 66](rag/evaluator/faithfulness_checker.py#L66)) that a
  `0.0` score is the intended, non-crashing outcome for this case.
- **Type breadth unknown** — the plan assumes `text` is either a `str`, `None`,
  or missing. If callers can pass other types (int, list), `.get() or ""` still
  won't stringify them; verify chunk construction to know whether `str(...)`
  coercion is warranted (see sub-task 3).

## Edge cases to handle gracefully

1. `{"text": None}` — key present, value `None` (the issue #153 crash). → treat as empty text.
2. `{"content": "..."}` — `text` key missing entirely. → treat as empty text (covered by `test_missing_text_key_in_chunk`).
3. `[{"text": None}, {"text": "Python skills"}]` — a mix of null and valid chunks. → the valid chunk's text still contributes; no crash.
4. All chunks null/empty (e.g. `[{"text": None}, {"text": ""}]`). → `context_text == ""`, score `0.0`, no crash.
5. `{"text": ""}` — empty string (already valid today). → must remain unaffected by the fix.

## Verification

```
pytest tests/unit/test_faithfulness_checker.py -k "none_context_chunk_text or missing_text_key"
pytest tests/unit/test_faithfulness_checker.py    # full module, no regressions
```

Both targeted tests should pass, and the original repro snippet should return a
float instead of raising.

## Out of scope

The reproduction run also surfaced **three unrelated failures**:

- `test_partial_support_returns_middle_score`
- `test_multiple_context_chunks`
- `test_multiple_claims_varying_support`

These fail with `score == 0.0` because of the scoring threshold in
[`_is_supported`](rag/evaluator/faithfulness_checker.py#L66-L88) (requires
`len(meaningful_overlap) >= 2`). This is a **separate concern** from the `None`
crash and is **not part of issue #153**; fixing the crash will not change these
results, and they should not be addressed under this ticket.
