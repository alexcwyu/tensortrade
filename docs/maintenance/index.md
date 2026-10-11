# tensortrade: portable verification records

Full raw logs are retained in the workspace maintenance evidence, not distributed here. Hashes identify those original logs. A successful build or selected test subset does not imply a full test-suite pass. Commands containing <workspace> describe the original execution environment.

[Machine-readable records](verification.json) · [Repository validation summary](../validation.md)

## Executed commands

| Check | Exit | Duration | Command | Outcomes |
| --- | ---: | ---: | --- | --- |
| build | 0 | unrecordeds | `uv build` | Successfully built dist/tensortrade-1.0.4.dev1.tar.gz<br>Successfully built dist/tensortrade-1.0.4.dev1-py3-none-any.whl |
| build-final | 0 | unrecordeds | `uv build` | Successfully built dist/tensortrade-1.0.4.dev1.tar.gz<br>Successfully built dist/tensortrade-1.0.4.dev1-py3-none-any.whl |
| ewm-reproduction | 1 | unrecordeds | `uv run --no-sync pytest tests/tensortrade/unit/feed/api/float/window/test_ewm.py::test_ewm_mean -x` | See exit status; no test-count assertion. |
| python-final-sync | 0 | unrecordeds | `uv sync --all-groups --all-extras --python 3.13.16` | See exit status; no test-count assertion. |
| python-rllib-sync | 0 | unrecordeds | `uv sync --all-groups --all-extras --python 3.13.16` | See exit status; no test-count assertion. |
| python-sync | 0 | unrecordeds | `uv sync --all-groups --all-extras --python 3.13.16` | See exit status; no test-count assertion. |
| python314-sync | 2 | unrecordeds | `uv sync --all-groups --all-extras --python 3.14.8` | See exit status; no test-count assertion. |
| python315-lock | 0 | unrecordeds | `uv lock --upgrade --python 3.15` | See exit status; no test-count assertion. |
| python315-sync | 2 | unrecordeds | `uv sync --python 3.15 --all-groups --all-extras` | See exit status; no test-count assertion. |
| tests | 2 | unrecordeds | `uv run --no-sync pytest` | See exit status; no test-count assertion. |
| tests-current-pandas-ray | 1 | unrecordeds | `uv run --no-sync pytest` | 251 passed, 2 skipped, 3 errors in 93.68s (0:01:33) |
| tests-fixed-docstrings | 2 | unrecordeds | `uv run --no-sync pytest` | See exit status; no test-count assertion. |
| tests-ray-cleanup | 0 | unrecordeds | `uv run --no-sync pytest` | 251 passed, 2 skipped in 46.17s |
| tests-registered-markers | 2 | unrecordeds | `uv run --no-sync pytest` | See exit status; no test-count assertion. |
| tests-rllib-deps | 1 | unrecordeds | `uv run --no-sync pytest` | 13 failed, 228 passed, 2 skipped, 10 errors in 22.92s |
