Using the TensorFlow API from R
================

Introduction
------------

The **tensorflow** package provides access to the entire TensorFlow API from within R. The principle motivation for the package is to provide a foundation for higher-level R libraries that make building models with TensorFlow productive, straightforward, and well-integrated with the R ecosystem.

While the main motivation is the creation of higher-level R packages, it is also possible to use the lower-level TensorFlow API directly from R. The main technical consideration here is that substantial parts TensorFlow are implemented in Python, which requires bridging between the R and Python runtimes, data type conversion, etc. This document describes the R API to TensorFlow and the various idioms and techniques required to use it productively.

### Hello, TensorFlow

The R API to TensorFlow is based on direct invocation of TensorFlow's Python functions and objects. The two languages have many similarities so the R code will often look quite similar to the equivalent Python code. Here is the R version "Hello, World" for TensorFlow:

``` r
library(tensorflow)

sess = tf$Session()

hello <- tf$constant('Hello, TensorFlow!')
sess$run(hello)

a <- tf$constant(10L)
b <- tf$constant(32L)
sess$run(a + b)
```

Here is the equivalent Python code:

``` python
import tensorflow as tf

sess = tf.Session()

hello = tf.constant('Hello, TensorFlow!')
print(sess.run(hello))

a = tf.constant(10)
b = tf.constant(32)
print(sess.run(a + b))
```

Data Types
----------

When interacting with the TensorFlow API R data types are automatically converted to Python data-types and vice-versa, making the use of the API fairly seamless from R. Core data types are mapped as follows:

| R                      | Python/TensorFlow | Examples                                 |
|------------------------|-------------------|------------------------------------------|
| Single-element vector  | Scalar            | `1`, `1L`, `TRUE`, `"foo"`               |
| Multi-element vector   | List              | `c(1, 2, 3)`, `c(1L, 2L, 3L)`            |
| List of multiple types | Tuple             | `list(1, TRUE, "foo")`                   |
| Named list             | Dict              | `list(a = 1, b = 2)`                     |
| Matrix/Array           | NumPy ndarray     | `matrix(c(1,2,3,4), nrow = 2, ncol = 2)` |
| NULL, TRUE, FALSE      | None, True, False | `NULL`, `TRUE`, `FALSE`                  |

The automatic conversion of data types makes for R code which looks very similar to equivalent Python code (save for the `$` used in R which is analogous to the `.` which is used in Python). Here's another simple example of TensorFlow code in R:

``` r
library(tensorflow)

# Create 100 phony x, y data points, y = x * 0.1 + 0.3
x_data <- runif(100, min=0, max=1)
y_data <- x_data * 0.1 + 0.3

# Try to find values for W and b that compute y_data = W * x_data + b
# (We know that W should be 0.1 and b 0.3, but TensorFlow will
# figure that out for us.)
W <- tf$Variable(tf$random_uniform(shape(1L), -1.0, 1.0))
b <- tf$Variable(tf$zeros(shape(1L)))
y <- W * x_data + b

# Minimize the mean squared errors.
loss <- tf$reduce_mean((y - y_data) ^ 2)
optimizer <- tf$train$GradientDescentOptimizer(0.5)
train <- optimizer$minimize(loss)

# Launch the graph and initialize the variables.
sess = tf$Session()
sess$run(tf$initialize_all_variables())

# Fit the line (Learns best fit is W: 0.1, b: 0.3)
for (step in 1:201) {
  sess$run(train)
  if (step %% 20 == 0)
    cat(step, "-", sess$run(W), sess$run(b), "\n")
}
```

You can see the equivalent Python code here: <https://www.tensorflow.org/get_started/>. Aside from the use of the `shape` function (discussed in more detail below) the R and Python code are nearly identical.

There are however some important differences between how Python and R treat various data types. Navigating these differences generally requires the use of the helper functions described below.

### Numeric Types

The TensorFlow API is more finicky about numeric types than is customary in R. Many TensorFlow function paramters require integers (e.g. for tensor dimensions) and in those cases it's important to use an R integer literal (e.g. `1L`). Here's an example of specifying the `strides` parameter for a 4-dimensional tensor using integer literals:

``` r
tf$nn$conv2d(x, W, strides=c(1L, 1L, 1L, 1L), padding='SAME')
```

Here's another exmaple of using integer literals when defining a set of integer flags:

