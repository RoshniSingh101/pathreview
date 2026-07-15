## Week 7 — Issue selection

**Issue link:** https://github.com/ascherj/pathreview/issues/153

**Issue title:** Faithfulness checker crashes when a context chunk has text: None #153

**Tier:** [x] Tier 1  [ ] Tier 2  [ ] Tier 3

**Problem summary:** The issue is that in line 34 of faithfulness_checker.py, the context chunks are joined together to form a single string. However, some context chunks may have None as their text value, which causes a TypeError. Therefore, we need to implement error handling to catch and throw the exceptions so that the checker no longer crashes. 

**Branch name:** tree/fix/153-faithfulness

**Setup confirmation:** [x] App runs locally at localhost:5173

**Cohort ledger:** [x] Issue added to cohort ledger