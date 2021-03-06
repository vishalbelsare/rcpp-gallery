---
title: "Documenting Rcpp functions and classes in R packages"
author: "Artem Sokolov and Dirk Eddelbuettel"
license: GPL (>= 2)
tags: modules basics
date: 2020-01-02
summary: Demonstrates how to add Roxygen blocks in .cpp source
layout: post
src: 2020-01-02-documenting-rcpp-packages.Rmd
---

### Introduction

[Roxygen2](https://cran.r-project.org/package=roxygen2) is a convenient way to document functions in
R packages. Based on the [Doxygen](https://en.wikipedia.org/wiki/Doxygen) model, it parses relevant
information from the comments and generates the corresponding `man/*.Rd` files. The major strength
of this model -- besides not having to write `*.Rd` files by hand -- is that documentation ends up
living right next to the functionality it describes, thus enabling easy maintenance and
synchronization between the two.

When using Rcpp in package development, the R functions in `RcppExports.R` are generated
automatically from the source `.cpp` files. Rcpp also faithfully transcribes comment blocks from
`.cpp` to `.R`. We can take advantage of this fact to add documentation directly to the `.cpp` files
and have it appear in the package reference manual.

This vignette demonstrates four different ways to document Rcpp functions and classes in R
packages. We will create an R package from scratch, add some very basic functionality and show how
Roxygen2 blocks appear in the final documentation. Throughout the vignette, we will focus on base R,
Rcpp and roxygen2 with no additional dependencies. There are a number of popular add-on packages
helping with package development (_e.g._ `devtools`, `remotes`, `usethis` as of January 2020), and
these additional helpers may be reviewed in a follow-up vignette.  In case
Rcpp or roxygen2 are not yet installed, do


{% highlight r %}
install.packages(c("Rcpp","roxygen2"))
{% endhighlight %}

### Create a package

We begin by initializing a package skeleton. The following functions create a new directory and
establish a barebones structure for an R package with Rcpp support:


{% highlight r %}
Rcpp::Rcpp.package.skeleton("testpkg", path="~/test/")
{% endhighlight %}

Using your favorite text editor, create a new file `~/test/testpkg/R/zzz.R` and add the following
lines to it:


{% highlight r %}
Rcpp::loadModule("double_cpp", TRUE)
{% endhighlight %}

This exposes the functionality of an [Rcpp
module](http://dirk.eddelbuettel.com/code/rcpp/Rcpp-modules.pdf) that we will use to encapsulate our
C++ class. Replace `"double_cpp"` with your desired module name, but note that the name has to match
what appears in the `.cpp` file below.

### Add C++ functionality and Roxygen2 comment blocks

All code blocks presented in this section are part of the same contiguous `.cpp` file. For the
purposes of this tutorial, assume that the file lives in `~/test/testpkg/src/mult.cpp`. Let's review
the four different ways to introduce Roxygen2 comment blocks inside `.cpp` files.

#### Approach 1: Standalone functions

The first approach is the most straightforward and mimics traditional usage of Roxygen2 in pure R
packages. The comments are placed immediately before a stand-alone function and begin with `//'`,
which Rcpp will translate into `#'` in the corresponding `RcppExports.R` file:


{% highlight cpp %}
#include <Rcpp.h>

//' Multiplies two doubles
//'
//' @param v1 First value
//' @param v2 Second value
//' @return Product of v1 and v2
// [[Rcpp::export]]
double mult( double v1, double v2 ) {return v1*v2;}
{% endhighlight %}

#### Approach 2: Nested field structure

Classes impose an additional layer of hierarchy by grouping multiple functions and variables. We
need to decide how this hierarchy should be represented in the R help pages, which themselves follow a
flat format. If a class is relatively small, a simple solution is to use a nested field structure to
describe individual member functions. The entire class will then appear as a single entry in the
final package reference manual.


{% highlight cpp %}
//' @name Double
//' @title Encapsulates a double
//' @description Type the name of the class to see its methods
//' @field new Constructor
//' @field mult Multiply by another Double object \itemize{
//' \item Parameter: other - The other Double object
//' \item Returns: product of the values
//' }
class Double {
public:
  Double( double v ) : value(v) {}
  double mult( const Double& other ) {return value * other.value;}
private:
  double value;
};
{% endhighlight %}

#### Approach 3: Stand-alone pages for individual class methods

Occasionally, a member function may be complex enough to require its own `?` reference entry. A
major advantage of Roxygen2 is that such entries can be created with relative ease by placing the
corresponding comment block at the top level in a file.


{% highlight js %}
//' @name Double$new
//' @title Constructs a new Double object
//' @param v A value to encapsulate
{% endhighlight %}

The only disadvantage is that the documentation does not live directly next to the function it
describes. Unforunately, this is due to a limitation that only the top-level comment blocks are
exported into R by Rcpp.

#### Approach 4: Rcpp module docstrings

The final place for function documentation is inside the docstring feature provided by the Rcpp
modules themselves. This works well for relatively simple classes. Unfortunately, the documentation
becomes overly verbose, if a class makes heavy use of templates.


{% highlight cpp %}
RCPP_EXPOSED_CLASS(Double)
RCPP_MODULE(double_cpp) {
  using namespace Rcpp;

  class_<Double>("Double")
    .constructor<double>("Wraps a double")
    .method("mult", &Double::mult, "Multiply by another Double object")
    ;
}
{% endhighlight %}

### Compile, install and test

After finishing `~/test/testpkg/R/zzz.R` and `~/test/testpkg/src/mult.cpp`, run the following
commands in an R session somewhere inside `~/test/testpkg`:


{% highlight r %}
Rcpp::compileAttributes()           # this updates the Rcpp layer from C++ to R
roxygen2::roxygenize(roclets="rd")  # this updates the documentation based on roxygen comments
{% endhighlight %}

followed by the usual `R CMD build` and `R CMD install` (or equivalent helper functions via RStudio,
the `usethis` or `devtools` package or alike). 

The first command compiles the `.cpp` code and generates `R/RcppExports.R`. You will notice that the
roxygen comment blocks are faithfully transcribed from `.cpp` to `.R`. The second command then
generates the `man/*.Rd` files based on these roxygen blocks. Having prepared the sources, we then
build and install package as usual, making it available through the standard `library(testpkg)`
interface.

To test the package and view its documentation, start a fresh R session and examine the help pages.


{% highlight r %}
library( help=testpkg )     # Lists our functions: Double, Double$new and mult
                            #     along with those generated by Rcpp.package.skeleton()
?testpkg::mult              # Approach 1: stand-alone functions
?testpkg::Double            # Approach 2: nested field structure
?testpkg::`Double$new`      # Approach 3: individual class methods
testpkg::Double             # Approach 4: docstrings
{% endhighlight %}

We can also make sure that the package functions as expected.


{% highlight r %}
testpkg::mult(2, 3)    
# [1] 6

d1 = testpkg::Double$new(5)
d2 = testpkg::Double$new(3)
d1$mult(d2)
# [1] 15
{% endhighlight %}

### Going Further

As mentioned above, additional packages (or tools like the RStudio IDE) offer to help with package
creation, documentation, build and more.  As well, the
[roxygen2](https://cran.r-project.org/package=roxygen2) can do more than we showed here by
auto-generating the `NAMESPACE` file, collating R files as needed and more. However, we feel it
helps to understand what each relevant tool offers in and by itself, and also appreciate the
relative simplicity of the tools describe in the [Writing R
Extensions](https://cran.r-project.org/doc/manuals/r-release/R-exts.html) manual that is part of
base R.  A follow-up vignette may describe the additional tools.
