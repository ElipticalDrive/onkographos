# Step Function Resampling

Also called: Inner/Outer Measures

## Overview
Step Function Resampling constructs inner and outer measures to resample a step function defined by endpoints $t_p$ and values $v_p$ into a new set of query intervals $t$. It solves the problem of accurately mapping discrete, piecewise-constant data to a different resolution or sampling grid, which is critical in volume rendering when mapping classification transfer functions or density fields. It differs from [Piecewise Linear Interpolation (Lookup Table)](2643-piecewise-linear-interpolation-lookup-table.md) by preserving the integral (mass) of the step function rather than interpolating values at points, and from [Blurring a Step Function](4383-blurring-a-step-function.md) by providing exact measure-based aggregation rather than kernel-based smoothing.

## How it works
**Inputs:**
- Step function endpoints $t_1$ (array of size $N$) and values $y_1$ (array of size $N$).
- Query points $t_0$ (array of size $M$) defining new intervals.
- Boolean flag `use_avg`: if `True`, returns weighted average; if `False`, returns sum.
- Small epsilon $\epsilon$ to prevent division by zero (default: machine epsilon of $t_0$'s dtype).

**Outputs:**
- Resampled values $y_0$ (array of size $M-1$) corresponding to intervals defined by $t_0$.

**Data Structures:**
- Cumulative sum array `cy1` of size $N+1$.
- Indices `idx_lo` and `idx_hi` of size $M$.

**Steps:**

1. **Compute Cumulative Sum:**
   Construct `cy1` by prepending a zero to the cumulative sum of $y_1$:
   $$ \text{cy1} = [0, y_1[0], y_1[0]+y_1[1], \dots, \sum_{i=0}^{N-1} y_1[i]] $$
   In code: `cy1 = torch.cat([torch.zeros_like(y1[..., :1]), torch.cumsum(y1, dim=-1)], dim=-1)` [S1, unofficial].

2. **Find Insertion Indices:**
   Determine the positions of $t_0$ within $t_1$ using binary search (e.g., `searchsorted`):
   $$ \text{idx\_lo}, \text{idx\_hi} = \text{searchsorted}(t_1, t_0) $$
   Here, `idx_lo` is the index of the first element in $t_1$ greater than or equal to $t_0$, and `idx_hi` is the index of the first element strictly greater than $t_0$ [S1, unofficial].

3. **Gather Cumulative Values:**
   Retrieve cumulative values at the found indices:
   $$ \text{cy1\_lo} = \text{cy1}[\text{idx\_lo}] $$
   $$ \text{cy1\_hi} = \text{cy1}[\text{idx\_hi}] $$
   Note: Implementation uses `torch.take_along_dim` to handle batched dimensions correctly [S1, unofficial].

4. **Compute Outer Measure:**
   The outer measure represents the total mass of the step function within the query intervals, regardless of overlap with the original step boundaries. It is calculated as the difference between the cumulative sums at the boundaries of the query intervals:
   $$ y_{0,\text{outer}}[i] = \text{cy1\_hi}[i+1] - \text{cy1\_lo}[i] $$
   Specifically: `y0_outer = cy1_hi[..., 1:] - cy1_lo[..., :-1]` [S1, unofficial].

5. **Compute Inner Measure:**
   The inner measure represents the mass of the step function that lies strictly within the query intervals, excluding any partial overlaps at the boundaries. It is non-zero only if the query interval is fully contained within a single step of the original function:
   $$ y_{0,\text{inner}}[i] = \begin{cases} \text{cy1\_lo}[i+1] - \text{cy1\_hi}[i] & \text{if } \text{idx\_hi}[i] \le \text{idx\_lo}[i+1] \\ 0 & \text{otherwise} \end{cases} $$
   In code: `y0_inner = torch.where(idx_hi[..., :-1] <= idx_lo[..., 1:], cy1_lo[..., 1:] - cy1_hi[..., :-1], torch.zeros_like(idx_lo[..., 1:]))` [S1, unofficial].

6. **Final Resampling:**
   - If `use_avg` is `False`:
     The result is the outer measure, which corresponds to the sum of the step function values over the query intervals.
     $$ y_0 = y_{0,\text{outer}} $$
   - If `use_avg` is `True`:
     The result is the weighted average value over the query intervals. This requires resampling the weighted values $v_p \cdot w_p$ and the widths $w_p$ separately, where $w_p = t_1[i+1] - t_1[i]$ are the widths of the original steps.
     1. Compute original step widths: $w_p = \text{diff}(t_1)$.
     2. Compute weighted values: $z_p = v_p \cdot w_p$.
     3. Resample $z_p$ and $w_p$ into the new intervals using the **sum mode** (same logic as steps 1-4 above, but applied to $z_p$ and $w_p$ respectively). Let the resampled sums be $S_z$ and $S_w$.
     4. Compute average:
        $$ y_0 = \frac{S_z}{\max(\epsilon, S_w)} $$
     [S1, unofficial].

**Parameters:**
- `eps`: Small value to prevent division by zero when `use_avg=True`. Default is `torch.finfo(t.dtype).eps` [S1, unofficial].

**Edge Cases and Pitfalls:**
- If $t_0$ contains values outside the range of $t_1$, `searchsorted` behavior must be defined (typically clamping or raising an error). The provided implementation assumes valid ranges or handles boundary indices via `take_along_dim` which may require padding $t_1$ or handling out-of-bounds indices explicitly [G].
- If `num_samples` for query points is 1, the output array size becomes 0. This is a valid edge case but requires careful handling in downstream code [G].
- Precision issues may arise if $t_1$ has very small intervals or if $y_1$ values vary significantly in magnitude. Using `use_avg=True` helps normalize the output but introduces division by small $S_w$ if query intervals are tiny [S1, unofficial].

## Subtopics

### [Max Dilation](8984-max-dilation.md)
Max Dilation is a resampling technique for step functions that expands the support of non-negative values using max-pooling logic, effectively "blurring" sharp boundaries by considering neighboring intervals. It differs from standard sum-based resampling ([Step Function Resampling](7924-step-function-resampling.md)) by preserving the maximum value within an expanded kernel rather than aggregating mass, which is useful for preventing aliasing or smoothing discrete density fields without introducing negative artifacts [8984-max-dilation.md].

### Interval Sampling
Interval Sampling is a variant of step function resampling that samples intervals (rather than points) from a step function by reflecting midpoints around sampled centers. It ensures that the sampled intervals are centered around the query points, providing a more accurate representation of the function's behavior over the interval.

**Steps:**
1. Sample centers using `sample(..., deterministic_center=True)` [S1, unofficial].
2. Compute midpoints between adjacent centers:
   $$ \text{mid} = \frac{\text{centers}[:, 1:] + \text{centers}[:, :-1]}{2} $$
3. Compute the first fencepost by reflecting the first midpoint around the first center, clamped to `minval`:
   $$ \text{first} = \max(\text{minval}, 2 \cdot \text{centers}[:, :1] - \text{mid}[:, :1]) $$
4. Compute the last fencepost by reflecting the last midpoint around the last center, clamped to `maxval`:
   $$ \text{last} = \min(\text{maxval}, 2 \cdot \text{centers}[:, -1:] - \text{mid}[:, -1:]) $$
5. Concatenate first, mid, and last to form the sample points $t_{\text{samples}}$:
   $$ t_{\text{samples}} = [\text{first}, \text{mid}, \text{last}] $$
   [S1, unofficial].

**Pitfall:**
- Raises `ValueError` if `num_samples <= 1` [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \log M + M)$, where $N$ is the number of steps in the original step function and $M$ is the number of query points. The dominant operations are the cumulative sum ($O(N)$) and the binary search for insertion indices ($O(M \log N)$) [S1, unofficial].
- **Memory Complexity:** $O(N + M)$ for storing the cumulative sum array and indices [S1, unofficial].
- **Strengths:** Exact preservation of mass (sum) or average value; efficient for large datasets; handles arbitrary query intervals.
- **Limitations:** Discontinuous output if `use_avg=False` and query intervals do not align with step boundaries; requires careful handling of boundary conditions; `use_avg=True` involves additional resampling steps for widths and weighted values, increasing constant factors [S1, unofficial].
- **When to use:** When mapping discrete, piecewise-constant data (e.g., transfer functions, density fields) to a different resolution while preserving integral properties.
- **When not to use:** When smooth interpolation is required (use [Piecewise Linear Interpolation (Lookup Table)](2643-piecewise-linear-interpolation-lookup-table.md)) or when kernel-based smoothing is preferred (use [Blurring a Step Function](4383-blurring-a-step-function.md)) [S1, unofficial].

## Background
Step Function Resampling is a fundamental technique in volume rendering and computer graphics for handling discrete data representations. It is particularly relevant in the context of transfer function classification and density field visualization, where accurate mapping of values to sampling grids is crucial for visual quality and performance. The method is implemented in libraries such as NVlabs/InstantSplat for efficient processing of 3D data [S1, unofficial].

## Sources
[S1] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
