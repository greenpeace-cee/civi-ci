# civi-ci

GitHub Actions CI for CiviCRM extensions, as two reusable workflows:

- [`tests.yml`](.github/workflows/tests.yml) builds a CiviCRM instance with
  [civibuild](https://github.com/civicrm/civicrm-buildkit) for every
  combination of a test matrix, installs the calling repository as an
  extension and runs its PHPUnit tests.
- [`civilint.yml`](.github/workflows/civilint.yml) runs CiviCRM's `civilint`
  and posts `phpcbf` fixes as suggested changes on pull requests.

## Tests

Add `.github/workflows/unit-tests.yml` to the extension repository:

```yaml
name: "Run unit tests"

on:
  push:
    branches:
      - main
  pull_request:

jobs:
  tests:
    uses: greenpeace-cee/civi-ci/.github/workflows/tests.yml@main
```

That is all an extension without special needs has to do. The defaults
(CiviCRM 6.10 and master, PHP 8.1, MariaDB 11.4, `standalone-clean`,
`ubuntu-24.04`) are maintained here and apply to every extension that does
not override them. Commits whose message starts with `WIP` are skipped.

### Matrix

The matrix dimensions are JSON arrays. Each one is resolved in this order:

1. A repository or organization
   [variable](https://docs.github.com/en/actions/learn-github-actions/variables)
   named `CIVI_CI_<INPUT>`. Variables win over the workflow file so the matrix
   can be adjusted for one repository, or temporarily, without a commit.
2. The `with:` input of the calling workflow.
3. The default in `tests.yml`.

| Input        | Variable             | Default                      | Notes                                                   |
|--------------|----------------------|------------------------------|---------------------------------------------------------|
| `civicrm`    | `CIVI_CI_CIVICRM`    | `["6.10", "master"]`         | Passed to `civibuild create --civi-ver`                 |
| `php`        | `CIVI_CI_PHP`        | `["8.1"]`                    | Used for CLI (civibuild, cv, phpunit) and Apache        |
| `database`   | `CIVI_CI_DATABASE`   | `["mariadb:11.4"]`           | Docker image: `mariadb:<version>` or `mysql:<version>`  |
| `build-type` | `CIVI_CI_BUILD_TYPE` | `["standalone-clean"]`       | civibuild type, e.g. `drupal-clean`, `wp-demo`          |
| `os`         | `CIVI_CI_OS`         | `["ubuntu-22.04"]`           | Runner image                                            |
| `excludes`   | `CIVI_CI_EXCLUDES`   | `[]`                         | Combinations to skip, like `strategy.matrix.exclude`    |

```yaml
jobs:
  tests:
    uses: greenpeace-cee/civi-ci/.github/workflows/tests.yml@main
    with:
      civicrm: '["5.75", "6.10", "master"]'
      php: '["8.1", "8.3"]'
      build-type: '["drupal-clean"]'
      excludes: '[{"civicrm": "5.75", "php": "8.3"}]'
```

### Dependencies

Extensions that the tested extension depends on are cloned into the extension
directory before it is installed. One per line: a git URL, optionally followed
by a branch or tag. Dependencies that ship a `composer.json` get a
`composer install --no-dev`. Enabling them is left to `cv en`, which follows
the `<requires>` in `info.xml`.

```yaml
    with:
      dependencies: |
        https://github.com/Project60/org.project60.sepa.git
        https://github.com/Project60/org.project60.banking.git 1.0
        https://github.com/greenpeace-cee/mjwshared.git
```

### Install and test scripts

The remaining two steps are Bash scripts that can be replaced:

| Input            | Default                                         | Working directory                     |
|------------------|-------------------------------------------------|---------------------------------------|
| `install-script` | `cv en "$EXTENSION"`                            | The civibuild build (`$BUILD_DIR`)    |
| `test-script`    | `phpunit9 --log-junit "$JUNIT_DIR/phpunit.xml"` | The extension (`$EXT_DIR/$EXTENSION`) |

Both run with buildkit's `bin` on `$PATH` and these variables:

- `EXTENSION`: the extension key, from `info.xml` or the `extension` input.
- `EXT_DIR`: the CiviCRM extension directory. The repository was copied to
  `$EXT_DIR/$EXTENSION` before `install-script` runs.
- `BUILD_DIR`: the civibuild build directory (`cv` and `civibuild` work there).
- `JUNIT_DIR`: an empty directory for JUnit XML reports

```yaml
    with:
      install-script: |
        composer install --working-dir="$EXT_DIR/$EXTENSION" --no-dev
        cv en "$EXTENSION"
        cv api4 Setting.set '{"values": {"my_setting": "ci"}}'
      test-script: |
        phpunit9 --group headless --log-junit "$JUNIT_DIR/headless.xml"
        phpunit9 --group e2e --log-junit "$JUNIT_DIR/e2e.xml"
```

`phpunit9` picks `CIVICRM_UF=UnitTests` or the real CMS itself depending on
whether the suite contains headless or end-to-end tests. Set it explicitly in
`test-script` if a suite needs it before the first test boots.

## civilint

Add `.github/workflows/civilint.yml` to the extension repository:

```yaml
name: "civilint"

on:
  pull_request:

permissions:
  contents: read
  # Needed by reviewdog to post suggested changes as review comments.
  pull-requests: write

jobs:
  civilint:
    uses: greenpeace-cee/civi-ci/.github/workflows/civilint.yml@main
```

Like core's style check, it runs `civilint` (`php -l`, phpcs with the CiviCRM
coding standard, jshint) on pull requests only, over the files the pull request
touches; any finding in those files fails the job. Findings are reported as
inline annotations, and `phpcbf` fixes are posted as review suggestions via
reviewdog, which is why the caller has to grant `pull-requests: write`.

`php -l` checks against the first PHP version of the test matrix (the
`CIVI_CI_PHP` variable, or 8.1). Pass `with: php: "8.3"` to use another one.

## Testing changes to civi-ci

Point the calling workflow at a branch of this repository:

```yaml
    uses: greenpeace-cee/civi-ci/.github/workflows/tests.yml@my-branch
```

## Composite action (deprecated)

[`action.yml`](action.yml) is the previous generation: a composite action that
only builds CiviCRM and leaves matrix, database service, extension installation
and tests to each repository. It is kept so that existing workflows using
`greenpeace-cee/civi-ci@main` keep working, but receives no further changes
and will be removed once all extensions use the reusable workflow.
