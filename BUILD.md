# Build details

## Local builds

To build locally you do need:

once:
```
(have at least nodejs >=22.17 installed)
npm install -g typescript
npm install -g vsce
yarn install
# Go to the webview source directory
cd /src/webview
# Install dependencies using yarn
yarn install
# Go back to fishbone directory
cd ../..
````
then
```
vsce package
```
and you can install your own generated package (fishbone-...vsix).

## Creating a PR

After testing your changes locally you can simply create a PR the usual github way.
Please use branch name "feature/..." or "feat/..." for features and "fix/..." for fixes.

## CI setup

I describe the setup here that is used to build and release versions.
The releases are generated automatically in CI.

The following steps are done:
- enforce commit message rules to be able to
  - autogenerate changelog (via semantic-release)
  - automatically define the version (via semantic-release)
- on push to main branch: generate and publish a new release.

### Commit message rules

The [conventional commits](https://www.conventionalcommits.org/en/v1.0.0/) rules are used.

commitlint is used:
```sh
yarn add -D @commitlint/{cli,config-conventional}
or
npm install --save-dev @commitlint/{cli,config-conventional}
```

We use the config in the package.json (as we use git hooks ... later as well):
```json
 "commitlint": {
    "extends": [
      "@commitlint/config-conventional"
    ]
  }
```

Husky v6 is used for easy git commit hooks on local setup:
```sh
npm install --save-dev husky
or 
yarn add -D husky
```
and activated in package.json as well:
```json
"scripts": {
  "prepare": "husky install",
  ...
```
and
```sh
npx husky add .husky/commit-msg 'npx --no-install commitlint --edit "$1"' 
```

activated as github action (see file `.github/workflows/commitlint.yml`)
```yml
name: Lint Commit Messages
on:
  push:
    # release.yml workflow does it already there.
    branches-ignore:
      - main
      - gh-pages
  pull_request:
jobs:
  commitlint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
      - uses: wagoid/commitlint-github-action@v5
```

### Semantic release

Semantic-release is used to
 - determine the next semantic version based on the commit messages and last version tag
 - create/update release notes / CHANGELOG.md
 - update package.json version
 - create a github release tag/version

```sh
yarn add -D semantic-release @semantic-release/changelog @semantic-release/git
yarn add -D semantic-release-vsce
or
npm install --save-dev semantic-release @semantic-release/changelog @semantic-release/git
npm install --save-dev semantic-release-vsce
```
and configured in package.json via:
```json
"release": {
    "branches": [
      "main"
    ],
    "plugins": [
       [
        "@semantic-release/commit-analyzer",
        {
          "releaseRules": [
            {
              "type": "docs",
              "scope": "readme",
              "release": "patch"
            }
          ]
        }
      ],
      "@semantic-release/release-notes-generator",
      [
        "@semantic-release/changelog",
        {
          "changelogFile": "CHANGELOG.md",
          "changelogTitle": "# Change log for 'Fishbone':"
        }
      ],
      [
        "@semantic-release/npm",
        {
          "npmPublish": false
        }
            ],
            [
                "semantic-release-vsce",
                {
                    "packageVsix": "fishbone.vsix"
                }
            ],
      [
        "@semantic-release/github",
        {
          "assets": [
            {
              "path": "fishbone.vsix",
              "label": "Fishbone Visual Studio Code extension package"
            }
          ]
        }
      ],
      [
        "@semantic-release/git",
        {
          "message": "chore(release): ${nextRelease.version} [skip ci]"
        }
      ]
    ],
    "preset": "conventionalcommits"
  }
```

In addition to the conventional-commits rule a commit of type ```'docs'``` and scope ```'readme'``` triggers a patch release as well.
Besides that only the default rules apply:
- breaking -> major
- revert -> patch
- feat -> minor
- fix -> patch
- perf -> patch.

the last non automatic released version (v0.9.6) was tagged via
```sh
git tag v0.9.6 98f0d72513e0109594c0c7a44c0d660e878f0d1a
git push origin v0.9.6
```

as semantic-release doesn't support major version being 0 and I didn't want to introduce an artifical breaking change I manually created a tag v1.0.0:

```sh
git tag v1.0.0 9c074d9cd932f3ef26096c904445fd8abff0ce4e
git push origin v1.0.0
```

with this a manual release can be created via
```sh
VSCE_TOKEN=<your pat> GH_TOKEN=... npx semantic-release --no-ci
````

activated as github action (see file `.github/workflows/release.yml`)
```yml
name: Semantic Release and Publish
on:
  push:
    branches:
      - main
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3
        with:
          fetch-depth: 0
      - name: Commitlint
        uses: wagoid/commitlint-github-action@v5
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: "lts/*"
      - name: Install dependencies
        run: yarn install --frozen-lockfile
      - name: Install webview dependencies
        run: yarn install --frozen-lockfile
        working-directory: ./src/webview
      - name: Release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          VSCE_TOKEN: ${{ secrets.VSCE_TOKEN }}
        run: npx semantic-release
```
This needs a ```VSCE_TOKEN``` secret defined in github/repo/settings/secrets.