``` r
flags$DEFINE_integer('max_steps', 2000L, 'Number of steps to run trainer.')
flags$DEFINE_integer('hidden1', 128L, 'Number of units in hidden layer 1.')
flags$DEFINE_integer('hidden2', 32L, 'Number of units in hidden layer 2.')
```

### Lists

Some TensorFlow APIs call for lists of a numeric type. Typically you can use the `c` function (as illustrated above) to create lists of numeric types. However, there are a couple of special cases (mostly involving specifying the shapes of tensors) where you may need to create a numeric list with an embedded `NULL` or a numeric list with only a single item. In those cases you'll want to use the `list` function rather than `c` in order to force the argument to be treated as a list rather than a scalar, and to ensure that `NULL` elements are preserved. For example:

``` r
x <- tf$placeholder(tf$float32, list(NULL, 784L))
W <- tf$Variable(tf$zeros(list(784L, 10L)))
b <- tf$Variable(tf$zeros(list(10L)))
```

### Tensor Shapes

This need to use `list` rather than `c` is very common for shape arguments (since they are often of one dimension and in the case of placeholders often have a `NULL` dimension). For these cases there is a `shape` function which you can use to make the calling syntax a bit more more clear. For example, the above code could be re-written as:

``` r
x <- tf$placeholder(tf$float32, shape(NULL, 784L))
W <- tf$Variable(tf$zeros(shape(784L, 10L)))
b <- tf$Variable(tf$zeros(shape(10L)))
```

### Tensor Indexes

Tensor indexes within the TensorFlow API are 0-based (rather than 1-based as R vectors are). This typically comes up when specifing the dimension of a tensor to operate on (e.g with a function like `tf$reduce_mean` or `tf$argmax`). The first dimenion is `0L`, the second `1L`, and so on. For example:

``` r
# call tf$reduce_mean on the second dimension of the specified tensor
cross_entropy <- tf$reduce_mean(
  -tf$reduce_sum(y_ * tf$log(y_conv), reduction_indices=1L)
)

# call tf$argmax on the second dimension of the specified tensor
correct_prediction <- tf$equal(tf$argmax(y_conv, 1L), tf$argmax(y_, 1L))
```

### Dictionaries

To pass a dictionary keyed by character string to Python you can simply create an R named list and it will be automatically converted to a Python dictionary. However, Python dictionaries can also be keyed by arbitrary object types, and in particular by tensors within the TensorFlow API. To create a Python dictionary keyed by tensor you can use the `dict` function. For example:

``` r
sess$run(train_step, feed_dict = dict(x = batch_xs, y_ = batch_ys))
```

The `x` and `y_` variables in the above example are tensor placeholders which are substituted for by the specified training data.

### With Contexts

The TensorFlow API makes extensive use of the python `with` keyword for creating scoped execution contexts. R has a `with` generic function which is overloaded for use with TensorFlow objects for the same purpose. For example:

``` r
with(tf$name_scope('input'), {
  x <- tf$placeholder(tf$float32, shape(NULL, 784L), name='x-input')
  y_ <- tf$placeholder(tf$float32, shape(NULL, 10L), name='y-input')
})
```

There is also a custom `%as%` operator defined for creating a local alias to the object passed to the `with` function:

``` r
with(tf$Session() %as% sess, {
  sess$run(hello)
})
```

Getting Help
------------

As you use TensorFlow from R you'll want to get help on the various functions and classes available within the API. You can find the full reference to the TensorFlow Python API here: <https://www.tensorflow.org/api_docs/python/>. It's generally very easy to map Python constructs used in the documentation to R, and most R code will look nearly identical to it's Python counterpart.

If you are running the vary latest [Preview Release](https://www.rstudio.com/products/rstudio/download/preview/) (v1.0.18 or later) of RStudio IDE you can also get code completion and inline help for the TensorFlow API within RStudio. For example:

<img src="images/completion-functions.png" style="margin-bottom: 15px; border: solid 1px #cccccc;" width="804" height="260" />

Inline help is also available for function parameters:

<img src="images/completion-params.png" style="margin-bottom: 15px; border: solid 1px #cccccc;" width="804" height="177" />

You can press the F1 key while viewing inline help (or whenever your cursor is over a TensorFlow API symbol) and you will be navigated to the location of that symbol's help within the [TensorFlow API](https://www.tensorflow.org/api_docs/python/) documentation.