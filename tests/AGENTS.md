# Tests — isolation and behavior contracts

Root invariants apply. Use `scripts/run_tests.sh`, never bare pytest: it clears credential vars/PYTHONPATH, sets UTC/C.UTF-8 and an isolated HERMES_HOME, and runs files in separate subprocesses via `scripts/run_tests_parallel.py` (no xdist).

## Environment and runner

- Build the separate test environment with the checkout's bootstrapped interpreter: `python -m pm.build_env --source . --out .venv --group dev --group test`.
- This is a fresh build. Stop users of an existing disposable output before intentional removal/regeneration; do not sync it in place. For an external output set HERMES_PYTHON to its interpreter; PM activation alone does not supply pytest.
- `scripts/run_tests.sh` runs all tests; append a directory/file and pytest flags (`-k`, `-v`, `--tb=long`) for focused runs. Isolation/retries remain file-granular.
- Failed files retry once fresh; `HERMES_TEST_FILE_RETRIES=0` disables retries. Signal-killed/time-limited files never retry. Report pass-on-retry as flaky and fix it.
- Timing tests use events or `tests.waiting.wait_until`/`async_wait_until`, not added nonzero sleeps (HX013). Allow ≥2s wall-clock bounds and avoid negative wait races under load.

## Placement and homes

- Mirror source under `tests/<source area>/`; only root-module tests stay at tests root. Installer/updater tests go under `tests/scripts/{install,desktop_update}/`. Put issue numbers in module docstrings, not filenames.
- `scripts/ci/classify_changes.py` selects lanes. Assertions about JS/TS source or package/lock/tsconfig metadata belong in Vitest; Python tests will not run for JS-only changes.
- Never write the real user home. `tests/conftest.py::_isolate_hermes_home` redirects HERMES_HOME. Profile tests also patch Path.home so HOME-anchored profile roots stay isolated; patching Path.home alone is insufficient.

## Don't fake the host OS

- Test OS-specific behavior on that OS with ONE `@pytest.mark.platforms(...)`; never fake sys.platform or module IS_WINDOWS flags. Pure helpers taking platform as data and packaging declarations remain unmarked.
- A marker accepts multiple any-of specs and optional arch: linux, macos, windows, posix, any, not <spec>. Example: `@pytest.mark.platforms("linux", "macos")`.
- Do not stack function/module platform markers, use obsolete *_only markers, or platform-gated bare skipif. CI `scripts/ci/list_os_marked_tests.py` selects quoted specs; misspellings/duplicate markers error.
- Split parametrized host rows into marked tests rather than skipping foreign rows. Non-host conditions can use separate skipif alongside the host marker.
- Real Windows process topology uses `windows-venv-e2e.yml` / `tests/hermes_cli/test_venv_holder_windows_live.py` on native Windows; its explicit `wine2e/**` branch lane is inert on ordinary PRs/main. Prove unfixed and fixed behavior; assert the gateway ancestor, not launcher-shim direct parent.

## Assertions

- Assert behavior and relationships, not expected-to-change data: catalogs/config versions/counts must not create update detectors. Migration reaches DEFAULT_CONFIG's current version; every advertised model has metadata; forbidden groups do not leak.
- Do not read source text to assert its shape. Extract a pure or dependency-injected function and call it; matching a source regex cannot prove wiring or runtime behavior.
- Resolution, configuration, security and I/O changes need real-import E2E with isolated state. Add focused regression contracts proven red on the base; keep runtime mocks at external boundaries.
