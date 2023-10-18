# Changelog

## 2.0.0 - 2023-10-18

* Beaker: Add a `beaker_facter` parameter
* Make [puppet_metadata](https://github.com/voxpupuli/puppet_metadata) set the Beaker matrix name and env vars
* Use Ruby 3.2 in CI

## 1.1.0 - 2023-10-18

* Make CI runner configureable with a parameter
* Require Puppet 7.x in matrix setup job
* Beaker: Add a `domain` parameter
* Pin Beaker workflow to Ubuntu 20.04 images for EL7 support
* Beaker: Add a `beaker_hypervisor` parameter
* Beaker: Add a `additional_packages` parameter
* Document github.ref_name instead of .head_ref
* Add a `cache-version` parameter
* Add a `working-directory` parameter

## 1.0.0 - 2022-01-27

Initial version
