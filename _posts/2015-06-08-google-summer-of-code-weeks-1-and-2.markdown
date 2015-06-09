---
layout: post
title: "Google Summer of Code, Weeks 1 and 2"
date: 2015-06-08T21:56:21-04:00
---

### What I did
Work on my [atlas\_plugin branch](https://github.com/wlevine/nmatrix/tree/atlas_plugin):

 * Set up a framework to build, test, package, and install multiple gems (with C
 extensions) from the nmatrix repository. This ended up being a lot more
 complicated than described in the reference below, and deserves a blog post 
 of its own, maybe after things are more finalized.
 * Removed all external dependencies (CBLAS, CLAPACK) from the core nmatrix
 gem
 * Created a new gem nmatrix-atlas which should eventually implement all the
 stuff removed from the core gem. For the time being it just implements one
 function (`clapack_getri`) as a test, until the design is fully worked out.
 * [Inventoried BLAS and LAPACK functions in nmatrix](https://github.com/wlevine/nmatrix/wiki/Inventory)

### What I learned
Definitely missing a lot of stuff from this list, since I wasn't really keeping track

 * Some stuff about gemspecs and Gemfiles, but I think I forgot it already
 * In BLAS, a negative stride means access elements in reverse order, starting from the end
 * Ways of passing arguments to rake. What was appropriate for me was using
 `rake whatever arg1=val1`, which sets an environment variable `arg1`, but
 you can also [pass arguments to specific tasks](https://stackoverflow.com/questions/825748/how-do-i-pass-command-line-arguments-to-a-rake-task).
 * `Bundler::GemHelper.install_tasks` vs `Gem::PackageTask`: It turns out
 that the nmatrix Rakefile has two parallel methods for building and
 packaging gems. `Gem::PackageTask` adds the package, and repackage tasks
 (which are the ones listed in the nmatrix docs), while`Bundler::GemHelper.install_tasks` adds build, install, and release
 tasks (which are a secret?). They do basically the same thing, but don't talk
 to each other. Oh, and also  the latter doesn't work with multiple gems. I removed `Bundler::GemHelper.install_tasks`
 and added a custom install task.

### Useful references
 * [Best practices for rspec](http://betterspecs.org/)
 * [Multiple gems from same repository](http://opensoul.org/2012/05/30/releasing-multiple-gems-from-one-repository/)
 * [RSpec shared examples](https://www.relishapp.com/rspec/rspec-core/docs/example-groups/shared-examples)
 * [Nice reference on Ruby C extensions, unfortunately spread across multiple blog posts with no nice table of contents of anything](http://clalance.blogspot.com/2011/01/writing-ruby-extensions-in-c-part-3.html)
 * [Another reference on Ruby C extensions](http://phrogz.net/programmingruby/ext_ruby.html)
 * Unfortunately I didn't keep track of the stuff I read about bundler and rubygems
