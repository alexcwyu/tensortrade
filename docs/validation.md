# Validation evidence

Recorded 2026-10-12 on macOS arm64. Selected Python: **3.13.16**; uv0.13.0; native Cargo/Rust1.99.0. Commands below ran from the repository root. Exit124 means the documented time bound expired; exit139 means a native segmentation fault. An earlier failed attempt is retained even if a later rerun succeeds.

## Current scope and result

Full existing suite:251passed,2pre-existing skips. Strict warning checks remain enabled. Pandas3, numerical domain operations, EWM gap weighting, and current Ray subprocess cleanup were repaired; all existing value assertions remain.

Current recorded status: **complete**. Interpreter fallback evidence and native prerequisites are described in [development](development.md). See [architecture](ayu-develop.md) for workflows, components, and existing source links.

## Executed commands

| Step | Exact command | Exit | Elapsed seconds | Evidence |
|---|---|---:|---:|---|
| python315-sync | `uv sync --python 3.15 --all-groups --all-extras` | 2 | None | [tensortrade-python315-sync.log](../../docs/maintenance-2026-10-12/tensortrade-python315-sync.log) |
| python315-lock | `uv lock --upgrade --python 3.15` | 0 | None | [tensortrade-python315-lock.log](../../docs/maintenance-2026-10-12/tensortrade-python315-lock.log) |
| python314-sync | `uv sync --all-groups --all-extras --python 3.14.8` | 2 | 0.1 | [tensortrade-python314-sync.log](../../docs/maintenance-2026-10-12/tensortrade-python314-sync.log) |
| python-sync | `uv sync --all-groups --all-extras --python 3.13.16` | 0 | 16.65 | [tensortrade-python-sync.log](../../docs/maintenance-2026-10-12/tensortrade-python-sync.log) |
| tests | `uv run --no-sync pytest` | 2 | 10.23 | [tensortrade-tests.log](../../docs/maintenance-2026-10-12/tensortrade-tests.log) |
| build | `uv build` | 0 | 2.4 | [tensortrade-build.log](../../docs/maintenance-2026-10-12/tensortrade-build.log) |
| tests-fixed-docstrings | `uv run --no-sync pytest` | 2 | 34.52 | [tensortrade-tests-fixed-docstrings.log](../../docs/maintenance-2026-10-12/tensortrade-tests-fixed-docstrings.log) |
| tests-registered-markers | `uv run --no-sync pytest` | 2 | 11.37 | [tensortrade-tests-registered-markers.log](../../docs/maintenance-2026-10-12/tensortrade-tests-registered-markers.log) |
| python-rllib-sync | `uv sync --all-groups --all-extras --python 3.13.16` | 0 | 1.34 | [tensortrade-python-rllib-sync.log](../../docs/maintenance-2026-10-12/tensortrade-python-rllib-sync.log) |
| tests-rllib-deps | `uv run --no-sync pytest` | 1 | 25.92 | [tensortrade-tests-rllib-deps.log](../../docs/maintenance-2026-10-12/tensortrade-tests-rllib-deps.log) |
| python-final-sync | `uv sync --all-groups --all-extras --python 3.13.16` | 0 | 1.98 | [tensortrade-python-final-sync.log](../../docs/maintenance-2026-10-12/tensortrade-python-final-sync.log) |
| ewm-reproduction | `uv run --no-sync pytest tests/tensortrade/unit/feed/api/float/window/test_ewm.py::test_ewm_mean -x` | 1 | 13.85 | [tensortrade-ewm-reproduction.log](../../docs/maintenance-2026-10-12/tensortrade-ewm-reproduction.log) |
| tests-current-pandas-ray | `uv run --no-sync pytest` | 1 | 100.59 | [tensortrade-tests-current-pandas-ray.log](../../docs/maintenance-2026-10-12/tensortrade-tests-current-pandas-ray.log) |
| tests-ray-cleanup | `uv run --no-sync pytest` | 0 | 48.6 | [tensortrade-tests-ray-cleanup.log](../../docs/maintenance-2026-10-12/tensortrade-tests-ray-cleanup.log) |
| build-final | `uv build` | 0 | 3.57 | [tensortrade-build-final.log](../../docs/maintenance-2026-10-12/tensortrade-build-final.log) |

## Test summaries and failures

### tests

Exit 2; elapsed 10.23seconds.

```text
ERROR tests/tensortrade/unit/env/default/test_observers.py
ERROR tests/tensortrade/unit/env/default/test_rewards.py
ERROR tests/tensortrade/unit/env/default/test_stream_selector.py
ERROR tests/tensortrade/unit/env/test_gpu_compatibility.py
ERROR tests/tensortrade/unit/stochastic/processes/test_cox.py
ERROR tests/tensortrade/unit/stochastic/processes/test_fbm.py
ERROR tests/tensortrade/unit/stochastic/processes/test_gbm.py
ERROR tests/tensortrade/unit/stochastic/processes/test_heston.py
ERROR tests/tensortrade/unit/stochastic/processes/test_merton.py
ERROR tests/tensortrade/unit/stochastic/processes/test_ornstein_uhlenbeck.py
!!!!!!!!!!!!!!!!!!! Interrupted: 17 errors during collection !!!!!!!!!!!!!!!!!!!
17 errors in 8.50s
```

### tests-fixed-docstrings

Exit 2; elapsed 34.52seconds.

