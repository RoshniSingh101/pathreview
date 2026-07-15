## Week 7 — Issue selection

**Issue link:** https://github.com/ascherj/pathreview/issues/153

**Issue title:** Faithfulness checker crashes when a context chunk has text: None #153

**Tier:** [x] Tier 1  [ ] Tier 2  [ ] Tier 3

**Problem summary:** In `check()` (line 34 of faithfulness_checker.py), the context chunks are joined together into a single string with `" ".join(...)`. The text is pulled with `chunk.get("text", "")`, but the `""` default only applies when the `text` key is *missing* — when the key is present with value `None`, `.get()` returns `None`, and joining a `None` into the string raises a `TypeError`. Currently, passing a chunk like `{"text": None}` crashes the checker instead of scoring the feedback. A successful fix makes `check()` handle a `None` (or missing) text value gracefully — coercing it to an empty string so the join succeeds — and return a valid faithfulness score (a float in `0.0`–`1.0`) without raising.

**Selection reasoning:** I selected this Tier 1 issue because it is well-scoped and matches my current comfort with the Python codebase — it is a single-file fix in `faithfulness_checker.py` with a clear reproduction snippet from the issue and an existing failing test (`test_none_context_chunk_text`) that defines exactly what "fixed" looks like. The narrow, testable scope makes it a good fit rather than one I picked just because it looked interesting.

**Branch name:** tree/fix/153-faithfulness

**Setup confirmation:** [x] App runs locally at localhost:5173

**Cohort ledger:** [x] Issue added to cohort ledger

---

## Week 8 — Reproduction & solution planning

**Reproduction commit link:** _<paste the commit URL here once the reproduction is committed, e.g. https://github.com/RoshniSingh101/pathreview/commit/<sha>>_

**Reproduction output:**
```
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/Users/roshnisingh/Desktop/pathreview/rag/evaluator/faithfulness_checker.py", line 34, in check
    context_text = " ".join([
                   ^^^^^^^^^^
TypeError: sequence item 0: expected str instance, NoneType found
```

**Reproduction summary:**
I called the following script listed in the issue in a new python terminal:
```
>>> from rag.evaluator.faithfulness_checker import FaithfulnessChecker
>>> FaithfulnessChecker().check('Knows Python.', [{'text': None}])
```
I also ran pytest test_faithfulness_checker.py to get a better feel for how the tests were structured and how the checker was supposed to work.

