#!/usr/bin/env rake
# encoding: utf-8
# 3p
require 'rake/clean'
require 'rubocop/rake_task'

# Flavored Travis CI jobs
require './ci/checks_mock'
require './ci/core_integration'
require './ci/default'
require './ci/system'
require './ci/windows'
require './ci/docker_daemon'

CLOBBER.include '**/*.pyc'

# CI-like environment for local use
unless ENV['CI']
  rakefile_dir = File.dirname(__FILE__)
  ENV['TRAVIS_BUILD_DIR'] = rakefile_dir
  # Commented out to make 'rake run' work with integrations in ../integrations and
  # configuration in ./confi.d/*.yaml
  # ENV['INTEGRATIONS_DIR'] = File.join(rakefile_dir, 'embedded')
  ENV['CHECKSD_OVERRIDE'] = File.join(rakefile_dir, 'tests/checks/fixtures/checks')
  ENV['PIP_CACHE'] = File.join(rakefile_dir, '.cache/pip')
  ENV['VOLATILE_DIR'] = '/tmp/dd-agent-testing'
  ENV['CONCURRENCY'] = ENV['CONCURRENCY'] || '2'
  ENV['NOSE_FILTER'] = 'not windows'
  ENV['JMXFETCH_URL'] = "file://" + File.join(rakefile_dir, "binaries")
end

desc 'Setup a development environment for the Agent'
task 'setup_env' do
  `mkdir -p venv`
  `wget -O venv/virtualenv.py https://raw.github.com/pypa/virtualenv/1.11.6/virtualenv.py`
  `python venv/virtualenv.py -p python2 --no-site-packages --no-pip --no-setuptools venv/`
  `wget -O venv/ez_setup.py https://bootstrap.pypa.io/ez_setup.py`
  `venv/bin/python venv/ez_setup.py --version="20.9.0"`
  `wget -O venv/get-pip.py https://bootstrap.pypa.io/get-pip.py`
  `venv/bin/python venv/get-pip.py`
  `venv/bin/pip install -r requirements.txt`
  `venv/bin/pip install -r requirements-test.txt`
  # These deps are not really needed, so we ignore failures
  ENV['PIP_COMMAND'] = 'venv/bin/pip'
  `./utils/pip-allow-failures.sh requirements-opt.txt`
end

desc 'Grab libs'
task 'setup_libs' do
  in_venv = system "python -c \"import sys ; exit(not hasattr(sys, 'real_prefix'))\""
  raise 'Not in dev venv/CI environment - bailing out.' if !in_venv && !ENV['CI']

  jmx_version = `python -c "import config ; print config.JMX_VERSION"`
  jmx_version = jmx_version.delete("\n")
  puts "jmx-fetch version: #{jmx_version}"
  jmx_artifact = "jmxfetch-#{jmx_version}-jar-with-dependencies.jar"
  if File.size?("checks/libs/#{jmx_artifact}")
    puts "Artifact already in place: #{jmx_artifact}"
  else
    # let's use `sh` so we can see on the log if wget fails
    sh "curl -o checks/libs/#{jmx_artifact} #{ENV['JMXFETCH_URL']}/#{jmx_artifact}"
  end
end

namespace :test do
  desc 'Run stsstatsd tests'
  task 'stsstatsd' do
    sh 'nosetests tests/core/test_dogstatsd.py'
  end

  desc 'Run performance tests'
  task 'performance' do
    sh 'nosetests --with-xunit --xunit-file=nosetests-performance.xml tests/core/benchmark*.py'
  end

  desc 'cProfile unit tests (requires \'nose-cprof\')'
  task 'profile' do
    sh 'nosetests --with-cprofile tests/core/benchmark*.py'
  end

  desc 'cProfile tests, then run pstats'
  task 'profile:pstats' => ['test:profile'] do
    sh 'python -m pstats stats.dat'
  end

  desc 'Display test coverage for checks'
  task 'coverage' => 'ci:default:coverage'
end

RuboCop::RakeTask.new(:rubocop) do |t|
  t.patterns = ['ci/**/*.rb', 'Gemfile', 'Rakefile']
end

desc 'Lint the code through pylint'
task 'lint' => ['ci:default:lint'] do
end

desc 'Run the Agent locally'
task 'run' do
  sh('supervisord -n -c supervisord.dev.conf')
end

namespace :ci do
  desc 'Run integration tests'
  task :run, :flavor do |_, args|
    puts 'Assuming you are running these tests locally' unless ENV['TRAVIS']
    flavor = args[:flavor] || ENV['TRAVIS_FLAVOR'] || 'default'
    flavors = flavor.split(',')
    flavors.each { |f| Rake::Task["ci:#{f}:execute"].invoke }
  end
end

task default: ['lint', 'ci:run']
