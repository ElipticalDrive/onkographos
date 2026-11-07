# Error Data Generation
Also called: Adaptive Resolution Error Measure
## Overview
Error Data Generation computes a quantitative measure of the approximation error introduced by the Adaptive Resolution (AR) data reduction process [S1, unofficial]. It solves the problem of quantifying data loss during the transition from fine-grained volume data to a coarser, adaptive representation, enabling the inclusion of error metrics in subsequent isosurface rendering pipelines [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: A set of coarse data values $V_c$ and their corresponding fine-resolution child values $V_f$.
- Output: A set of error values $E$ associated with each coarse node.

**Data Structures:**
The error data is stored alongside the adaptive resolution data [S1, unofficial]. For each coarse node $i$, an error value $E_i$ is computed [S1, unofficial]. The layout is a scalar field aligned with the coarse grid nodes [G].

**Steps:**
1. **Identify Child Nodes:** For each coarse data point (parent node) in the adaptive resolution representation, identify the 8 child nodes in the next finer resolution level of the octree [S1, unofficial].
2. **Compute Individual Errors:** For each of the 8 child nodes $j$, calculate the difference between the child's original value $v_{f,j}$ and the parent's coarse value $v_c$ [S1, unofficial].
3. **Aggregate Error:** Compute the error for the coarse point $E_i$ as the average of the errors of its 8 children:
   $$ E_i = \frac{1}{8} \sum_{j=1}^{8} (v_{f,j} - v_c)^2 $$
   This represents the mean square error of representing the 8 original fine values with the single coarse value [S1, unofficial].
4. **Store Error:** Associate $E_i$ with the coarse node for use in rendering [S1, unofficial].

**Parameters:**
- The divisor 8 is fixed by the octree structure (binary subdivision in 3D) [S1, unofficial].

**Edge Cases and Pitfalls:**
- If a coarse node has no children (leaf in the fine grid or boundary condition), the error is undefined or zero depending on implementation [G].
- Precision issues may arise if the range of values is large; normalization might be required for rendering [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_error_data(coarse_fine_pairs):
    """
    coarse_fine_pairs: list of tuples (v_c, [v_f1, ..., v_f8])
    Returns: list of tuples (v_c, E)
    """
    error_data = []
    for v_c, children in coarse_fine_pairs:
        sum_sq = 0.0
        for v_f in children:
            diff = v_f - v_c
            sum_sq += diff * diff
        error_data.append((v_c, sum_sq / 8.0))
    return error_data
```
## Subtopics

### [Hierarchical Error Metric Calculation](6607-hierarchical-error-metric-calculation.md)
A fast method to calculate error values at every tree node based on the mean square error (MSE) between a parent node's low-resolution data and its eight immediate child nodes, taking the maximum error of the children into account. This avoids the high I/O and computational cost of comparing against raw data.

## Operating characteristics
**Time and Memory Complexity:**
- Time: Not given in the sources [S1, unofficial].
- Memory: Not given in the sources [S1, unofficial].

**Measured Performance:**
- Not given in the sources [S1, unofficial].

**Scaling:**
- Not given in the sources [S1, unofficial].

**Strengths and Limitations:**
- Strength: Simple to compute and provides a direct measure of data fidelity [S1, unofficial].
- Limitation: Not given in the sources [S1, unofficial].

**When to Use:**
- When isosurface rendering quality needs to be improved by incorporating error metrics [S1, unofficial].

**When Not to Use:**
- Not given in the sources [S1, unofficial].
## Background
**Origin:**
- Not given in the sources [S1, unofficial].

**History and Development:**
- The method is part of the Adaptive Resolution (AR) data representation framework, which aims to reduce memory space and computation time during rendering by eliminating redundant cubes from the data set [S1, unofficial].

**Where it is used:**
- It is used in IsoSurface Rendering of Adaptive Resolution Data [S1, unofficial].
## Sources
[S1] IsoSurface Rendering of Adaptive Resolution Data, https://www.cs.unh.edu/~rdb/papers/ARiso.pdf, fetched 2026-10-29 (unofficial)
