# Puppet Github Actions

These are [reusable workflows](https://docs.github.com/en/actions/learn-github-actions/reusing-workflows) to run Puppet tests within Github Actions.

They are heavily focused on the workflow that Vox Pupuli uses, which is notably not using the PDK but rather direct rake tasks from [puppetlabs_spec_helper](https://github.com/puppetlabs/puppetlabs_spec_helper). It also uses [beaker](https://github.com/voxpupuli/beaker) for acceptance testing.
