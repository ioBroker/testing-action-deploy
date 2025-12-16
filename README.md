# testing-action-deploy

Shared GitHub Actions for ioBroker testing workflows: Deploy step

## Inputs

| Input             | Description                                                                                                                                                                                          | Required? |                            Default                             |
|-------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------|:--------------------------------------------------------------:|
| `node-version`    | Node.js version to use in tests. Should be LTS.                                                                                                                                                      | ✔         |                               -                                |
| `install-command` | Overwrite the default install command. When changing this, `package-cache` may need to be disabled.                                                                                                  | ❌         |                           `'npm ci'`                           |
| `package-cache`   | For which package manager dependencies should be cached. Set to `'false'` or `''` to disable caching. More documentation [here](https://github.com/actions/setup-node#caching-global-packages-data). | ❌         |                            `'npm'`                             |
| `build`           | Set to `'true'` when the adapter needs a build step before testing                                                                                                                                   | ❌         |                            `false`                             |
| `build-command`   | Overwrite the default build command                                                                                                                                                                  | ❌         |                       `'npm run build'`                        |
| `npm-token`       | The token to use to publish to npm                                                                                                                                                                   | ❌         | If npm-token is not set, trusted publishing must be activated. |
| `github-token`    | The token to use to create a GitHub release                                                                                                                                                          | ✔         |                               -                                |
| `tag`             | You can provide the custom tag (without starting "v") to release under this tag and not by `github.ref`                                                                                              | ❌         |                               -                                |


If Sentry integration is desired, the following inputs are used to configure it:

| Input                       | Description                                                                                | Required?             |                     Default   |
|-----------------------------|--------------------------------------------------------------------------------------------|-----------------------|-------------------------------|
| `sentry`                    | Set to `'true'` to enable Sentry releases integration                                      | ❌                     | `false`                       |
| `sentry-token`              | The token to use to create a Sentry release                                                | if `sentry` is `true` | -                             |
| `sentry-url`                | Under which URL the Sentry instance is running                                             | ❌                     | `https://sentry.iobroker.net` |
| `sentry-org`                | Which Sentry organization the project is under                                             | ❌                     | `iobroker`                    |
| `sentry-project`            | The project name on Sentry                                                                 | if `sentry` is `true` | -                             |
| `sentry-version-prefix`     | The prefix for release versions on Sentry. Should be something like `iobroker.adaptername` | if `sentry` is `true` | -                             |
| `sentry-github-integration` | Set to true once Github integration is set up in Sentry                                    | ❌                     | `false`                       |
| `sentry-sourcemap-paths`    | If sourcemaps should be uploaded to Sentry, specify their path here                        | ❌                     | -                             |

## Usage
### Pipelines without `official-release` job

```yml
# ... rest of your workflow ...

jobs:
  deploy:
    needs: [check-and-lint, adapter-tests]

    # Trigger this step only when a commit on any branch is tagged with a version number
    if: |
      contains(github.event.head_commit.message, '[skip ci]') == false &&
      github.event_name == 'push' &&
      startsWith(github.ref, 'refs/tags/v')

    runs-on: ubuntu-latest

    # Write permissions are required to create GitHub releases
    permissions:
      id-token: write
      contents: write

    steps:
      - uses: ioBroker/testing-action-deploy@v1
        with:
          node-version: "22.x" # This should be LTS
          # build: 'true' # optional
          npm-token: ${{ secrets.NPM_TOKEN }} # This must be created on https://www.npmjs.com in your profile under "Access Tokens". Omit this line if trusted publishing is activated.
          github-token: ${{ secrets.GITHUB_TOKEN }} # This exists by default in GitHub Actions and does not need to be created.
          # If you want Sentry:
          sentry: true
          sentry-token: ${{ secrets.SENTRY_AUTH_TOKEN }}
          sentry-project: "iobroker-my-adapter"
          sentry-version-prefix: "iobroker.my-adapter"
          # ... other options
```

### Pipelines with `official-release` job (auto-merge)

```yml
# ... rest of your workflow ...

jobs:
  # Deploys the final package to NPM
  auto-merge:
    needs: [adapter-tests]

    if: |
      always() &&
      github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - id: automerge
        name: automerge
        uses: 'pascalgn/automerge-action@v0.16.4'
        env:
          GITHUB_TOKEN: '${{ secrets.GITHUB_TOKEN }}'
          MERGE_LABELS: 'automated pr 🔧'
          MERGE_FILTER_AUTHOR: 'foxriver76'
          MERGE_FORKS: 'false'
          MERGE_DELETE_BRANCH: 'false'
          UPDATE_LABELS: 'automated pr 🔧'
          MERGE_METHOD: 'squash'
          MERGE_COMMIT_MESSAGE: 'pull-request-title-and-description'
          MERGE_RETRIES: '20'
          MERGE_RETRY_SLEEP: 10000

      - name: Checkout repository
        if: steps.automerge.outputs.mergeResult == 'merged'
        uses: actions/checkout@v6
        with:
          fetch-depth: 0 # Fetch the history, or this action won't work
          ref: 'master'

      - name: Use Node.js 20
        if: steps.automerge.outputs.mergeResult == 'merged'
        uses: actions/setup-node@v6
        with:
          node-version: 20

      - name: Determine version
        if: steps.automerge.outputs.mergeResult == 'merged'
        id: version
        uses: actions/github-script@v8
        with:
          result-encoding: string
          script: |
            return require('./package.json').version;

      - name: Publish to NPM
        if: steps.automerge.outputs.mergeResult == 'merged'
        uses: ioBroker/testing-action-deploy@v1.4.2 # version is important here
        with:
          node-version: '22.x'
          build: true
          github-token: ${{ secrets.GITHUB_TOKEN }}
          tag: ${{ steps.version.outputs.result }}
```

### Usage of TRUSTED PUBLISHING

npm recommends to use trusted publishing. As npm tokens are no longer available for more than 90 days, trusted publishing is the only way to configure a permanent solution.
Please follow the official [guide to set up trusted publishing for github](https://docs.npmjs.com/trusted-publishers#configuring-trusted-publishing).

After setting up the configuration at npmjs.com please ensure that your workflow has been adapted as following:
- the following permissions are set at workflow test-and.release.yml
  ```yml
  permissions:
      id-token: write
      contents: write
  ```
  
- a npm-token is NOT provided
  ```yml
  - uses: ioBroker/testing-action-deploy@v1
    with:
      node-version: "22.x" # This should be LTS
      # build: 'true' # optional
      npm-token: ${{ secrets.NPM_TOKEN }} # This must be created on https://www.npmjs.com in your profile under "Access Tokens". Omit this line if trusted publishing is activated.
      github-token: ${{ secrets.GITHUB_TOKEN }} # This exists by default in Github Actions and does not need to be created.
  ```
