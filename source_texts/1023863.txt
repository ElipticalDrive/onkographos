# DualNumbersHW

Author: Matthew Parker

This package implements forward-mode automatic differentiation using parameteric dual numbers

This package supports:
- First-order derivatives
- Second-order derivatives (via nested dual numbers)
- Operator overloading (+, -, *, /, power)
- Elementary functions: sin, cos, exp, sqrt

# Background
A dual number is defined as:
x + εx'
where ε² = 0
It is possible to perform automatic differentiation by propagating derivatives through arithmatic
The nesting of the dual numbers allows higher-order derivatives to be taken

---

## Installation

From Julia REPL:

```julia
using Pkg
Pkg.add(url="https://github.com/mbparker668/DualNumbersHW.git")

# Example Use
using DualNumbersHW

f(x) = x^3 + 2x + 1

# First derivative
x = Dual(3.0, 1.0)
f(x)

# Second Derivative
x2 = Dual(Dual(3.0,1.0), Dual(1.0,0.0))
res = f(x2)
res.deriv.deriv
