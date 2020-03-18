# autotag-action

A lightning fast autotagger for `semver` tagging.

This action has been inspired by [anothrNick/github-tag-action](/anothrNick/github-tag-action), but is written completely in javascript and runs directly within the runner.

`autotag-action` uses [`Octokit`](https://octokit.github.io/rest.js/v17) for tagging and does not depend on checking out the repository. 

## Inputs

### `github-token`

**Required** The github token for accessing the repository.

### `dry-run`

**Optional**  If the value is not `FALSE` (case insensitive), the action performs all steps, but omit the actual tagging. (Default: `FALSE`).

This input is useful if the release version has to be known for other steps before actually tagging the final commit. It is very handy if a build or cleanup steps will extend the initial commit or merge request. 

### `bump`

**Optional** semver bumping. Valid values are `major`, `minor` or `patch` (Default: `minor`)

### `with-v`

**Optional** If not `FALSE` (case insensitive), then the action adds a `v` to prefix the tag. (Default: `FALSE`)

### `branch`

**Optional** Sets a branch to perform the action on. This is useful if the triggering action's branch is different to the final action. 

This input is useful if subsequent steps manipulate a different branch, which should get tagged. This is useful when tagging of merge requests. 

### `release-branch`

**Optional** a comma-separated list of branch names or regular expressions. (Default: `master`)

## Outputs

### `tag`

The previous latest tag before this action ran.

### `new-tag`

The latest tag after this action ran.

## Example Usage

Minimal call

```
name: Auto.Tag

on: 
- push

jobs:
  tag:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: 
        - 12
    steps: 
    - uses: phish108/autotag-action@v1
      with:
        github-token: ${{ secrets.GITHUB_TOKEN}}
```

More advanced call as part of a test pipeline.

```
name: Node.CI

on: 
- push

jobs:
  build:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: 
        - 12

    steps:
    - run: echo 00010
    - uses: actions/checkout@v2
    - name: Use Node.js ${{ matrix.node-version }}
      uses: actions/setup-node@v1
      with:
        node-version: ${{ matrix.node-version }}
    - run: npm ci
    - run: npm test
      env:
        CI: "true"
    - uses: phish108/autotag-action@v1
      with:
        github-token: ${{ secrets.GITHUB_TOKEN}}
        with-v: "true"
```

more complex scenario in merge requests

```
name: Pull Request CI

on: 
  pull_request:
    branches: 
      - master

jobs:
  lint:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: 
        - 12.x

    steps:
    - uses: actions/checkout@v2
    - uses: actions/setup-node@v1
      with:
        node-version: ${{ matrix.node-version }}
    - run: npm ci
    - run: npm run lint

  build:
    needs: lint
    runs-on: ${{ matrix.os }}

    strategy:
      matrix:
        node-version: 
        - 12.x
        - 13.x
        os:
        - ubuntu-latest
        - macos-latest
        - windows-latest

    steps:
    - uses: actions/checkout@v2
    - name: Use Node.js ${{ matrix.node-version }}
      uses: actions/setup-node@v1
      with:
        node-version: ${{ matrix.node-version }}
    - run: npm ci
    - run: npm test
      env:
        CI: true

  merge: 
    needs: build
    if: github.actor == 'phish108' ||  startsWith(github.actor, 'dependabot')
    
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v2
    - name: automerge
      uses: "pascalgn/automerge-action@5ad9f38505afff96c6ad2d1c1bf2775135a7d309"
      env:
        GITHUB_TOKEN: "${{ secrets.GITHUB_TOKEN }}"
        MERGE_LABELS: ""

  release: 
    needs: merge
    if: github.actor == 'phish108' || startsWith(github.actor, 'dependabot')

    runs-on: ubuntu-latest

    steps:
      - id: contributor 
        run: echo ::set-output name=release::minor
        if: github.actor == 'phish108'

      - id: bot
        run: echo ::set-output name=release::patch
        if: startsWith(github.actor, 'dependabot')

      - uses: actions/checkout@v2
        with:
          ref: master

      - run: |
          git config --local user.email "action@github.com"
          git config --local user.name "GitHub Action"
  
      - uses: phish108/autotag-action@v1
        id: tagger
        env:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          bump: ${{ steps.contributor.outputs.release || steps.bot.outputs.release }}
          dry-run: 'true'

      - run: npm --no-git-tag-version --allow-same-version version ${{ steps.tagger.outputs.new_tag }}

      - run: git commit -m "version bump to $NPM_VERSION" -a
        env: 
          NPM_VERSION: ${{ steps.tagger.outputs.new_tag }}

      - name: Push changes
        uses: ad-m/github-push-action@master
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}

      - uses: phish108/github-tag-action@v1
        env:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          branch: "master"
          bump: ${{ steps.contributor.outputs.release || steps.bot.outputs.release }}
          with-v: true
```
