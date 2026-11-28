# CDF Integration from Weights

Also called: Weight Integration

## Overview
Computes the cumulative distribution function (CDF) from a set of discrete weights that sum to 1, treating them as a step function [S1, unofficial]. It preserves the integral mass of the distribution rather than finding local extrema, and serves as the inverse operation to [Weight-PDF Conversion](12194-weight-pdf-conversion.md) [G].

## How it works
**Inputs:**
- `w`: Tensor of weights. Assumed to sum to 1 along the last axis [S1, unofficial].

**Outputs:**
- `cw0`: Tensor where `cw0[..., 0] = 0` and `cw0[..., -1] = 1` [S1, unofficial]. The size on the last dimension is one greater than the input size [S1, unofficial].

**Data Structures:**
- The output is a tensor with the same shape as the input, except the last dimension is expanded by 1 [S1, unofficial].

**Steps:**
1. Compute the cumulative sum of all weights except the last one along the last axis:
   $$cw = \text{cumsum}(w[..., :-1], \text{axis}=-1)$$
2. Clamp the cumulative sum to a maximum value of 1 to prevent numerical overflow:
   $$cw = \min(cw, 1)$$
3. Prepend a tensor of zeros and append a tensor of ones to ensure the CDF starts exactly at 0 and ends exactly at 1:
   $$cw0 = \text{concatenate}([0, cw, 1], \text{axis}=-1)$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cdf_integration_from_weights(w):
    # w: 2D list of weights. Assumes each row sums to 1.
    cw0_rows = []
    for row in w:
        # Step 1: Cumulative sum of all weights except the last one
        cw = []
        acc = 0.0
        for i in range(len(row) - 1):
            acc += row[i]
            cw.append(acc)
        
        # Step 2: Clamp to 1 to prevent numerical overflow
        cw = [min(val, 1.0) for val in cw]
        
        # Step 3: Prepend zeros and append ones
        cw0 = [0.0] + cw + [1.0]
        cw0_rows.append(cw0)
    return cw0_rows
```

**Parameters:**
- `w`: Input tensor of weights assumed to sum to 1 along the last axis [S1, unofficial].

**Edge Cases and Pitfalls:**
- The function silently breaks if the input weights do not sum to 1 along the last axis [S1, unofficial].

## Subtopics

### [Weight-PDF Conversion](12194-weight-pdf-conversion.md)
Transforms a vector of discrete weights that sum to 1 into a probability density function (PDF) that integrates to 1 over given spatial or temporal intervals, and vice versa. This method serves as the inverse operation to CDF Integration from Weights, bridging the gap between discrete sampling weights and continuous density representations required for accurate volume rendering classification. It is distinct from simple normalization by accounting for the variable width of the intervals defined by the boundaries $t$.

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] AlexSheldrick/density_bounded_ray_weights, https://github.com/AlexSheldrick/density_bounded_ray_weights, fetched 2026-10-27 (unofficial)
