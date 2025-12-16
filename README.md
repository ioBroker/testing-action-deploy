# testing-action-deploy

Shared GitHub Actions for ioBroker testing workflows: Deploy step

## Inputs

| Input             | Description                                                                                                                                                                                          | Required? |      Default      |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------- | :---------------: |
| `node-version`    | Node.js version to use in tests. Should be LTS.                                                                                                                                                      | ✔         |         -         |
| `install-command` | Overwrite the default install command. When changing this, `package-cache` may need to be disabled.                                                                                                  | ❌        |    `'npm ci'`     |
| `package-cache`   | For which package manager dependencies should be cached. Set to `'false'` or `''` to disable caching. More documentation [here](https://github.com/actions/setup-node#caching-global-packages-data). | ❌        |      `'npm'`      |
| `build`           | Set to `'true'` when the adapter needs a build step before testing                                                                                                                                   | ❌        |      `false`      |
| `build-command`   | Overwrite the default build command                                                                                                                                                                  | ❌        | `'npm run build'` |
| `npm-token`       | The token to use to publish to npm                                                                                                                                                                   | ✔         |         -         |
| `github-token`    | The token to use to create a GitHub release                                                                                                                                                          | ❌         |         -         |


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

## Usage of TRUSTED PUBLISHING

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
