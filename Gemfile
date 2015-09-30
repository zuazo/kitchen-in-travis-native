# -*- mode: ruby -*-
# vi: set ft=ruby :

source 'https://rubygems.org'

gem 'rake'
gem 'berkshelf', '~> 3.3' # Comes with ChefDK 0.8.0

group :integration do
  gem 'test-kitchen', '~> 1.4'
end

group :vagrant do
  gem 'vagrant-wrapper', '~> 2.0'
  gem 'kitchen-vagrant', '~> 0.18'
end

group :docker do
  gem 'kitchen-docker', '~> 2.1.0'
end
