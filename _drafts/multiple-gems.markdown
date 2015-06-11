---
layout: post
title: "Multiple Gems"
---
Releasing multiple gems from the same repository.

Maybe not totally finalized yet, will update if changes happen.

Based on this [blog post on the same
subject](http://opensoul.org/2012/05/30/releasing-multiple-gems-from-one-repository/), but more complicated due
to C extensions, rspec.

Core gem is called nmatrix, plugin gems are called nmatrix-atlas,
nmatrix-xxx, as per [these
conventions](http://guides.rubygems.org/name-your-gem/).

Discuss common header files.

Need to wrap code.

### Repository directory structure

```
ext/nmatrix/ - common header files live here in addition to C files. How to
use common header files?
ext/nmatrix\_atlas/
lib/nmatrix.rb - main file for nmatrix
lib/nmatrix/ auxillary ruby files
lib/nmatrix/atlas.rb main file for nmatrix-atlas, so the extension can be
loaded with `require 'nmatrix/atlas'` again according
to the convention.
spec/
spec/plugins/atlas/
```

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
  gem.test_files    = `git ls-files -- spec/plugins/atlas`.split("\n")
  gem.extensions = ['ext/nmatrix_atlas/extconf.rb']
  gem.require_paths = ["lib"]

  gem.required_ruby_version = '>= 1.9'

  gem.add_dependency 'nmatrix', NMatrix::VERSION::STRING
end
```

The important thing is making sure we add all the needed files to `gem.files`:
all the needed ruby files (here we have only one), all of the files from
`ext/nmatrix_atlas` and the common header files that we will need to build
the extension when the user runs `gem install`. Setting `gem.test_files`
(doesn't actually do
anything)[https://stackoverflow.com/questions/18871541/what-is-the-purpose-of-test-files-configuration-in-a-gemspec]
from the perspective of RubyGems, but we will make use of it when setting up the
spec task in our Rakefile.

Then of course we need to make our plugin gem dependent on the core nmatrix gem.

### Core gemspec

The original post had a neat trick for adding all files to
the main gem, except for those that were added to plugin gems, but it's a
little more complicated that us since we have shared header files that we
want to be installed with all gems:

```ruby
lib = File.expand_path('../lib/', __FILE__)
$:.unshift lib unless $:.include?(lib)

require 'nmatrix/version'

#get files that are used by plugins rather than the main nmatrix gem
plugin_files = []
plugin_test_files = []
Dir["nmatrix-*.gemspec"].each do |gemspec_file|
  gemspec = eval(File.read(gemspec_file))
  plugin_files += gemspec.files
  plugin_test_files += gemspec.test_files
end

Gem::Specification.new do |gem|
  gem.name = "nmatrix"
  gem.version = NMatrix::VERSION::STRING

  # [...] boring stuff goes here

  gem.files         = `git ls-files`.split("\n") - plugin_files
  gem.files         += `git ls-files -- ext/nmatrix`.split("\n") #need to explicitly add this, since some of these files are included in plugin_files
  gem.files.uniq!
  gem.test_files    = `git ls-files -- spec`.split("\n") - plugin_test_files
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

The Rakefile is where most of the magic goes down. You can pass arguments to
rake by calling the command `rake task arg1=val1`. This sets an environment
variable `arg1` that you can access from within rake. I use the
`nmatrix_plugin` env variable to specify which plugins rake should
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

So we create a `Rake::ExtensionTask` for each C extension that we have:

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

and a `Gem::PackageTask` for each gemspec:

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

(note that [`Bundler::GemHelper.install_tasks` does not work with multiple
gems in the same directory.](https://github.com/bundler/bundler/issues/2971))

The `spec` task for using RSpec was a bit more complicated so I think I'll
save that for another post.