```
======================================================== test session starts =========================================================
platform darwin -- Python 3.12.7, pytest-9.1.1, pluggy-1.6.0
benchmark: 5.2.3 (defaults: timer=time.perf_counter disable_gc=False min_rounds=5 min_time=0.000005 max_time=1.0 calibration_precision=10 warmup=False warmup_iterations=100000)
rootdir: /Users/roshnisingh/Desktop/pathreview
configfile: pyproject.toml
plugins: benchmark-5.2.3, cov-7.1.0, asyncio-1.4.0, anyio-4.14.2, pytest_httpserver-1.1.5, hypothesis-6.156.6
asyncio: mode=Mode.STRICT, debug=False, asyncio_default_fixture_loop_scope=None, asyncio_default_test_loop_scope=function
collected 22 items                                                                                                                   

test_faithfulness_checker.py ..F...F......F....F...                                                                            [100%]

============================================================== FAILURES ==============================================================
_________________________________ TestFaithfulnessChecker.test_partial_support_returns_middle_score __________________________________

self = <tests.unit.test_faithfulness_checker.TestFaithfulnessChecker object at 0x104a07c20>
checker = <rag.evaluator.faithfulness_checker.FaithfulnessChecker object at 0x104a30a40>

    def test_partial_support_returns_middle_score(self, checker):
        """Test partial support returns score between 0 and 1."""
        feedback = "The developer shows Python expertise and Kubernetes knowledge."
        context_chunks = [
            {
                "text": "Strong Python programming skills demonstrated in projects."
            },
        ]
    
        score = checker.check(feedback, context_chunks)
    
        assert isinstance(score, float)
        assert 0.0 <= score <= 1.0
        # Partial support should be middle range
>       assert 0.2 < score < 0.8
E       assert 0.2 < 0.0

test_faithfulness_checker.py:61: AssertionError
-------------------------------------------------------- Captured stdout call --------------------------------------------------------
2026-07-15 13:00:13 [info     ] faithfulness_checked           claims_count=1 score=0.0 supported_count=0
________________________________________ TestFaithfulnessChecker.test_multiple_context_chunks ________________________________________

self = <tests.unit.test_faithfulness_checker.TestFaithfulnessChecker object at 0x104a306e0>
checker = <rag.evaluator.faithfulness_checker.FaithfulnessChecker object at 0x104a32090>

    def test_multiple_context_chunks(self, checker):
        """Test multiple context chunks contribute to score."""
        feedback = "The developer has Python, JavaScript, and Docker experience."
        context_chunks = [
            {"text": "Python expertise shown in backend projects."},
            {"text": "JavaScript skills demonstrated in frontend development."},
            {"text": "Docker and containerization knowledge evident in CI/CD pipelines."},
        ]
    
        score = checker.check(feedback, context_chunks)
    
        assert isinstance(score, float)
        assert 0.0 <= score <= 1.0
        # All three claims supported
>       assert score > 0.5
E       assert 0.0 > 0.5

test_faithfulness_checker.py:106: AssertionError
-------------------------------------------------------- Captured stdout call --------------------------------------------------------
2026-07-15 13:00:13 [info     ] faithfulness_checked           claims_count=1 score=0.0 supported_count=0
____________________________________ TestFaithfulnessChecker.test_multiple_claims_varying_support ____________________________________

self = <tests.unit.test_faithfulness_checker.TestFaithfulnessChecker object at 0x104a319d0>
checker = <rag.evaluator.faithfulness_checker.FaithfulnessChecker object at 0x104a320f0>

    def test_multiple_claims_varying_support(self, checker):
        """Test scoring with multiple claims of varying support."""
        feedback = "Python expert. Knows Rust. Skilled with Docker."
        context_chunks = [
            {"text": "Python and Docker expertise shown in projects."}
        ]
    
        score = checker.check(feedback, context_chunks)
    
        # Two claims supported, one not
        assert isinstance(score, float)
>       assert 0.2 < score < 0.8
E       assert 0.2 < 0.0

test_faithfulness_checker.py:184: AssertionError
-------------------------------------------------------- Captured stdout call --------------------------------------------------------
2026-07-15 13:00:13 [info     ] faithfulness_checked           claims_count=2 score=0.0 supported_count=0
________________________________________ TestFaithfulnessChecker.test_none_context_chunk_text ________________________________________

self = <tests.unit.test_faithfulness_checker.TestFaithfulnessChecker object at 0x104a31c10>
checker = <rag.evaluator.faithfulness_checker.FaithfulnessChecker object at 0x104a32c90>

    def test_none_context_chunk_text(self, checker):
        """Test handling of None in context chunk text."""
        feedback = "Has Python skills"
        context_chunks = [
            {"text": None}
        ]
    
>       score = checker.check(feedback, context_chunks)
                ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

test_faithfulness_checker.py:238: 
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ 

self = <rag.evaluator.faithfulness_checker.FaithfulnessChecker object at 0x104a32c90>, feedback = 'Has Python skills'
context_chunks = [{'text': None}]

    def check(self, feedback: str, context_chunks: list[dict]) -> float:
        """Check faithfulness of feedback to context.
    
        Args:
            feedback: Generated feedback text
            context_chunks: Retrieved context chunks
    
        Returns:
            Faithfulness score 0.0-1.0 (ratio of supported claims)
        """
        if not feedback or not context_chunks:
            logger.info("faithfulness_empty_input", has_feedback=bool(feedback),
                       has_chunks=bool(context_chunks))
            return 0.0
    
        # Extract key claims from feedback (sentences)
        claims = self._extract_claims(feedback)
        if not claims:
            logger.info("faithfulness_no_claims_extracted")
            return 0.5  # Default to neutral if no extractable claims
    
        # Concatenate context text
>       context_text = " ".join([
            chunk.get("text", "") for chunk in context_chunks
        ])
E       TypeError: sequence item 0: expected str instance, NoneType found

../../rag/evaluator/faithfulness_checker.py:34: TypeError
====================================================== short test summary info =======================================================
FAILED test_faithfulness_checker.py::TestFaithfulnessChecker::test_partial_support_returns_middle_score - assert 0.2 < 0.0
FAILED test_faithfulness_checker.py::TestFaithfulnessChecker::test_multiple_context_chunks - assert 0.0 > 0.5
FAILED test_faithfulness_checker.py::TestFaithfulnessChecker::test_multiple_claims_varying_support - assert 0.2 < 0.0
FAILED test_faithfulness_checker.py::TestFaithfulnessChecker::test_none_context_chunk_text - TypeError: sequence item 0: expected str instance, NoneType found
==================================================== 4 failed, 18 passed in 0.33s ====================================================
```

