# LinearOperator

<hr />

[![Test](https://github.com/cornellius-gp/linear_operator/actions/workflows/run_test_suite.yml/badge.svg)](https://github.com/cornellius-gp/linear_operator/actions/workflows/run_test_suite.yml)
[![Documentation](https://readthedocs.org/projects/linear-operator/badge/?version=latest)](https://linear-operator.readthedocs.io/en/latest/?badge=latest)

[![Conda](https://img.shields.io/conda/v/gpytorch/linear_operator.svg)](https://anaconda.org/gpytorch/linear_operator)
[![PyPI](https://img.shields.io/pypi/v/linear_operator.svg)](https://pypi.org/project/linear_operator)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

LinearOperator is a PyTorch package for abstracting away the linear algebra routines needed for structured matrices (or operators).

**LinearOperator is in beta.**
Currently, most of the functionality only supports positive semi-definite and triangular operators.

Package development TODOs:
 - [x] Support PSD operators
 - [x] Support triangular operators
 - [ ] Interface to specify structure (i.e. symmetric, triangular, PSD, etc.)
 - [ ] Add algebraic routines for symmetric operators
 - [ ] Add algebraic routines for generic operators

## Why LinearOperator
Before describing what linear operators are and why they make a useful abstraction, it's easiest to see an example.
Let's say you wanted to compute a matrix solve:

$$\boldsymbol A^{-1} \boldsymbol b.$$

If you didn't know anything about the matrix $\boldsymbol A$, the simplest (and best) way to accomplish this in code is:

```python
# A = torch.randn(1000, 1000)
# b = torch.randn(1000)
torch.linalg.solve(A, b)  # computes A^{-1} b
```

While this is easy, the `solve` routine is $\mathcal O(N^3)$, which gets very slow as $N$ grows large.

However, let's imagine that we knew that $\boldsymbol A$ was equal to a low rank matrix plus a diagonal
(i.e. $\boldsymbol A = \boldsymbol C \boldsymbol C^\top + \boldsymbol D$
for some skinny matrix $\boldsymbol C$ and some diagonal matrix $\boldsymbol D$.)
There's now a very efficient $\boldsymbol O(N)$ routine to compute $\boldsymbol A^{-1}$
called the [Woodbury formula](https://en.wikipedia.org/wiki/Woodbury_matrix_identity).
**In general**, if we know that $\boldsymbol A$ has structure,
we want to use efficient linear algebra routines (rather than the general routines)
that will exploit this structure.


### Without LinearOperator

Implementing the efficient solve that exploits the structure in $\boldsymbol A$ would look something like this:

```python
def low_rank_plus_diagonal_solve(C, d, b):
    # A = C C^T + diag(d)
    # A^{-1} b = D^{-1} b - D^{-1} C (I + C^T D^{-1} C)^{-1} C^T D^{-1} b
    #   where D = diag(d)

    D_inv_b = b / d
    D_inv_C = C / d.unsqueeze(-1)
    eye = torch.eye(C.size(-2))
    return (
        D_inv_b - D_inv_c @ torch.cholesky_solve(
            C.mT @ D_inv_b,
            torch.linalg.cholesky(eye + C.mT @ D_invC, upper=False),
            upper=False
        )
    )


# C = torch.randn(1000, 20)
# d = torch.randn(1000)
# b = torch.randn(1000)
low_rank_plus_diagonal_solve(C, d, b)  # computes A^{-1} b efficiently
```

While this is efficient code, it's not ideal for a number of reasons:
1. It's a lot more complicated than `torch.linalg.solve(A, b)`.
2. There's no object that represents $\boldsymbol A$.
   To perform any math with $\boldsymbol A$, we have to pass around the matrix `C` and the vector `d`.


### With LinearOperator

The LinearOperator package offers the best of both worlds:

```python
from linear_operator.operators import DiagLinearOperator, RootLinearOperator
# C = torch.randn(1000, 20)
# d = torch.randn(1000)
# b = torch.randn(1000)
A = RootLinearOperator(C) + DiagLinearOperator(d)  # represents C C^T + diag(d)
```

it provides an interface that lets us treat $\boldsymbol A$ as if it were a generic tensor,
using the standard PyTorch API...

```python
torch.linalg.solve(A, d, b)  # computes A^{-1} b efficiently!
```

but under-the-hood, the `LinearOperator` object keeps track of the algebraic structure of $\boldsymbol A$ (low rank plus diagonal)
and determines the most efficient routine to use (the Woodbury formula).
This way, we can get a efficient ($\mathcal O(N)$) solve while abstracting away all of the details.

Crucially, $\boldsymbol A$ is never explicitly instantiated as a matrix, which makes it possible to scale
to very large operators without running out of memory:

```python
C = torch.randn(10000000, 20)
d = torch.randn(10000000)
b = torch.randn(10000000)
A = RootLinearOperator(C) + DiagLinearOperator(d)  # represents a 10M x 10M matrix!
torch.linalg.solve(A, d, b)  # computes A^{-1} b efficiently!
```

### What is a Linear Operator?
A linear operator is a generalization of a matrix.
It is a linear function that is defined in by its application to a vector.
The most common linear operators are (potentially structured) matrices,
where the function applying them to a vector are (potentially efficient)
matrix-vector multiplication routines.

In code, a `LinearOperator` is a class that
1. specifies the tensor(s) needed to define the LinearOperator,
1. specifies a `_matmul` function (how the LinearOperator is applied to a vector),
1. specifies a `_size` function (how big is the LinearOperator if it is represented as a matrix, or batch of matrices), and
1. specifies a `_transpose_nonbatch` function (the adjoint of the LinearOperator).

For example:

```python
class DiagLinearOperator(linear_operator.LinearOperator):
    r"""
    A LinearOperator representing a diagonal matrix.
    """
    def __init__(self, diag):
        # diag: the vector that defines the diagonal of the matrix
        self.diag = diag

    def _matmul(self):
        return torch.Size([*self.diag.shape, self.diag.size(-1)])

    def _size(self):
        return torch.Size([*self.diag.shape, self.diag.size(-1)])

    def _transpose_nonbatch(self):
        return self  # Diagonal matrices are symmetric


# ...

D = DiagLinearOperator(torch.tensor([1., 2., 3.])
# Represents the matrix
#   [[1., 0., 0.],
#    [0., 2., 0.],
#    [0., 0., 3.]]
torch.matmul(D, torch.tensor([4., 5., 6.])
# Returns [4., 10., 18.]
```

Other functions (e.g. `logdet`, `eigh`, etc.) can be defined as needed if efficient routines exist.

#### Why is This Useful?
While `_matmul`, `_size`, and `_transpose_nonbatch` might seem like a limited set of functions,
it turns out that most functions on the `torch` and `torch.linalg` namespaces can be efficiently implemented
using only these three primitative functions.

Moreover, because `_matmul` is a linear function, it is very easy to compose linear operators in various ways.
For example: adding two linear operators (`SumLinearOperator`) just requires adding the output of their `_matmul` functions.
This makes it possible to define very complex compositional structures that still yield efficient linear algebraic routines.


## Use Cases

There are several use cases for the LinearOperator package.
Here we highlight two general themes:

### Modular Code for Structured Matrices

For example, let's say that you have a generative model that involves
sampling from a high-dimensional multivariate Gaussian.
This sampling operation will require storing and manipulating a large covariance matrix,
so to speed things up you might want to experiment with different structured
approximations of that covariance matrix.
This is easy with the LinearOperator package.

```python
from gpytorch.distributions import MultivariateNormal

# variance = torch.randn(10000)
cov = DiagLinearOperator(variance)
# or
# cov = RootLinearOperator(...) + DiagLinearOperator(...)
# or
# cov = KroneckerProductLinearOperator(...)
# or
# cov = ToeplitzLinearOperator(...)
# or
# ...

mvn = MultivariateNormal(torch.zeros(cov.size(-1), cov) # 10000-dimensional MVN
mvn.rsample()  # returns a 10000-dimensional vector
```

### Efficient Routines for Complex Operators

Many of the efficient linear algebra routines in LinearOperator are iterative algorithms
based on matrix-vector multiplication.
Since matrix-vector multiplication obeys many nice compositional properties
it is possible to obtain efficient routines for extremely complex compositional LienarOperators:

```python
from linear_operator.operators import KroneckerProductLinearOperator, RootLinearOperator, ToeplitzLinearOperator

# mat1 = 200 x 200 PSD matrix
# mat2 = 100 x 100 PSD matrix
# vec3 = 20000 vector

A = KroneckerProductLinearOperator(mat1, mat2) + RootLinearOperator(ToeplitzLinearOperator(vec3))
# represents a 20000 x 20000 matrix

torch.linalg.solve(A, torch.randn(20000))  # Sub O(N^3) routine!
```


## Using the torch API with LinearOperator

LinearOperator objects share (mostly) the same API as `torch.Tensor` objects.
They can also be used with (most) functions available in the `torch` and `torch.linalg` namespace.
This includes
- `torch.add`
- `torch.cat`
- `torch.clone`
- `torch.diagonal`
- `torch.dim`
- `torch.div`
- `torch.expand`
- `torch.logdet`
- `torch.matmul`
- `torch.mul`
- `torch.numel`
- `torch.permute`
- `torch.prod`
- `torch.squeeze`
- `torch.sub`
- `torch.sum`
- `torch.transpose`
- `torch.unsqueeze`
- `torch.linalg.cholesky`
- `torch.linalg.eigh`
- `torch.linalg.eigvalsh`
- `torch.linalg.solve`
- `torch.linalg.svd`

Each of these functions will either return a `torch.Tensor`, or a new `LinearOperator` object,
depending on the function.
For example:

```python
# A = RootLinearOperator(...)
# B = ToeplitzLinearOperator(...)
# d = vec

C = torch.mul(A, B)  # A new LienearOperator representing the elementwise mul between A and B
torch.linalg.solve(C, d)  # A torch.Tensor
```

For more examples, see the [examples folder](https://github.com/cornellius-gp/linear_operator/blob/main/examples/).


## Contributing

See the contributing guidelines [CONTRIBUTING.md](https://github.com/cornellius-gp/linear_operator/blob/main/CONTRIBUTING.md)
for information on submitting issues and pull requests.


## License

LinearOperator is [MIT licensed](https://github.com/cornellius-gp/linear_operator/blob/main/LICENSE).
