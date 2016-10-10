kitchen-in-travis-native Cookbook [![Build Status](https://travis-ci.org/zuazo/kitchen-in-travis-native.svg?branch=master)](https://travis-ci.org/zuazo/kitchen-in-travis-native)
=================================

Proof of concept cookbook to run [test-kitchen](http://kitchen.ci/) inside [Travis CI](https://travis-ci.org/) using the [native Docker service](http://blog.travis-ci.com/2015-08-19-using-docker-on-travis-ci/) and [kitchen-docker](https://github.com/portertech/kitchen-docker).

You can use this in your cookbook by using a *.travis.yml* file similar to the following:

```yaml
rvm: 2.2

sudo: required
services: docker

env:
  matrix:
  - INSTANCE=default-ubuntu-1404
  - INSTANCE=default-centos-66

before_install: curl -L https://www.getchef.com/chef/install.sh | sudo bash -s -- -P chefdk -v 0.18.30

install: chef exec bundle install

# https://github.com/zuazo/kitchen-in-travis-native/issues/1#issuecomment-142455888
before_script: sudo iptables -L DOCKER || sudo iptables -N DOCKER

script:
# Run test-kitchen with docker driver, for example:
- KITCHEN_LOCAL_YAML=.kitchen.docker.yml chef exec bundle exec kitchen verify ${INSTANCE}
```

Look [below](https://github.com/zuazo/kitchen-in-travis-native#how-to-implement-this-in-my-cookbook) for more complete examples.

The following files will help you understand how this works:

* [*.travis.yml*](https://github.com/zuazo/kitchen-in-travis-native/blob/master/.travis.yml)
* [*.kitchen.docker.yml*](https://github.com/zuazo/kitchen-in-travis-native/blob/master/.kitchen.docker.yml)
* [*Rakefile*](https://github.com/zuazo/kitchen-in-travis-native/blob/master/Rakefile)

This example cookbook only installs nginx. It also includes some [Serverspec](http://serverspec.org/) tests to check everything is working correctly.

## Related Projects

* [kitchen-in-travis](https://github.com/zuazo/kitchen-in-travis): Runs test-kitchen inside [Travis CI](https://travis-ci.org/) using [User Mode Linux](https://github.com/jpetazzo/sekexe), without using the new native Docker service. The build times are longer but more customizable. Recommended if you want to run tests against many instances. For example, to test multiple instances for each build.
* [kitchen-in-circleci](https://github.com/zuazo/kitchen-in-circleci): Runs test-kitchen inside [CircleCI](https://circleci.com/).

## Install the Requirements

First you need to install [Docker](https://docs.docker.com/installation/).

Then you can use [bundler](http://bundler.io/) to install the required ruby gems:

    $ gem install bundle
    $ bundle install

## Running the Tests in Your Workstation

    $ bundle exec rake

This example will run kitchen **with Vagrant** in your workstation. You can use `$ bundle exec rake integration:docker[default-ubuntu-1404]` to run kitchen with Docker, as in Travis CI.

## Available Rake Tasks

    $ bundle exec rake -T
    rake integration:docker[instance]  # Run integration tests with kitchen-docker
    rake integration:vagrant           # Run integration tests with kitchen-vagrant

## How to Implement This in My Cookbook

First, create a `.kitchen.docker.yml` file with the platforms you want to test:

```yaml
---
driver:
  name: docker
  privileged: true

platforms:
- name: centos-6.6
- name: ubuntu-14.04
  run_list: recipe[apt]
# [...]
```

If not defined, it will get the platforms from the main `.kitchen.yml` by default.

You can get the list of the platforms officially supported by Docker [here](https://hub.docker.com/explore/).

Then, I recommend you to create a task in your *Rakefile*:

```ruby
# Rakefile
require 'bundler/setup'

# [...]

desc 'Run Test Kitchen integration tests'
namespace :integration do
  desc 'Run integration tests with kitchen-docker'
  task :docker, [:instance] do |_t, args|
    args.with_defaults(instance: 'default-ubuntu-1404')
    require 'kitchen'
    Kitchen.logger = Kitchen.default_file_logger
    loader = Kitchen::Loader::YAML.new(local_config: '.kitchen.docker.yml')
    instances = Kitchen::Config.new(loader: loader).instances
    # Travis CI Docker service does not support destroy:
    instances.get(args.instance).verify
  end
end
```

This will allow us to use `$ bundle exec rake integration:docker[INSTANCE]` to run tests against an instance. If you want more elaborate rake tasks, [see the `kitchen-in-travis` example](https://github.com/zuazo/kitchen-in-travis#how-to-run-tests-in-many-platforms).

The *.travis.yml* file example:

```yaml
rvm: 2.2

sudo: required
services: docker

env:
  matrix:
  - INSTANCE=default-ubuntu-1404
  - INSTANCE=default-centos-66

before_install: curl -L https://www.getchef.com/chef/install.sh | sudo bash -s -- -P chefdk -v 0.18.30

install: chef exec bundle install --jobs=3 --retry=3

# https://github.com/zuazo/kitchen-in-travis-native/issues/1#issuecomment-142455888
before_script: sudo iptables -L DOCKER || sudo iptables -N DOCKER

script: travis_retry chef exec bundle exec rake integration:docker[${INSTANCE}]
```

If you are using a *Gemfile*, you should add the following to it:

```ruby
# Gemfile

gem 'berkshelf', '~> 5.1' # Comes with ChefDK 0.18.30

group :integration do
  gem 'test-kitchen', '~> 1.13'
end

group :docker do
  gem 'kitchen-docker', '~> 2.6'
end
```

## Real-world Examples

* [supermarket-omnibus](https://github.com/chef-cookbooks/supermarket-omnibus-cookbook) cookbook ([*.travis.yml*](https://github.com/chef-cookbooks/supermarket-omnibus-cookbook/blob/master/.travis.yml), [*.kitchen.docker.yml*](https://github.com/chef-cookbooks/supermarket-omnibus-cookbook/blob/master/.kitchen.docker.yml).

* [owncloud](https://github.com/zuazo/owncloud-cookbook) cookbook ([*.travis.yml*](https://github.com/zuazo/owncloud-cookbook/blob/master/.travis.yml), [*.kitchen.docker.yml*](https://github.com/zuazo/owncloud-cookbook/blob/master/.kitchen.docker.yml), [*Rakefile*](https://github.com/zuazo/owncloud-cookbook/blob/master/Rakefile)): Runs kitchen tests against many platforms. Includes Serverspec tests using [infrataster](https://github.com/ryotarai/infrataster).

* [mysql_tuning](https://github.com/zuazo/mysql_tuning-cookbook) cookbook ([*.travis.yml*](https://github.com/zuazo/mysql_tuning-cookbook/blob/master/.travis.yml), [*.kitchen.docker.yml*](https://github.com/zuazo/mysql_tuning-cookbook/blob/master/.kitchen.docker.yml), [*Rakefile*](https://github.com/zuazo/mysql_tuning-cookbook/blob/master/Rakefile)): Runs kitchen tests against many platforms. Includes Serverspec tests.

## Known Issues

### Privileged Containers

It's recommended to run the containers in privileged mode to avoid some weird errors when starting system services or when running Serverspec tests.

```yaml
---
driver:
  name: docker
  privileged: true
```

### The Test Cannot Exceed 50 Minutes

Each test can not take more than 50 minutes to run within Travis CI.

### Cannot Destroy Containers

Containers inside Travis CI Docker service can not be destroyed, so we need to `$ kitchen verify` instead of `$ kitchen test` to run the tests.

The Travis build error output:

```
Kitchen::ActionFailed: Failed to complete #destroy action: [Expected process to exit with [0], but received '1'
---- Begin output of sudo -E docker -H unix:///var/run/docker.sock stop 1a92da7 ----
STDOUT:
STDERR: Error response from daemon: Cannot stop container 1a92da7: [8] System error: permission denied
Error: failed to stop containers: [1a92da7]
---- End output of sudo -E docker -H unix:///var/run/docker.sock stop 1a92da7 ----
```

### Only One Instance for Each Build

As [containers can not be destroyed](#cannot-destroy-containers), you should run one instance for each build. So things like running all Ubuntu tests in a single build is not recommended.

Look at the examples in this documentation to learn how to do this.

### `bundle install` Error: *No output has been received in the last 10m*

This is a Travis build example output:

```
$ chef exec bundle install
[...]
Installing dep-selector-libgecode 1.0.2 with native extensions

No output has been received in the last 10m0s, this potentially indicates a stalled build or something wrong with the build itself.
```

This is because, for some strange reason, the compilation of certain gems takes too long inside Travis Docker builds. To avoid this error you can install a specific version of ChefDK and use the gems that come with it. This avoids the installation of some heavyweight gems like Berkshelf. For this, you need include in your *Gemfile* the same version of Berkshelf that comes with ChefDK.

For example:

```yaml
# .travis.yml
before_install: curl -L https://www.getchef.com/chef/install.sh | sudo bash -s -- -P chefdk -v 0.18.30
```

```ruby
# Gemfile
gem 'berkshelf', '~> 5.1' # Comes with ChefDK 0.18.30
```

The same applies for other gems you have in your Gemfile: Use the version that comes with ChefDK if possible. If you need gems that conflict with ChefDK, try [this alternatives](#related-projects).

If the error is not due to gems, but a command that can take a long time to run and is very quiet, you may need to run it with some flags to increase verbosity such as: `--verbose`, `--debug`, `--l debug`, ...

### Official CentOS 7 and Fedora Images

Cookbooks requiring [systemd](http://www.freedesktop.org/wiki/Software/systemd/) may not work correctly on CentOS 7 and Fedora containers. See [*Systemd removed in CentOS 7*](https://github.com/docker-library/docs/tree/master/centos#systemd-integration).

You can use alternative images that include systemd. These containers must run in **privileged** mode:

```yaml
# .kitchen.docker.yml

# Non-official images with systemd
- name: centos-7
  driver_config:
    # https://registry.hub.docker.com/u/milcom/centos7-systemd/dockerfile/
    image: milcom/centos7-systemd
    privileged: true
- name: fedora
  driver_config:
    image: fedora/systemd-systemd
    privileged: true
```

### Problems with Upstart in Ubuntu

Some cookbooks requiring [Ubuntu Upstart](http://upstart.ubuntu.com/) may not work correctly.

You can use the official Ubuntu images with Upstart enabled:

```yaml
# .kichen.docker.yml

- name: ubuntu-14.10
  run_list: recipe[apt]
  driver_config:
    image: ubuntu-upstart:14.10
```

### Install `netstat` Package

It's recommended to install `net-tools` on some containers if you want to test listening ports with Serverspec. This is because some images come without `netstat` installed.

This is required for example for the following Serverspec test:

```ruby
# test/integration/default/serverspec/default_spec.rb
describe port(80) do
  it { should be_listening }
end
```

You can ensure that `netstat` is properly installed running the [`netstat`](https://supermarket.chef.io/cookbooks/netstat) cookbook:

 ```yaml
# .kitchen.docker.yml

- name: debian-6
  run_list:
  - recipe[apt]
  - recipe[netstat]
```

## Feedback Is Welcome

Currently I'm using this for my own projects. It may not work correctly in many cases. If you use this or a similar approach successfully with other cookbooks, please [open an issue and let me know about your experience](https://github.com/zuazo/kitchen-in-travis-native/issues/new). Problems, discussions and ideas for improvement, of course, are also welcome.

## Acknowledgements

Special thanks to [Jonathan Hartman](https://github.com/RoboticCheese) for his work in the [`test-kitchen-test-chef`](https://github.com/RoboticCheese/test-kitchen-test-chef) cookbook example.

See [here](https://github.com/zuazo/docker-in-travis#acknowledgements) for more.

# License and Author

|                      |                                          |
|:---------------------|:-----------------------------------------|
| **Author:**          | [Xabier de Zuazo](https://github.com/zuazo) (<xabier@zuazo.org>)
| **Contributor:**     | [Irving Popovetsky](https://github.com/irvingpop)
| **Contributor:**     | [Tim Smith](https://github.com/tas50)
| **Copyright:**       | Copyright (c) 2015, Xabier de Zuazo
| **License:**         | Apache License, Version 2.0

    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at

        http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.