**PLAN.md link:** [click here to access the PLAN.md file](PLAN.md)


**Blockers or open questions:**
I need to understand how to properly test code within the PATHREview environment, specifically with regards to the LLM and the faithfulness checker. I also want to understand how exceptions are handled in the rest of the codebase before proceeding with the PLAN.md file.

---

## Week 9 — Solution building & PR submission

### Check-in 1 (mid-week)

**Current progress:**
Implemented the core fix from PLAN.md. Sub-tasks completed:
- Reproduced the crash locally and confirmed `test_none_context_chunk_text` failed with `TypeError`.
- Edited `check()` in `rag/evaluator/faithfulness_checker.py` to coerce a `None`/missing chunk text to an empty string (`chunk.get("text") or ""`) before the join.
- Added two edge-case tests (mixed None+valid chunks, all-None chunks).
- Ran `make test-unit`: the target test and both new tests pass; the three pre-existing scoring failures are unchanged.

**Next steps:**
- Run `make check` and confirm no new lint/format/type errors are introduced.
- Open a draft PR, request peer/mentor feedback, then mark ready for review.

**Blockers:**
None. Noted that the repo has ~53 pre-existing unit-test failures and pre-existing `make check` errors unrelated to this issue; documenting them in the PR so my scope stays limited to issue #153.

---

### Check-in 2 (end of week)

**PR link:** https://github.com/RoshniSingh101/pathreview/pull/1

**Branch:** `tree/fix/153-faithfulness`

**What you built:**
Fixed `FaithfulnessChecker.check()` so a context chunk whose `text` key is present but `None` (or missing/falsy) no longer crashes the join with a `TypeError`. The value is now coerced to an empty string, so such chunks simply contribute no tokens and the checker returns a valid float score.

**Tests added or updated:**
`tests/unit/test_faithfulness_checker.py` — the existing `test_none_context_chunk_text` now passes; added `test_mixed_none_and_valid_chunk_text` (a valid chunk still contributes when another is `None`) and `test_all_chunks_none_text` (all-`None` chunks score `0.0` without crashing).

**Self-review confirmation:** [x] make check passes  [x] make test-unit passes

_Note on "passes": this repo has documented pre-existing failures (≈53 failing unit tests and pre-existing lint/type errors in unrelated modules). Baseline before my change: 53 failed / 375 passed. After my change: 52 failed / 378 passed — my change fixes the target test, adds two passing tests, and introduces no new failures. My edited lines are black- and ruff-clean and mypy reports no new errors._

**Draft PR feedback received from:** Pending
