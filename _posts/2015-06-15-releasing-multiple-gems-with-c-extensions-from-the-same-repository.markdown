---
layout: post
title: "Releasing multiple gems (with C extensions) from the same repository"
date: 2015-06-15T19:36:11-04:00
---
UPDATE 2015-06-26: Change gem.files and gem.test_files in gemspecs

Currently nmatrix relies on the ATLAS library, which can be a big pain to
install.
We want people to be able to build and use nmatrix without having to install
ATLAS, but at the same time allow those who do have ATLAS to use it as
before. This means separating nmatrix into two gems: the core one `nmatrix`
and the plugin `nmatrix-atlas`. Eventually there may be other nmatrix plugins
`nmatrix-xxx` (see [ruby naming conventions](http://guides.rubygems.org/name-your-gem/)).
We would like to develop both gems in the same repository so that
development of the two gems will stay in sync.

I found a quite useful [blog post on the same
subject](http://opensoul.org/2012/05/30/releasing-multiple-gems-from-one-repository/), 
which is what this post is based on, however the nmatrix case is a bit more
complicated since both the core gem and the plugin rely on C code.

The design is maybe not totally finalized yet, I will update if changes happen.

### Repository directory structure

```
ext/nmatrix/          nmatrix C extension
ext/nmatrix_atlas/    nmatrix-atlas C extension
lib/nmatrix.rb        main file for nmatrix
lib/nmatrix/          auxillary ruby files
lib/nmatrix/atlas.rb  main file for nmatrix-atlas, so the extension can be
                        loaded with `require 'nmatrix/atlas'` as required
                        by the naming convention
spec/                 shared tests, tests for nmatrix, auxillary files for tests
spec/plugins/atlas/   tests for nmatrix-atlas
```

Common header files which are needed by both C extensions are located in
`ext/nmatrix`. How should `nmatrix_atlas` get access to these common header
files? I thought of three ways to do this:

1. Install the nmatrix gem and use the installed header files to build
nmatrix-atlas. This is a bad
solution because it makes the build process clunky for developers.
You would have to
`gem install nmatrix` before you could even build nmatrix-atlas.

2. Just add `ext/nmatrix` to the include path for the nmatrix-atlas
build process, and then package all the necessary headers with the
nmatrix-atlas gem. This has the downside that you will end up
packaging identical header files twice in two (or more) different
gems.

3. Deal with the issue differently depending on whether we are
building in a development environment or building during a gem
installation. In the first case, use the headers from `ext/nmatrix`
in the tree, in the second case use the headers from the installed
nmatrix gem. This would mean that the headers would only need to be
packaged in one place, but it would add complexity to the build
process.

I chose to go with option 2.

### Plugin gemspecs

First we need to make a gemspec for each gem: `nmatrix.gemspec`,
`nmatrix-atlas.gemspec`. What to put in these gemspecs?

For `nmatrix-atlas.gemspec`, it looks like a normal gemspec except for a few
interesting things:

```ruby
lib = File.expand_path('../lib/', __FILE__)
$:.unshift lib unless $:.include?(lib)

require 'nmatrix/version'

Gem::Specification.new do |gem|
  gem.name = "nmatrix-atlas"
  gem.version = NMatrix::VERSION::STRING #use the same version as the main gem

  # [...] some boring stuff goes here

  gem.files         = ["lib/nmatrix/atlas.rb"]
  gem.files         += `git ls-files -- ext/nmatrix_atlas`.split("\n")
  gem.files         += `git ls-files -- ext/nmatrix | grep ".h$"`.split("\n") #need nmatrix header files to compile
  gem.test_files    = `git ls-files -- spec`.split("\n")
  gem.test_files    -= `git ls-files -- spec/plugins`.split("\n")
  gem.test_files    += `git ls-files -- spec/plugins/atlas`.split("\n")
  gem.extensions = ['ext/nmatrix_atlas/extconf.rb']
  gem.require_paths = ["lib"]

  gem.required_ruby_version = '>= 1.9'

  gem.add_dependency 'nmatrix', NMatrix::VERSION::STRING
end
```

The important thing is making sure we add all the needed files to `gem.files`:
all the needed ruby files (here we have only one), all of the files from
`ext/nmatrix_atlas`, and the common header files that we will need to build
the extension when the user runs `gem install`. Setting `gem.test_files`
[doesn't actually do anything](https://stackoverflow.com/questions/18871541/what-is-the-purpose-of-test-files-configuration-in-a-gemspec)
from the perspective of RubyGems, but we will make use of it when setting up the
spec task in our Rakefile.

Then of course we need to make our plugin gem dependent on the core nmatrix gem.

### Core gemspec

The original post had a neat trick for adding all files to
the main gem, except for those that were added to plugin gems, but it's a
little more complicated for us since we have shared header files that we
want to be installed with all gems:

```ruby
lib = File.expand_path('../lib/', __FILE__)
$:.unshift lib unless $:.include?(lib)

require 'nmatrix/version'

#get files that are used by plugins rather than the main nmatrix gem
plugin_files = []
Dir["nmatrix-*.gemspec"].each do |gemspec_file|
  gemspec = eval(File.read(gemspec_file))
  plugin_files += gemspec.files
end
plugin_lib_files = plugin_files.select { |file| file.match(/^lib\//) }

Gem::Specification.new do |gem|
  gem.name = "nmatrix"
  gem.version = NMatrix::VERSION::STRING

  # [...] boring stuff goes here

  gem.files         = `git ls-files -- ext/nmatrix`.split("\n")
  gem.files         += `git ls-files -- lib`.split("\n")
  gem.files         -= plugin_lib_files
  gem.test_files    = `git ls-files -- spec`.split("\n")
  gem.test_files    -= `git ls-files -- spec/plugins`.split("\n")
  gem.extensions = ['ext/nmatrix/extconf.rb']
  gem.require_paths = ["lib"]

  gem.required_ruby_version = '>= 1.9'

  # add ordinary dependencies here
end
```

### Gemfile

This is pretty much the same as in the original post:

```ruby
source 'https://rubygems.org'

#main gemspec
gemspec :name => 'nmatrix'

#plugin gemspecs
Dir['nmatrix-*.gemspec'].each do |gemspec_file|
  plugin_name = gemspec_file.match(/(nmatrix-.*)\.gemspec/)[1]
  gemspec(:name => plugin_name, :development_group => :plugin)
end
```

### Rakefile

The Rakefile is where most of the magic goes down.

I think in the original post the author was assuming that a developer would
always want to build all the plugins. But for us since `nmatrix-atlas` has a
nasty dependency, we don't want to force everyone to build it. So we pass an
argument to rake to tell it what plugins we want. You can pass arguments to
rake by calling the command `rake task arg1=val1`. This sets an environment
variable `arg1` that you can access from within rake. I use the
`nmatrix_plugins` env variable to specify which plugins rake should
build/package/test/whatever:

```ruby
#Specify plugins to build on the command line like:
#rake whatever nmatrix_plugins=atlas,lapack
#or
#rake whatever nmatrix_plugins=all
#If you want to build *only* plugins and not the core nmatrix gem:
#rake whatever nmatrix_plugins=all nmatrix_core=false
if ENV["nmatrix_plugins"] == "all"
  gemspecs = Dir["*.gemspec"]
else
  plugins = []
  plugins = ENV["nmatrix_plugins"].split(",") if ENV["nmatrix_plugins"]
  gemspecs = ["nmatrix.gemspec"] #always include the main nmatrix gem
  plugins.each do |plugin|
    gemspecs << "nmatrix-#{plugin}.gemspec"
  end
end
if ENV["nmatrix_core"] == "false"
  gemspecs -= ["nmatrix.gemspec"]
end
gemspecs.map! { |gemspec| eval(IO.read(gemspec)) }
```

So now `gemspecs` is an array of all the gemspecs that we want rake to deal
with.

So we create a `Rake::ExtensionTask`
for each C extension that we have, which makes a single `compile` task
that compiles all of the extensions:

```ruby
require 'rake'
require "rake/extensiontask"

gemspecs.each do |gemspec|
  next unless gemspec.extensions
  gemspec.extensions.each do |extconf|
    ext_name = extconf.match(/ext\/(.*)\/extconf\.rb/)[1]
    Rake::ExtensionTask.new do |ext|
      ext.name = ext_name
      ext.ext_dir = "ext/#{ext_name}"
      ext.lib_dir = 'lib/'
      ext.source_pattern = "**/*.{c,cpp,h}"
    end
  end
end
```

and a `Gem::PackageTask` for each gemspec, which gives us `package` and
`repackage` tasks:

```ruby
gemspecs.each do |gemspec|
  Gem::PackageTask.new(gemspec).define
end
```

and a custom `install` task:

```ruby
desc "Build and install into system gems."
task :install => :package do
  gemspecs.each do |gemspec|
    gem_file = "pkg/#{gemspec.name}-#{gemspec.version}.gem"
    system "gem install '#{gem_file}'"
  end
end
```

Note that [`Bundler::GemHelper.install_tasks` does not work with multiple
gems in the same directory](https://github.com/bundler/bundler/issues/2971).

The `spec` task for using RSpec was a bit more complicated so I split that
off into a [different blog post]({% post_url 2015-06-22-rspec-tasks-for-nmatrix-plugins %}).

I still haven't tried this with travis-cl, maybe that will require a little
bit more work.
