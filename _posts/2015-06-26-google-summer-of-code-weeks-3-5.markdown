---
layout: post
title: "Google Summer of Code, Weeks 3-5"
date: 2015-06-26T18:01:36-04:00
---

### What I did

 * Finished adding LAPACK and BLAS functions to nmatrix-atlas plugin, so that
my new nmatrix plus nmatrix-atlas is functionally equivalent to the old
nmatrix. The trickiest part of this was the cases (like matrix
multiplication) where we want to use ATLAS functions when applicable, but
still must call the original nmatrix implementation otherwise.
 * Changed `rake spec` task so that nmatrix-atlas tests run the entire nmatrix test
suite in addition to ATLAS specific tests
 * Miscellaneous bug fixes to BLAS/LAPACK functions in nmatrix
 * [Started trying to implement a general LAPACK backend for nmatrix using
LAPACKE](https://github.com/wlevine/nmatrix/tree/lapack_plugin)
 * [Reviewed some code in agisga's mixed_models gem](https://github.com/agisga/mixed_models/pull/1)

### What I learned

 * [C++ template stuff]({% post_url 2015-06-15-c-template-notes %})
 * A little about ruby exception handling

### Useful references

 * Two part lecture on linking: [Part 1](http://www.cs.utexas.edu/~fussell/courses/cs429h/lectures/Lecture_20-429h.pdf)
and [Part 2](http://www.cs.utexas.edu/~fussell/courses/cs429h/lectures/Lecture_21-429h.pdf)
 * [Useful tips for monkey-patching](http://www.justinweiss.com/blog/2015/01/20/3-ways-to-monkey-patch-without-making-a-mess/)
 * [Overwriting methods in monkey patches](http://stackoverflow.com/questions/4470108/when-monkey-patching-a-method-can-you-call-the-overridden-method-from-the-new-i)
