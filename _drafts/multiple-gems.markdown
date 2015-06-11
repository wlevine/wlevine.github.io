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

Directory structure:
ext/nmatrix
ext/nmatrix\_atlas
lib/nmatrix.rb
lib/nmatrix/atlas.rb
spec/
spec/plugins/atlas/

So the extension can be loaded with `require 'nmatrix/atlas'` again according
to the convention.

First we need to make a gemspec for each gem: `nmatrix.gemspec`,
`nmatrix-atlas.gemspec`. What to put in these gemspecs?

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
