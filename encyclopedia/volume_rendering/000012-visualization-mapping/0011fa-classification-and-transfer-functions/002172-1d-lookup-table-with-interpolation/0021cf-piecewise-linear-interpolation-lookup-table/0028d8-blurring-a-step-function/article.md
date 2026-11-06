# Blurring a Step Function

Also called: Step Function Blurring

## Overview
Blurring a Step Function is an algorithm that convolves a step function with a rectangular pulse to produce a continuous piecewise linear spline [S1]. It serves as the core differentiable operation within the Power Ladder framework for anti-aliasing transfer functions in neural radiance fields, ensuring that discretized control points yield smooth, band-limited classification responses [S1]. It distinguishes itself from standard interpolation by explicitly modeling the integration of a uniform kernel over the domain, thereby preventing aliasing artifacts during optimization [S1].

## How it works
**Inputs:**
- `x`: Array of $N$ monotonically increasing domain values (control point locations).
- `y`: Array of $N$ range values (control point heights).
- `r`: Scalar half-width of the rectangular convolution kernel.

**Outputs:**
- `xr`: Array of $2N$ sorted domain values defining the breakpoints of the resulting spline.
- `yr`: Array of $2N$ range values defining the piecewise linear spline at `xr`.

**Data Structures:**
- Arrays are dense, contiguous, and support vectorized operations (e.g., NumPy/JAX tensors).
- `xr` and `yr` have length $2N$.

**Parameters:**
- `r`: Half-width of the rectangular pulse. Must be positive. Typical values depend on the resolution of the domain and the desired smoothing strength [S1].

**Algorithm Steps:**
1.  **Compute Breakpoints:** Concatenate the shifted domain arrays $[x - r, x + r]$ and sort them to obtain the new domain `xr` and the sorting indices `sortidx`. The sort must return both the sorted values and the indices of the original elements [S1][S2, unofficial].
2.  **Compute Slopes ($y'$):** Create a shifted version of $y$ by appending a zero at the end and prepending a zero. Specifically, compute the difference between $[y, 0]$ and $[0, y]$, then divide by $2r$. This yields the slope of the linear segments:
    $$ y' = \frac{[y, 0] - [0, y]}{2r} $$
    Here, $[y, 0]$ is $y$ with a zero appended, and $[0, y]$ is $y$ with a zero prepended [S1][S2, unofficial].
3.  **Compute Second Derivative Terms ($y''$):** Concatenate $y'$ and $-y'$ to form an array of length $2N$. Reorder this concatenated array using `sortidx`, excluding the last index. This step aligns the slope contributions with the sorted breakpoints:
    $$ y'' = \text{concatenate}([y', -y'])[\text{sortidx}[:-1]] $$
    [S1][S2, unofficial].
4.  **Integrate to Output ($yr$):** Compute the interval widths between consecutive breakpoints in `xr`. Calculate the cumulative sum of $y''$. Multiply the interval widths by this cumulative sum, then take the cumulative sum of the result. Prepend a zero to the final array to align it with `xr`:
    $$ yr = [0, \text{cumsum}((xr[1:] - xr[:-1]) \cdot \text{cumsum}(y''))] $$
    [S1][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def blur_stepfun(x, y, r):
    """
    Convolve a step function (x, y) with a rectangular pulse of half-width r.
    Returns piecewise linear spline (xr, yr).
    """
    # Step 1: Compute breakpoints and sort indices
    # Concatenate [x - r, x + r]
    x_shifted = concatenate([x - r, x + r])
    # Sort and get indices. Note: sort() must return indices (argsort behavior)
    xr, sortidx = sort(x_shifted, return_indices=True)
    
    # Step 2: Compute slopes y'
    # [y, 0] is y with a zero appended
    y_appended = concatenate([y, [0]])
    # [0, y] is y with a zero prepended
    y_prepended = concatenate([[0], y])
    # y' = (y_appended - y_prepended) / (2r)
    y_prime = (y_appended - y_prepended) / (2 * r)
    
    # Step 3: Compute y''
    # Concatenate [y', -y']
    y_double = concatenate([y_prime, -y_prime])
    # Index with sortidx, excluding the last element
    # sortidx has length 2N, we take the first 2N-1 indices? 
    # Source says sortidx[:-1]. Since y_double has length 2N, and sortidx has length 2N,
    # y_double[sortidx] has length 2N. We take all but the last.
    y_double_prime = y_double[sortidx[:-1]]
    
    # Step 4: Compute yr
    # Interval widths
    dx = xr[1:] - xr[:-1]
    # Cumulative sum of y''
    cum_y_double_prime = cumsum(y_double_prime)
    # Multiply widths by cumulative sum
    weighted_cum = dx * cum_y_double_prime
    # Cumulative sum of the result
    cum_weighted = cumsum(weighted_cum)
    # Prepend zero
    yr = concatenate([[0], cum_weighted])
    
    return xr, yr
```

**Edge Cases and Pitfalls:**
- **Precision:** The subtraction $[y, 0] - [0, y]$ can suffer from cancellation errors if $y$ values are very small or nearly identical. Use stable floating-point operations if necessary [G].
- **Boundary Conditions:** The algorithm assumes the step function is zero outside the range $[x_0 - r, x_{N-1} + r]$. The output `yr` correctly reflects this by starting at 0 and integrating the slopes [S1].
- **Degenerate Input:** If $r=0$, division by zero occurs. The algorithm is undefined for $r=0$ [S1]. If $x$ is not sorted, the sort step handles it, but the logical meaning of "step function" implies monotonic $x$ [S1].

## Subtopics

### [Transfer Function Control Point Management](2718-transfer-function-control-point-management.md)
Manages the optimization of control points $x$ and $y$. Use when you need to adjust the shape of the transfer function dynamically.

### [Step Function Resampling](7924-step-function-resampling.md)
Resamples a step function to a different domain resolution. Use when changing the discretization of the input domain.

## Operating characteristics
- **Time Complexity:** $O(N \log N)$ due to the sorting step, where $N$ is the number of control points. The subsequent operations are linear $O(N)$ [S1].
- **Memory Complexity:** $O(N)$ for storing the intermediate arrays `xr`, `yr`, `y'`, and `y''` [S1].
- **Strengths:** Differentiable with respect to $x$, $y$, and $r$, enabling gradient-based optimization. Provides exact anti-aliasing for piecewise constant transfer functions [S1].
- **Limitations:** Only produces piecewise linear splines. Not suitable for higher-order continuity requirements without further smoothing [S1].
- **When to Use:** When anti-aliasing is required for neural radiance field transfer functions, specifically within the Power Ladder framework [S1].
- **When Not to Use:** When higher-order smoothness (e.g., cubic) is strictly required, or when $r=0$ is needed (use standard interpolation instead) [S1].

## Background
The algorithm was introduced in the Zip-NeRF paper as part of the Power Ladder framework for anti-aliased neural radiance fields [S1]. It addresses the aliasing artifacts that arise when discretizing continuous transfer functions for volume rendering [S1]. The method is implemented in JAX/NumPy and is designed to be differentiable for end-to-end training [S1].

## Sources
[S1] Zip-NeRF: Anti-Aliased Grid-Based Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01804, fetched 2026-10-26
[S2] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
