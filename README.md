# Puppet Github Actions

These are [reusable workflows](https://docs.github.com/en/actions/learn-github-actions/reusing-workflows) to run Puppet tests within Github Actions.

They are heavily focused on the workflow that Vox Pupuli uses, which rely on rake tasks from [puppetlabs_spec_helper](https://github.com/puppetlabs/puppetlabs_spec_helper). The tasks executed are:

* `validate`
* `lint`
* `check`
* `rubocop` if enabled (default)
* `parallel_spec` in a matrix for every supported Puppet version

For more information, see [GitHub's workflow reuse documentation](https://docs.github.com/en/actions/learn-github-actions/reusing-workflows).

## Calling test workflows

It is recommended to create a single workflow for all Puppet tests and name it `.github/workflows/puppet.yml`.

### Basic tests

There is a workflow defined that runs the basic tests. It does not run acceptance tests.

```yaml
name: CI

on: pull_request

concurrency:
  group: ${{ github.head_ref }}
  cancel-in-progress: true

jobs:
  puppet:
    name: Puppet
    uses: voxpupuli/gha-puppet/.github/workflows/basic.yml@v1
```

To disable Rubocop, modify the job:

```yaml
jobs:
  puppet:
    name: Puppet
    uses: voxpupuli/gha-puppet/.github/workflows/basic.yml@v1
    with:
      rubocop: false
```

### Beaker tests

For modules using [beaker](https://github.com/voxpupuli/beaker) for acceptance testing there is a workflow which also includes the basic tests. Like basic tests, it accepts a `rubocop` parameter.

```yaml
name: CI

on: pull_request

concurrency:
  group: ${{ github.head_ref }}
  cancel-in-progress: true

jobs:
  puppet:
    name: Puppet
    uses: voxpupuli/gha-puppet/.github/workflows/beaker.yml@v1
```

## Calling the release workflow

The release workflow relies on [puppet-blacksmith](https://github.com/voxpupuli/puppet-blacksmith) and in particular the `module:push` rake task.

There is one input:
* `allowed_owner` - The workflow only runs if the owner matches. This prevents forks from attempting to release.

There are multiple secrets ([GitHub's secrets documentation](https://docs.github.com/en/actions/security-guides/encrypted-secrets)):
* `username` - The Puppet Forge username
* `api_key` - The [Puppet Forge API key](https://forgeapi.puppet.com/#section/Authentication/ApiKeyAuth)

It is recommended to create a workflow `.github/workflows/release.yml`.

```yaml
name: Release

on:
  push:
    tags:
      - '*'

jobs:
  deploy:
    name: Deploy
    uses: voxpupuli/gha-puppet/.github/workflows/release.yml@v1
    with:
      allowed_owner: MY_GITHUB_USERNAME
    secrets:
      username: ${{ secrets.PUPPET_FORGE_USERNAME }}
      api_key: ${{ secrets.PUPPET_FORGE_API_KEY }}
```
