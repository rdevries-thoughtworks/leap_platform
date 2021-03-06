require 'puppetlabs_spec_helper/rake_tasks'
require 'puppet-lint/tasks/puppet-lint'
require 'puppet-syntax/tasks/puppet-syntax'
require 'puppet-catalog-test'

# return list of modules, either
# "external" (submodules or subrepos), "custom" (no submodules nor subrepos)
# or all modules
# so we can check each array seperately
def modules_pattern (type)
  external = Array.new
  internal = Array.new
  all = Array.new

  Dir['puppet/modules/*'].sort.each do |m|

    # submodule or subrepo ?
    system("grep -q #{m} .gitmodules 2>/dev/null || test -f #{m}/.gitrepo")
    if $?.exitstatus == 0
      external << m + '/**/*.pp'
    else
      internal << m + '/**/*.pp'
    end
    all << m + '/**/*.pp'
  end

  case type
    when 'external'
      external
    when 'internal'
      internal
    when 'all'
      all
  end
end

exclude_paths = ["**/vendor/**/*", "spec/fixtures/**/*", "pkg/**/*" ]

# redefine lint task so we don't lint submoudules for now
Rake::Task[:lint].clear
PuppetLint::RakeTask.new :lint do |config|
  # only check for custom manifests, not submodules for now
  config.pattern          = modules_pattern('internal')
  config.ignore_paths     = exclude_paths
  config.disable_checks   = ['documentation', '140chars', 'arrow_alignment']
  config.fail_on_warnings = false
end

# rake syntax::* tasks
PuppetSyntax.exclude_paths = exclude_paths
PuppetSyntax.future_parser = true

desc "Validate erb templates"
task :templates do
  Dir['**/templates/**/*.erb'].each do |template|
    sh "erb -P -x -T '-' #{template} | ruby -c" unless template =~ /.*vendor.*/
  end
end

namespace :platform do
  desc "Compile hiera config for test_provider"
  task :provider_compile do
    sh "cd tests/puppet/provider; bundle exec leap compile"
  end
end

PuppetCatalogTest::RakeTask.new('catalog') do |t|
  Rake::Task["platform:provider_compile"].invoke
  t.module_paths = ["puppet/modules"]
  t.manifest_path = File.join("puppet","manifests", "site.pp")
  t.facts = {
    "operatingsystem"           => "Debian",
    "osfamily"                  => "Debian",
    "operatingsystemmajrelease" => "8",
    "debian_release"            => "stable",
    "debian_codename"           => "jessie",
    "lsbdistcodename"           => "jessie",
    "concat_basedir"            => "/var/lib/puppet/concat",
    "interfaces"                => "eth0"
  }

  # crucial option for hiera integration
  t.config_dir = File.join("tests/puppet") # expects hiera.yaml to be included in directory

  # t.parser = "future"
  #t.verbose = true
end


namespace :test do
  # :syntax:templates fails on squirrel, see https://jenkins.leap.se/view/Platform%20Builds/job/platform_citest/115/console
  # but we have our own synax test
  desc "Run all puppet syntax checks required for CI (syntax , validate, templates, spec, lint)"
  task :syntax => [:"syntax:hiera", :"syntax:manifests", :validate, :templates, :spec, :lint]

  desc "Tries to compile the catalog"
  task :catalog => [:catalog]

  #task :all => [:syntax, :catalog]
end

# unfortunatly, we cannot have one taks to rule them all
# because :catalog would conflict with :syntax or :validate:
# rake aborted!
# Puppet::DevError: Attempting to initialize global default settings more than once!
# /home/varac/dev/projects/leap/git/leap_platform/vendor/bundle/ruby/2.3.0/gems/puppet-3.8.7/lib/puppet/settings.rb:261:in `initialize_global_settings'
#desc "Run all platform tests"
#task :test => 'test:all'
