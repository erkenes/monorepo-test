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
