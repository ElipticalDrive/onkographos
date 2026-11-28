# Inverse Sigmoid Function
Also called: Inverse Sigmoid
## Overview
Computes the inverse of the sigmoid function (logit) for a given input tensor [S1, unofficial][S2, unofficial]. Unlike the parent [Sigmoid Function with Jacobian](6497-sigmoid-function-with-jacobian.md) which maps reals to probabilities, this method performs the reverse transformation [G].
## How it works
**Inputs and Outputs:**
- Input tensor $x$: Floating-point values in $(0, 1)$ [G].
- Output tensor $y$: Floating-point values of the same shape [G].
**Data Structures:**
- Not given in the sources.
**Formula:**
$$ f(x) = \log\left(\frac{x}{1-x}\right) $$ [S2, unofficial]
where $\log$ denotes the natural logarithm [G].
**Steps:**
1. Compute the ratio $r = \frac{x}{1-x}$.
2. Compute the natural logarithm of the ratio: $y = \log(r)$ [S1, unofficial][S2, unofficial].
**Parameters:**
- None.
**Edge Cases and Pitfalls:**
- **Boundary Values:** The function is undefined at $x=0$ and $x=1$ due to division by zero and logarithm of zero [G]. In practice, inputs must be clamped to a small epsilon range, e.g., $[\epsilon, 1-\epsilon]$, where $\epsilon$ is a small positive number (e.g., $10^{-7}$) to prevent numerical instability [G].
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def inverse_sigmoid(x):
    # Applies element-wise to each component of the input tensor/list
    return [math.log(xi / (1.0 - xi)) for xi in x]
```
## Subtopics

## Operating characteristics

## Background

## Sources
[S1] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
[S2] LogosIntelligenceLab/4d-gaussian-splatting, https://github.com/LogosIntelligenceLab/4d-gaussian-splatting, fetched 2026-10-27 (unofficial)
