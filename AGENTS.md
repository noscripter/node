# Repository Guidelines

## Project Structure & Module Organization
Node.js core sources live in `src/` (C/C++ bindings and platform glue), while built-in JavaScript modules sit in `lib/`. Third-party dependencies such as V8 and libuv live under `deps/`. Tests are organized in `test/` with suites like `parallel/`, `sequential/`, `cctest/`, and `addons/`. Documentation resides in `doc/`, benchmarks in `benchmark/`, and helper scripts in `tools/`. Build outputs land in `out/`, which should stay out of version control.

## Build, Test, and Development Commands
- `./configure && make -j4`: configure and build a release binary.
- `make build-ci`: reproduce the CI build layout for benchmarking or diffs.
- `./node benchmark/compare.js --old ./node_old --new ./node`: exercise the freshly built binary.
- `make lint` / `make lint-js-fix`: run or auto-fix the mixed JS/C++/Markdown lint suite.
- `tools/test.py test/parallel/test-stream2-transform.js`: execute a single test file when iterating quickly.

## Coding Style & Naming Conventions
JavaScript in `lib/` and `test/` uses two-space indentation, strict mode by default, and descriptive camelCase identifiers. Prefer file names that mirror the exported module (for example `lib/internal/fs/utils.js`). Native code follows the guidance in `doc/contributing/cpp-style-guide.md` and the subsystem overviews in `src/README.md`. Always run `make lint` before submitting; it enforces `eslint`, `cpplint`, and Markdown style rules defined in `eslint.config.mjs`.

## Testing Guidelines
Add new coverage in the closest existing suite: `test/parallel` for isolated async behavior, `test/sequential` for stateful flows, and `test/cctest` for C++ unit tests. Name files `test-<feature>.js` and keep fixtures under `test/fixtures/`. Validate changes with `make test-only`; use `make -j4 test` before opening a PR to include linters. For deeper validation or platform checks, run `tools/test.py --mode=release,debug test/parallel/test-stream2-transform.js` and consider `make coverage` when touching hot paths.

## Commit & Pull Request Guidelines
Commit messages follow `subsystem: imperative summary`, keep the first line ≤72 characters, and wrap details with a blank second line. Reference issues using `Fixes:` or `Refs:` trailers. Group logically related edits per commit and avoid drive-by formatting. Pull requests should describe the motivation, list any platform-specific considerations, and include docs/tests updates. Link related issues, request reviews from subsystem maintainers, and note which CI jobs were exercised locally.
