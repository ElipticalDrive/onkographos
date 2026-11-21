# DualNumberAD

Dual number arithmetic for forward-mode automatic differentiation in Julia, supporting first and second derivatives via nested dual numbers.

## Installation (local)

1. Download or clone this repository.
2. In Julia:

```julia
import Pkg
Pkg.activate("path/to/DualNumberAD") Replace `"path/to/DualNumberAD"` with the path to where the package is saved
Pkg.instantiate()
using DualNumberAD
```

## Example Usage
First derivative
```julia
using DualNumberAD

f(x) = x^3 + 2x - 5
deriv1(f, 2.0)   # expected: 14.0
```
Second derivative
```julia
using DualNumberAD

f(x) = x^3 + 2x - 5
deriv2(f, 2.0)   # expected: 12.0
```

## Pluto notebook
If you open DualNumberAD_tests.jl in Pluto, it activates this project and evaluates first and second derivatives for several test functions.
