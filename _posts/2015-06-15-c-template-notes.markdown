---
layout: post
title: "C++ Template Notes"
date: 2015-06-15T15:27:08-04:00
---
`class` vs `typename` in template argument list:
[no difference](http://stackoverflow.com/questions/213121/use-class-or-typename-for-template-parameters)

Base (unspecialized) template:

```C++
template<typename T> class messenger {
 public:
  void message(T m) {
    cout << m << endl;
  }
};
```

Can specialize template:

```C++
template<> class messenger<int> {
 public:
  void message(int m) {
    cout << "int version" << endl;
    cout << m << endl;
  }
};
```

Can also partially specialize class templates, but not function templates
(example from
[here](http://en.cppreference.com/w/cpp/language/partial_specialization)):

```C++
template<class T1, class T2, int I>
class A {}; // primary template
 
template<class T>
class A<int, T*, 5> {}; // partial specialization where T1 is int, I is 5,
                        // and T2 is a pointer
```

With function templates, can omit template arguments (when either calling the
function or specializing the template) if they can be
inferred (second example from
[here](http://en.cppreference.com/w/cpp/language/template_argument_deduction)):

```C++
template<typename T> void trapeze(T trapezer) { }

template<> void trapeze<int>(int trapezer) { } //OK

template<> void trapeze(double trapezer) { } //Also OK

void trapeze_swing() {
  float f = 5.0;
  double d = 5.0;

  //All OK
  trapeze<float>(f);
  trapeze(f);
  trapeze<double>(d);
  trapeze(d);
}

//can also partially omit (some but not all) template arguments
template<typename To, typename From> To convert(From f);
 
void g(double d) 
{
    int i = convert<int>(d); // calls convert<int,double>(double)
    char c = convert<char>(d); // calls convert<char,double>(double)
}
```

Instead of specializing, can also override function templates with normal functions:

```C++
template<typename T> void trapeze(T trapezer) { }

void trapeze(int trapezer) { }
```

According to this guy, it's a [better choice to override than specialize for
functions](http://www.gotw.ca/publications/mill17.htm).

We do a lot of function specialization in nmatrix. Here's [one
example](https://github.com/SciRuby/nmatrix/blob/e8a430545d06c51e59903fc7704c368c4d21ba1e/ext/nmatrix/math/gemm.h):

```C++
template <typename DType>
inline void gemm(const enum CBLAS_ORDER Order, const enum CBLAS_TRANSPOSE TransA, const enum CBLAS_TRANSPOSE TransB, const int M, const int N, const int K,
                 const DType* alpha, const DType* A, const int lda, const DType* B, const int ldb, const DType* beta, DType* C, const int ldc)
{
}

template <>
inline void gemm(const enum CBLAS_ORDER Order, const enum CBLAS_TRANSPOSE TransA, const enum CBLAS_TRANSPOSE TransB, const int M, const int N, const int K,
                 const float* alpha, const float* A, const int lda, const float* B, const int ldb, const float* beta, float* C, const int ldc)
{
}
```

We can change the latter to an overload just by removing the `template <>`,
but then we need to change the [caller](https://github.com/SciRuby/nmatrix/blob/e8a430545d06c51e59903fc7704c368c4d21ba1e/ext/nmatrix/math.cpp#L513) from `gemm<DType>(...)`  to `gemm(...)`.
After that things work okay. Should we do this?

Since templates need to be defined in the header file, and multiple source
files will include the same header, it is okay to link together multiple object
files each with a definition of the template. This means the function
symbols are marked as weak
symbols. However, it's not OK if these different definitions behave
differently. When linking the files, if there are multiple weak symbols with
the same name, the linker can just pick an arbitrary one.

However, we can only
have one definition of a function template specialization (these become
normal strong symbols). So these should be defined in
source files and only declared in headers. Unless of course the template
specialization is an inline function, then no problem if it's defined in a
header.
