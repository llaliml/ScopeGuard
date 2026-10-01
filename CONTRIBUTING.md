# Contributing

Run `python -m unittest discover -s tests -v` before submitting a change. Keep Python 3.10 compatibility and the dependency-free runtime.

New rules should include a local or synthetic regression test, positive and negative examples, evidence, a context-aware severity, remediation, and a primary guidance reference. Distinguish observations from exploit claims. Avoid recording secrets or making requests outside the explicit target origin.

Do not add live internet scanning to automated tests. Do not attach real customer reports to public issues. Any new active test should be opt-in and clearly documented.