```text
___ ERROR collecting tests/tensortrade/integration/rllib/test_checkpoint.py ____
'rllib' not found in `markers` configuration option
__ ERROR collecting tests/tensortrade/integration/rllib/test_env_creation.py ___
'rllib' not found in `markers` configuration option
__ ERROR collecting tests/tensortrade/integration/rllib/test_ray_training.py ___
'rllib' not found in `markers` configuration option
=========================== short test summary info ============================
ERROR tests/tensortrade/integration/rllib/test_checkpoint.py - Failed: 'rllib...
ERROR tests/tensortrade/integration/rllib/test_env_creation.py - Failed: 'rll...
ERROR tests/tensortrade/integration/rllib/test_ray_training.py - Failed: 'rll...
!!!!!!!!!!!!!!!!!!! Interrupted: 3 errors during collection !!!!!!!!!!!!!!!!!!!!
3 errors in 30.34s
```

### tests-registered-markers

Exit 2; elapsed 11.37seconds.

```text
$ uv run --no-sync pytest

==================================== ERRORS ====================================
___ ERROR collecting tests/tensortrade/integration/rllib/test_checkpoint.py ____
'slow' not found in `markers` configuration option
__ ERROR collecting tests/tensortrade/integration/rllib/test_ray_training.py ___
'slow' not found in `markers` configuration option
=========================== short test summary info ============================
ERROR tests/tensortrade/integration/rllib/test_checkpoint.py - Failed: 'slow'...
ERROR tests/tensortrade/integration/rllib/test_ray_training.py - Failed: 'slo...
!!!!!!!!!!!!!!!!!!! Interrupted: 2 errors during collection !!!!!!!!!!!!!!!!!!!!
2 errors in 8.87s
```

### tests-rllib-deps

Exit 1; elapsed 25.92seconds.

```text
ERROR tests/tensortrade/integration/rllib/test_checkpoint.py::TestCheckpoint::test_checkpoint_save_creates_files
ERROR tests/tensortrade/integration/rllib/test_checkpoint.py::TestCheckpoint::test_checkpoint_restore_loads_algorithm
ERROR tests/tensortrade/integration/rllib/test_checkpoint.py::TestCheckpoint::test_restored_algo_computes_actions
ERROR tests/tensortrade/integration/rllib/test_checkpoint.py::TestLSTMCheckpoint::test_lstm_checkpoint_restore
ERROR tests/tensortrade/integration/rllib/test_env_creation.py::TestRayEnvRegistration::test_env_registration_with_ray
ERROR tests/tensortrade/integration/rllib/test_env_creation.py::TestRayEnvRegistration::test_registered_env_reset_and_step
ERROR tests/tensortrade/integration/rllib/test_ray_training.py::TestPPOTraining::test_ppo_single_iteration
ERROR tests/tensortrade/integration/rllib/test_ray_training.py::TestPPOTraining::test_ppo_with_lstm_model
ERROR tests/tensortrade/integration/rllib/test_ray_training.py::TestPPOTraining::test_ppo_with_attention_model
ERROR tests/tensortrade/integration/rllib/test_ray_training.py::TestTunerConfig::test_tuner_config_validation
FAILED tests/tensortrade/unit/feed/api/float/test_utils.py::test_sqrt - Runti...
FAILED tests/tensortrade/unit/feed/api/float/test_utils.py::test_log - Runtim...
FAILED tests/tensortrade/unit/feed/api/float/test_utils.py::test_pct_change
FAILED tests/tensortrade/unit/feed/api/float/window/test_ewm.py::test_ewm_mean
FAILED tests/tensortrade/unit/feed/api/float/window/test_expanding.py::test_expanding_var
FAILED tests/tensortrade/unit/feed/api/float/window/test_expanding.py::test_expanding_std
FAILED tests/tensortrade/unit/feed/api/float/window/test_rolling.py::test_rolling_var
FAILED tests/tensortrade/unit/feed/api/float/window/test_rolling.py::test_rolling_std
FAILED tests/tensortrade/unit/stochastic/processes/test_cox.py::test_shape - ...
FAILED tests/tensortrade/unit/stochastic/processes/test_fbm.py::test_shape - ...
FAILED tests/tensortrade/unit/stochastic/processes/test_gbm.py::test_shape - ...
FAILED tests/tensortrade/unit/stochastic/processes/test_heston.py::test_shape
FAILED tests/tensortrade/unit/stochastic/processes/test_ornstein_uhlenbeck.py::test_shape
13 failed, 228 passed, 2 skipped, 10 errors in 22.92s
```

### tests-current-pandas-ray

Exit 1; elapsed 100.59seconds.

```text
ERROR tests/tensortrade/integration/rllib/test_checkpoint.py::TestLSTMCheckpoint::test_lstm_checkpoint_restore
ERROR tests/tensortrade/integration/rllib/test_env_creation.py::TestRayEnvRegistration::test_registered_env_reset_and_step
ERROR tests/tensortrade/integration/rllib/test_ray_training.py::TestTunerConfig::test_tuner_config_validation
251 passed, 2 skipped, 3 errors in 93.68s (0:01:33)
```

### tests-ray-cleanup

Exit 0; elapsed 48.6seconds.

```text
251 passed, 2 skipped in 46.17s
```

## Synchronization

All upstream branch and tag refs were fetched by the workspace synchronization procedure. Fork updates use non-forcing pushes; upstream tags also have a private local namespace to preserve divergent tags. The default branch was merged into feature/ayu_develop, preserving the modern package layout and upstream source changes. The workspace synchronization JSON and log retain per-ref outcomes and merge history. This document does not claim that any feature branch was pushed.
