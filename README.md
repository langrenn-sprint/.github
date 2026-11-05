# .github

Read the docs at https://langrenn-sprint.github.io/docs/

## Reusable workflows

### `build-test-publish.yml`

Builds the service image once with OCI labels (`org.opencontainers.image.revision`, `created`, `source`), runs the tests against that image, and, with `publish: true`, pushes the tested image to `ghcr.io/<owner>/<repo>` as `latest` and `sha-<short>`.

The tests run with `IMAGE_PREBUILT=1`. Repos with pytest-docker contract tests should override `docker_setup` in `tests/conftest.py` so they don't rebuild:

```python
@pytest.fixture(scope="session")
def docker_setup() -> list[str]:
    if env.get("IMAGE_PREBUILT") == "1":
        return ["up --wait"]
    return ["up --build --wait"]
```

Pull request workflow:

```yaml
on:
  pull_request:
    types: [ready_for_review, opened, reopened, synchronize]
    branches: [main]

jobs:
  build-and-test:
    if: github.event.pull_request.draft == false
    permissions:
      contents: read
    uses: langrenn-sprint/.github/.github/workflows/build-test-publish.yml@main
    with:
      test-env: |
        DB_NAME=test
    secrets:
      JWT_SECRET: ${{ secrets.JWT_SECRET }}
      ADMIN_USERNAME: ${{ secrets.ADMIN_USERNAME }}
      ADMIN_PASSWORD: ${{ secrets.ADMIN_PASSWORD }}
```

Push to main: the same, but with `publish: true` and `packages: write`:

```yaml
on:
  push:
    branches: [main]

jobs:
  build-test-publish:
    permissions:
      contents: read
      packages: write
    uses: langrenn-sprint/.github/.github/workflows/build-test-publish.yml@main
    with:
      publish: true
    secrets:
      JWT_SECRET: ${{ secrets.JWT_SECRET }}
      ADMIN_USERNAME: ${{ secrets.ADMIN_USERNAME }}
      ADMIN_PASSWORD: ${{ secrets.ADMIN_PASSWORD }}
```

| Input | Default | |
| --- | --- | --- |
| `publish` | `false` | Push `latest` and `sha-<short>` after the tests pass |
| `uv-version` | latest | uv version to install |
| `uv-sync-args` | `--locked --all-extras --dev` | Arguments to `uv sync` |
| `test-command` | `uv run poe release` | Linters, checks and tests |
| `test-env` | | Extra non-secret env vars, one `KEY=value` per line |

Optional secrets: `JWT_SECRET`, `ADMIN_USERNAME`, `ADMIN_PASSWORD`, `DB_USER`, `DB_PASSWORD`. Only the secrets that are passed are set for the tests.
