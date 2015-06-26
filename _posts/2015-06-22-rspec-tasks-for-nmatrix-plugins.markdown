---
layout: post
title: "RSpec tasks for nmatrix plugins"
date: 2015-06-22T19:46:57-04:00
---
UPDATE 2015-06-26: Now re-tests all specs when testing plugins

We want to be able to test multiple implementations of the same function. For
example, nmatrix has an internal implementation of `getrf` (used for
calculating the determinant, among other things), but
nmatrix-atlas overrides this with its own version of `getrf` which uses
the ATLAS implementation. We want to test both of these.

There's no way to tell rspec: "run this bunch of specs, then `require
"nmatrix/atlas"`, then run this other bunch of specs". It loads all the spec
files first, and then starts running the tests. So if one spec file
`require`s `nmatrix/atlas`, then the ATLAS functions will be available when
running all the specs. To avoid this, we make the `rake spec` task invoke
rspec multiple times. After setting up the Rakefile
as [described here]({% post_url 2015-06-15-releasing-multiple-gems-with-c-extensions-from-the-same-repository %})
(and also remembering to set `test_files` in all our gemspecs), we
need to add the following to set up the `spec` task:

```ruby
require 'rspec/core/rake_task'
require 'rspec/core'
namespace :spec do
  #We need a separate rake task for each plugin, rather than one big task that
  #runs all of the specs. This is because there's no way to tell rspec
  #to run the specs in a certain order with (say) "nmatrix/atlas" require'd
  #for some of the specs, but not for others, without splitting them up like
  #this.
  spec_tasks = []
  gemspecs.each do |gemspec|
    test_files = gemspec.test_files
    test_files.keep_if { |file| file =~ /_spec\.rb$/ }
    next if test_files.empty?
    spec_tasks << gemspec
    RSpec::Core::RakeTask.new(gemspec) do |spec|
      spec.pattern = FileList.new(test_files)
    end
  end
  task :all => spec_tasks
end

task :spec => "spec:all"
```

This has the desired behavior that if one the of spec sub-tasks fails, the
task will fail immediately and the most recent messages on the console will
give you the details about the error. It will not continue with the next
rspec invocation if the previous one fails.

So the first invocation (testing the core nmatrix plugin) includes all the
files in `spec/` excluding those in `spec/plugins/`, and the second invocation
(testing the nmatrix-atlas plugin)
includes all of the files from the first run plus the files in
`spec/plugins/atlas/` where `atlas_spec.rb` is located. In `atlas_spec.rb` we
`require 'nmatrix/atlas'`, so that the second time all tests are run with the
nmatrix-atlas plugin active.

Originally I planned to isolate all the specs that could possibly behave
differently with nmatrix-atlas, and run only these tests when testing
nmatrix-atlas. I eventually decided that it was better just to re-test
everything.

Another issue that could arise is if we'd like to use the exact same specs to test
equivalent functions. This is easily solved with [RSpec's shared examples](https://www.relishapp.com/rspec/rspec-core/docs/example-groups/shared-examples).
So we create a file `spec/lapack_shared.rb` (since it doesn't end in
`_spec.rb` it won't be included by our rake spec task):

```ruby
RSpec.shared_examples "LAPACK shared" do
  #getrf spec goes here
end
```

And then in `spec/plugins/atlas/atlas_spec.rb`:

```ruby
require "./lib/nmatrix/atlas"

require 'lapack_shared'

describe "NMatrix::LAPACK implementation from nmatrix-atlas plugin" do
  include_examples "LAPACK shared"
end
```

And something similar in `spec/lapack_spec.rb` except without the `require
"./lib/nmatrix/atlas"` line. I did this before, but now that the
nmatrix-atlas tests runs the entire nmatrix test suite, it's not necessary
for the moment.
