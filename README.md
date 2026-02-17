# Puppet Github Actions

These are [reusable workflows](https://docs.github.com/en/actions/learn-github-actions/reusing-workflows) to run Puppet tests within Github Actions.

They are heavily focused on the workflow that Vox Pupuli uses, which rely on rake tasks from [puppetlabs_spec_helper](https://github.com/puppetlabs/puppetlabs_spec_helper). The tasks executed are:

* `validate`
* `lint`
* `check`
* `rubocop` if enabled (default)
* `parallel_spec` in a matrix for every supported Puppet version

For more information, see [GitHub's workflow reuse documentation](https://docs.github.com/en/actions/learn-github-actions/reusing-workflows).

Vox Pupuli uses these workflows to test modules.
You can reuse them for your own modules (as documented in the next section).
But they can also be configured to test modules that are vendored in a controlrepository or a monorepository.
See [Working with a subdirectory](#working-with-a-subdirectory) for details.

## Gemfile integration examples

Below is an annotated example Gemfile which should provide a working environment. It's the minimum needed.

```ruby
source 'https://rubygems.org'

# The development group is intended for developer tooling. CI will never install this.
group :development do
end

# The test group is used for static validations and unit tests in gha-puppet's
# basic and beaker gha-puppet workflows.
group :test do
  # Require the latest Puppet by default unless a specific version was requested
  # CI will typically set it to '~> 7.0' to get 7.x
  gem 'puppet', ENV.fetch('PUPPET_GEM_VERSION', '>= 0'), require: false
  # Needed to build the test matrix based on metadata
  gem 'puppet_metadata', '~> 3.4', require: false
  # Needed for the rake tasks
  gem 'puppetlabs_spec_helper', '>= 2.16.0', '< 7', require: false
  # Rubocop versions are also specific so it's recommended
  # to be precise. Can be turned off via a parameter
  gem 'rubocop', require: false
end

# The system_tests group is used in gha-puppet's beaker workflow.
group :system_tests do
  gem 'beaker', require: false
  gem 'beaker-docker', require: false
  gem 'beaker-rspec', '>= 8.0', require: false
end

# The release group is used in gha-puppet's release workflow
group :release do
  gem 'puppet-blacksmith', '>= 6', '< 8', require: false
end
```

It is recommended to use [voxpupuli-test](https://github.com/voxpupuli/voxpupuli-test),
[voxpupuli-acceptance](https://github.com/voxpupuli/voxpupuli-acceptance), and
[voxpupuli-release](https://github.com/voxpupuli/voxpupuli-release) to manage the dependencies.

```ruby
source 'https://rubygems.org'

# The development group is intended for developer tooling. CI will never install this.
group :development do
end

# The test group is used for static validations and unit tests in gha-puppet's
# basic and beaker gha-puppet workflows.
group :test do
  # Require the latest Puppet by default unless a specific version was requested
  # CI will typically set it to '~> 7.0' to get 7.x
  gem 'puppet', ENV.fetch('PUPPET_GEM_VERSION', '>= 0'), require: false
  # Needed to build the test matrix based on metadata
  gem 'puppet_metadata', '~> 3.2',  require: false
  # metagem that pulls in all further requirements
  gem 'voxpupuli-test', '~> 7.0', require: false
end

# The system_tests group is used in gha-puppet's beaker workflow.
group :system_tests do
  gem 'voxpupuli-acceptance', '~> 2.1', require: false
end

# The release group is used in gha-puppet's release workflow
group :release do
  gem 'voxpupuli-release', '~> 3.0', '>= 3.0.1'
end
```

## Rakefile integration example

This is the most minimal Rakefile you can have and still use all the shared actions.

```ruby
begin
  require 'puppetlabs_spec_helper/rake_tasks'
rescue LoadError
  # Allowed to fail, only needed in test
end

begin
  require 'beaker-rspec/rake_task'
rescue LoadError
  # Allowed to fail, only needed in acceptance
end

begin
  require 'puppet_blacksmith/rake_tasks'
rescue LoadError
  # Allowed to fail, only needed in release
end
```

## Calling test workflows

It is recommended to create a single workflow for all Puppet tests and name it `.github/workflows/puppet.yml`.

### Workflow templates

Starter workflow templates are in [.github/workflow-templates](.github/workflow-templates).

If you are in an organization that maintains an org-level `.github` repository,
you can copy these files there to make them show up in GitHub's “New workflow” UI
for all repositories in the org.
Otherwise, you can copy the contents of the templates
into your own repository's `.github/workflows/` directory.

### Basic tests

There is a workflow defined that runs the basic tests. It does not run acceptance tests.

```yaml
name: CI

on: pull_request

concurrency:
  group: ${{ github.ref_name }}
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
  group: ${{ github.ref_name }}
  cancel-in-progress: true

jobs:
  puppet:
    name: Puppet
    uses: voxpupuli/gha-puppet/.github/workflows/beaker.yml@v1
```

### Install additional packages

The basic and the beaker workflow support the `additional_packages` input string.
You can use that to install additional packages.
The String is passed to `sudo apt-get install -y`

```yaml
jobs:
  puppet:
    name: Puppet
    uses: voxpupuli/gha-puppet/.github/workflows/basic.yml@v1
    with:
      additional_packages: 'libaugeas-dev augeas-tools'
```

## Calling the release prepare workflow

We've one workflow that can create a release PR, `prepare_release.yml`.

It relies on [puppet-blacksmith](https://github.com/voxpupuli/puppet-blacksmith) and [voxpupuli-release](https://github.com/voxpupuli/voxpupuli-release/?tab=readme-ov-file#vox-pupuli-release-gem).

There are a few inputs:

* `allowed_owner` - The workflow only runs if the owner matches. This prevents forks from attempting to release.
* `version` - Optional version that will be used to prepare the release.
* `working-directory` - The working directory where all jobs should be executed.
* `base-branch` - The branch that will be used as the origin for the release branch.

When `version` is not provided, the [module:bump](https://github.com/voxpupuli/puppet-blacksmith/blob/b2d6d41e99c9dde2ab049455d25c66d167f0fa1d/lib/puppet_blacksmith/rake_tasks.rb#L83-L88) rake task will be executed.
This will increase the version in metadata.json to the next patch level.

There is also one secret ([GitHub's secrets documentation](https://docs.github.com/en/actions/security-guides/encrypted-secrets)):

* `github_pat` - A PAT (personal access token) from a bot account. Defaults to `${{ secrets.PCCI_PAT_RELEASE_PREP }}`

Every interaction with the GitHub API needs to be authenticated.
By default, GitHub provides a token to CI jobs.
Those tokens cannot trigger another CI run.
This is a feature to prevent loops.
Since our goal is to create a "release PR", and we want to run the CI jobs for this PR, we need another token.
To do so, Vox Pupuli has a bot account, [pccibot](https://github.com/pccibot).
A 'fine grained access token' from such an account needs to be provided to to `github_pat` secret.

The token needs permissions on the correct GitHub namespace, the correct repository and:

* Read and Write access to code and pull requests
* Read access to metadata

An organisation admin needs to approve the requested token (settings -> Personal access tokens -> Pending requests).

## Calling the release workflow

The release workflow relies on [puppet-blacksmith](https://github.com/voxpupuli/puppet-blacksmith) and in particular the `module:push` rake task. It also uses the [gh cli](https://cli.github.com/).

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

## Working with a subdirectory

Assume you've a controlrepository or a monorepository:

```text
.
├── bin
│   └── config_script.sh
├── data
│   └── nodes
├── environment.conf
├── hiera.yaml
├── LICENSE
├── manifests
│   └── site.pp
├── Puppetfile
├── README.md
└── site
    └── profiles
        ├── files
        ├── Gemfile
        ├── manifests
        ├── metadata.json
        ├── Rakefile
        ├── REFERENCE.md
        └── templates
```

You can use our workflow for the vendored module (in this example `profiles`) as well.
They all support a `working-directory` input that you can set to the vendored module:

```yaml
name: CI

on: pull_request

concurrency:
  group: ${{ github.ref_name }}
  cancel-in-progress: true

jobs:
  puppet:
    name: Puppet
    uses: voxpupuli/gha-puppet/.github/workflows/beaker.yml@v1
    working-directory: ./site/profiles
```
