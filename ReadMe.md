# Monorepo test repository

In this repository, we test useful workflows and more for mono-repositories.

## Create custom GitHub App for token generation

To create a custom GitHub App, you need to follow the steps below. We are using this app to generate dynamically GitHub Tokens for the workflows.

1. Go to [Settings -> Developer settings -> GitHub Apps](https://github.com/settings/apps)
2. Click on "New GitHub App"
3. Fill in the required fields
   1. GitHub App name: a custom name
   2. Homepage URL: Your github user page
   3. Disable "Expire user authorization tokens"
   4. Disable "Webhook active"
   5. Add the following permission on "Repository permissions"
      1. Actions: Read & Write
      2. Administration: Read & Write
      3. Contents: Read & Write
      4. Packages: Read & Write
      5. Metadata: Read-only
      6. Pull requests: Read & Write
      7. Workflows: Read & Write
4. Click on "Create GitHub App"
5. Go to "Generate a private key" and click on "Generate private key"
6. Save the private key in a secure place
7. Go to "Install App" and click on "Install"
   1. Select the repositories where you want to use the app
   2. Click on "Install"
8. Go to "Settings" of the App and click on "General"
9. Copy the "App ID" and save it in a secure place
10. Go to the repository and open "Settings" and click on "Secrets and variables"
11. Go to "Actions" and click "New repository secret"
12. Create a new secret with the name "TOKEN_APP_ID" and paste the "App ID" from *9.* in the value field
13. Create new secret with the name "TOKEN_APP_PRIVATE_KEY" and paste the private key from *5.* in the value field

## Workflows

### New Version (Tag and Changelog)

To create a new version and a changelog with the newest changes, you need to follow the steps below.

1. Create a new workflow file in the `.github/workflows` folder
```yaml
# Path: .github/workflows/new-version.yml
name: Create new version
on:
  push:
    branches:
      - main
jobs:
  release-please:
    runs-on: ubuntu-latest
    steps:
      - name: Generate token
        id: generate_token
        uses: tibdex/github-app-token@586e1a624db6a5a4ac2c53daeeded60c5e3d50fe
        with:
          app_id: ${{ secrets.TOKEN_APP_ID }}
          private_key: ${{ secrets.TOKEN_APP_PRIVATE_KEY }}
      - uses: google-github-actions/release-please-action@v3
        with:
          command: manifest
          token: ${{ steps.generate_token.outputs.token }}
```

2. Create a `.release-please-manifest.json` at the root of the repository
```json
{
  ".": "0.0.1"
}
```

3. Create a `release-please-config.json` at the root of the repository
```json
{
  "packages": {
    ".": {
      "release-type": "simple",
      "include-v-in-tag": false,
      "draft": false,
      "prerelease": false,
      "bumpMinorPreMajor": false,
      "bumpPatchForMinorPreMajor": false,
      "changelogPath": "CHANGELOG.md",
      "versioning": "default"
    }
  }
}
```

### Split-Packages

This workflow is triggered when a new tag is pushed to the repository. It will split all packages that are configured in the strategy-matrix in this workflow and pushes them into their repository with the all commits to this package and the new tag.

If there is a new repository added, you should trigger this workflow manually to create the main branch.

```yaml
name: Monorepo Split

on:
  push:
    tags: [ '*.*.*' ]
  workflow_dispatch: # for initial setup or if a new package was added

env:
  REPOSITORY_OWNER: erkenes # this is the organization name
  REPOSITORY_PROTOCOL: https://
  REPOSITORY_HOST: github.com # this is the host name of the repository
  REPOSITORY_NAME: monorepo-test # this is the repository name

jobs:
  packages_split:
    runs-on: ubuntu-latest

    strategy:
      fail-fast: false
      matrix:
        # this is the list of packages that should be split
        package:
          -
            local_path: 'Packages/First.Package' # this is the local path of the package
            repository: 'erkenes/monorepo-test-target-1' # this is the repository name of the target repository
            default_branch: 'main' # this is the default branch of the target repository
          -
            local_path: 'Packages/Second.Package'
            repository: 'erkenes/monorepo-test-target-2'
            default_branch: 'main'

    steps:
      -
        name: Generate token
        id: generate_token
        uses: tibdex/github-app-token@586e1a624db6a5a4ac2c53daeeded60c5e3d50fe
        with:
          app_id: ${{ secrets.TOKEN_APP_ID }}
          private_key: ${{ secrets.TOKEN_APP_PRIVATE_KEY }}

      - uses: actions/checkout@v3

      # set branch or tag name as release version
      - name: Set release version for branch or tag
        run: echo "RELEASE_VERSION=${{ github.ref }}" >> $GITHUB_ENV

      - uses: erkenes/monorepo-split-action@1.3.0 # this is the action that will split the packages
        with:
          access_token: 'x-access-token:${{ steps.generate_token.outputs.token }}'
          repository_protocol: ${{ env.REPOSITORY_PROTOCOL }} # this is the protocol of the repository
          repository_host: ${{ env.REPOSITORY_HOST }} # this is the host name of the repository
          repository_organization: ${{ env.REPOSITORY_OWNER }} # this is the organization name
          repository_name: ${{ env.REPOSITORY_NAME }} # this is the repository name
          default_branch: ${{ matrix.package.default_branch }} # this is the default branch of the target repository
          target_branch: ${{ env.RELEASE_VERSION }} # this is the branch or tag name that will be created in the target repository
          package_directory: ${{ matrix.package.local_path }} # this is the local path of the package
          remote_repository: '${{ env.REPOSITORY_PROTOCOL }}${{ env.REPOSITORY_HOST }}/${{ matrix.package.repository }}.git' # this is the repository name of the target repository
          remote_repository_access_token: 'x-access-token:${{ steps.generate_token.outputs.token }}' # this is the access token for the target repository
```

#### Branch protection in target repositories

If you want to protect the main branch in the target repositories, you should add the following branch protection rules:


* Branch name pattern: `main`
* Require pull request reviews before merging: `true`
  * Required approving reviews: `1`
  * Dismiss stale pull request approvals when new commits are pushed: `true`
  * Require review from Code Owners: `true`
* Do not allow to bypassing the above settings: `false`
* Allow force pushes `true`
  * Specify who can force push: Add your generated GitHub App
