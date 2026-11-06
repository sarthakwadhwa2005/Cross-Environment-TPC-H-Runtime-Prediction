# Contributing to CETP

Thank you for contributing to Cross-Environment TPC-H Runtime Prediction (CETP).
Issues, documentation improvements, reproducibility fixes, tests, and focused code
changes are welcome.

## Before you start

1. Read the [README](README.md) to understand the project and its limitations.
2. Check existing issues and pull requests before starting substantial work.
3. For changes that affect published metrics, read the canonical-results guidance
   in the README and the relevant documentation under [`docs/`](docs/).

## Development setup

CETP requires Python 3.11 or newer.

```bash
python -m pip install -e .
python -m pip install -r requirements.txt
```

The research dependencies are intentionally separate from the lightweight runtime.
Install them only when working on analysis or result-generation scripts:

```bash
python -m pip install -e ".[research]"
```

## Testing and verification

Run the test suite before opening a pull request:

```bash
python -m pytest tests -q
```

You can also run the built-in self-test and gate demo:

```bash
cetp selftest
python gate/cetp_gate.py --demo
```

When modifying result-generation code, verify that regenerated outputs are
deterministic:

```bash
make verify-determinism
```

Do not treat files in `results/` as independently editable sources of truth.
Regenerate published results using the repository's supported commands and explain
any intentional metric changes in the pull request.

## Code and documentation guidelines

- Keep changes focused and preserve existing behavior unless a change is intentional
  and documented.
- Follow the existing Python style and naming conventions.
- Add or update tests for behavioral changes and regression fixes.
- Update directly related documentation and examples.
- Do not commit credentials, private query data, generated local files, or database
  dumps.
- Before sharing PostgreSQL plans, remove embedded literals with:

  ```bash
  cetp redact plan.json -o plan.redacted.json
  ```

## Pull requests

A good pull request should:

- clearly describe the problem and the solution;
- identify relevant tests or verification commands;
- call out changes to models, datasets, metrics, or generated artifacts;
- mention known limitations or follow-up work;
- keep unrelated formatting or refactoring out of scope.

Pull requests may be reviewed for correctness, reproducibility, test coverage,
documentation, and protection of sensitive query information.

## Reporting issues

Include the CETP version or commit, Python version, operating system, command
executed, expected behavior, actual behavior, and a minimal redacted reproduction
when possible. Never include secrets or unredacted production query plans.
