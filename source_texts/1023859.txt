# DualNumbers

# NOTICE

**This package is unmaintained. Its reliability is not guaranteed.**

# Introduction

This is a proof of concept implementation of (forward) automatic differentiation in Julia via operator overloading. Immutable types make this approach particularly efficient, with no temporary allocations needed.

The primary interface is ``autodiff1_wrapper``, which generates a function that evaluates the gradient of a given function. It's signature is ``autodiff1_wrapper(f, T, n)``, where ``f(x::Vector{T})`` is a function that takes a vector of type T of length ``n``. It returns a function ``g!(x,storage)`` where the gradient is written *in-place* to the vector ``storage``. 

The method works by passing a vector of dual numbers (``Dual1{T}``) to ``f`` in place of a ``Vector{T}``. This means that ``f`` must be written generically. Only a few methods have been overloaded so far to support dual numbers, so you may encounter errors related to this.

Note that a call to ``g!`` essentially has the cost of ``2n`` evaluations of ``f``, like central differencing. Unlike finite differencing, however, ``g!`` will return an *exact* numerical derivative within floating-point error, not an approximation.

The generated gradient function can be used with Optim:
```
using DualNumbers
using Optim


function rosenbrock(x::Vector)
    return (1.0 - x[1])^2 + 100.0 * (x[2] - x[1]^2)^2
end

rosenbrock_autogradient! = autodiff1_wrapper(rosenbrock,Float64,2)
optimize(rosenbrock, rosenbrock_autogradient!, [0.0, 0.0], method = :l_bfgs)

```
