# Puppet Github Actions

These are [reusable workflows](https://docs.github.com/en/actions/learn-github-actions/reusing-workflows) to run Puppet tests within Github Actions.

They are heavily focused on the workflow that Vox Pupuli uses, which rely on rake tasks from [puppetlabs_spec_helper](https://github.com/puppetlabs/puppetlabs_spec_helper). The tasks executed are:

* `validate`
* `lint`
* `check`
* `rubocop` if enabled (default)
* `parallel_spec` in a matrix for every supported Puppet version

For more information, see [GitHub's workflow reuse documentation](https://docs.github.com/en/actions/learn-github-actions/reusing-workflows).

Vox Pupuli uses these workflows to test modules. You can reuse them for your own
modules (as documented in the next section). But they can also be configured to
test modules that are vendored in a controlrepository or a monorepository. See
[Working with a subdirectory](#Working-with-a-subdirectory) for details.

## Gemfile integration example

Below is an annotated example Gemfile which should provide a working environment. It's the minimum needed.

```ruby
source 'https://rubygems.org'

# The development group is intended for developer tooling. CI will never install this.
group :development do
end

# The test group is used for static validations and unit tests in gha-puppet's
# basic and beaker gha-puppet workflows.
# Consider using https://github.com/voxpupuli/voxpupuli-test
group :test do
  # Require the latest Puppet by default unless a specific version was requested
  # CI will typically set it to '~> 7.0' to get 7.x
  gem 'puppet', ENV['PUPPET_GEM_VERSION'] || '>= 0', require: false
  # Needed to build the test matrix based on metadata
  gem 'puppet_metadata', '~> 1.10',  require: false
  # Needed for the rake tasks
  gem 'puppetlabs_spec_helper', '>= 2.16.0', '< 5', require: false
  # Rubocop versions are also specific so it's recommended
  # to be precise. Can be turned off via a parameter
  gem 'rubocop', require: false
end

# The system_tests group is used in gha-puppet's beaker workflow.
# Consider using https://github.com/voxpupuli/voxpupuli-acceptance
group :system_tests do
  gem 'beaker', require: false
  gem 'beaker-docker', require: false
  gem 'beaker-rspec', require: false
end

# The release group is used in gha-puppet's release workflow
# Consider using https://github.com/voxpupuli/voxpupuli-release
group :release do
  gem 'puppet-blacksmith', '~> 6.0', require: false
end
```

## Rakefile integration example

This is the most minimal Rakefile you can have and still use all the shared
actions.

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

The basic and the beaker workflow support the `additional_packages` input string. You can use that to install additional packages. The String is passed to `sudo apt-get install -y`

```yaml
jobs:
  puppet:
    name: Puppet
    uses: voxpupuli/gha-puppet/.github/workflows/basic.yml@v1
    with:
      additional_packages: 'libaugeas-dev augeas-tools'
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

## Working with a subdirectory

Assume you've a controlrepository or a monorepository:

```
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

You can use our workflow for the vendored module (in this example `profiles`) as
well. They all support a `working-directory` input that you can set to the
vendored module:

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
