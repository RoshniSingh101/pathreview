# Plan — Issue #153: Faithfulness checker crashes on `text: None`

**Issue:** https://github.com/ascherj/pathreview/issues/153
**Branch:** tree/fix/153-faithfulness

## Root cause

In [`rag/evaluator/faithfulness_checker.py:34-36`](rag/evaluator/faithfulness_checker.py#L34-L36), `check()` concatenates context text with:

```python
context_text = " ".join([
    chunk.get("text", "") for chunk in context_chunks
])
```

`dict.get("text", "")` only substitutes the default `""` when the key is **missing**. When the key is present with value `None` (e.g. `{"text": None}`), `.get()` returns `None`, and `" ".join([...])` raises:

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

## Solution

The test expects `check()` to **handle the bad chunk gracefully and return a
valid float** — not to raise. So the fix is to coerce a `None` (or any
missing/falsy) text value to an empty string before joining:

```python
context_text = " ".join([
    chunk.get("text") or "" for chunk in context_chunks
])
```

`chunk.get("text") or ""` covers both cases:
- present-but-`None` value (`{"text": None}`) — the issue #153 crash
- missing key (`{"content": "..."}`) — already covered by
  `test_missing_text_key_in_chunk` at
  [line 244](tests/unit/test_faithfulness_checker.py#L244)

No exception should be raised; the checker simply treats the chunk as empty text.

## Verification

Run the faithfulness checker tests:

```
pytest tests/unit/test_faithfulness_checker.py -k "none_context_chunk_text or missing_text_key"
```

Both should pass after the change. Then confirm the original repro snippet
returns a float instead of raising.

## Out of scope

The reproduction run also surfaced **three unrelated failures**:

- `test_partial_support_returns_middle_score`
- `test_multiple_context_chunks`
- `test_multiple_claims_varying_support`

These all fail with `score == 0.0` because of the scoring logic in
[`_is_supported`](rag/evaluator/faithfulness_checker.py#L66-L88), which requires
`len(meaningful_overlap) >= 2` keyword matches. This is a **separate concern**
from the `None` crash and is **not part of issue #153**. Fixing the crash will
not change these three results, and they should not be addressed under this
ticket.
