# Adjacent Gradient Difference

Also called: Gradient Magnitude Increase Measure

## Overview
Adjacent Gradient Difference is a volume rendering modulation technique that calculates an importance difference by comparing the gradient magnitudes of two adjacent samples along a viewing ray. It increases accumulated transparency whenever an increase in gradient magnitude is detected, distinguishing it from methods that rely on absolute gradient magnitude or scalar intensity changes.

## How it works
**Inputs and Outputs:**
*   **Input:** The gradient magnitude of the current sample, $\text{grad}_i$, and the gradient magnitude of the previous sample along the ray, $\text{grad}_{i-1}$.
*   **Output:** A modulation factor $\beta \in [0, 1]$ used to adjust accumulated transparency.

**Data Structures:**
No specific complex data structures are required beyond the scalar values $\text{grad}_i$ and $\text{grad}_{i-1}$ available during ray marching.

**Steps:**
1.  Retrieve the gradient magnitude of the current sample, $\text{grad}_i$.
2.  Retrieve the gradient magnitude of the previous sample, $\text{grad}_{i-1}$.
3.  Calculate the difference $\Delta = \text{grad}_i - \text{grad}_{i-1}$.
4.  Compute the modulation factor $\beta$ using the piecewise function defined below.

**Formula:**
The modulation factor $\beta$ is calculated as [S1]:

$$
\beta = B(\Delta_{imp}(\text{grad}_i, \text{grad}_{i-1})) = 
\begin{cases} 
1 - (\text{grad}_i - \text{grad}_{i-1}) & \text{if } \text{grad}_i > \text{grad}_{i-1} \\
1 & \text{otherwise}
\end{cases}
$$

Where:
*   $\text{grad}_i$ is the current sample's gradient magnitude.
*   $\text{grad}_{i-1}$ is the previous sample's gradient magnitude.
*   The term $(\text{grad}_i - \text{grad}_{i-1})$ represents the increase in gradient magnitude.

**Parameters:**
Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Dynamic Range:** The method tends to produce a wider dynamic range output, particularly if the dataset contains many non-transparent voxels. Tone-mapping (e.g., brightness/contrast and gamma curve) is required to map the high dynamic range output to the normal display range [S1].
*   **Transparency:** Because more samples along the viewing ray contribute to the final result, the rendered image has a more transparent look compared to maximum gradient magnitude measures [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adjacent_gradient_difference(grad_i, grad_prev):
    """
    Calculate the modulation factor beta based on adjacent gradient magnitudes.
    
    Args:
        grad_i (float): Gradient magnitude of the current sample.
        grad_prev (float): Gradient magnitude of the previous sample.
        
    Returns:
        float: Modulation factor beta in [0, 1].
    """
    if grad_i > grad_prev:
        # Increase in gradient magnitude detected
        # Beta decreases as the increase grows, but is bounded by 0 implicitly 
        # if the difference exceeds 1, though standard gradient magnitudes 
        # often normalize or scale such that differences are small.
        # Note: The source formula does not explicitly clamp the result to 0,
        # but physically beta represents a weight/factor, so it should likely be clamped.
        # However, strictly following the source formula [S1]:
        beta = 1.0 - (grad_i - grad_prev)
        # [G] Implementation note: In practice, beta should be clamped to [0, 1] 
        # to prevent negative weights or values > 1 if the gradient difference is large.
        return max(0.0, min(1.0, beta))
    else:
        # No increase in gradient magnitude
        return 1.0
```

## Subtopics

### [Scalar Intensity Increase Measure](6014-scalar-intensity-increase-measure.md)
A method using scalar intensity as the importance measure, considering only the difference between two adjacent samples, increasing accumulated transparency whenever there is an increase of scalar intensity. It sets itself apart from its sibling [Adjacent Gradient Difference](5787-adjacent-gradient-difference.md) by operating on raw scalar values rather than gradient magnitudes, making it sensitive to absolute intensity changes rather than rate-of-change changes.

## Operating characteristics
**Time and Memory Complexity:**
The computation is $O(1)$ per sample, requiring only a subtraction and a comparison. Memory complexity is $O(1)$ as it only requires storing the previous sample's gradient magnitude.

**Measured Performance:**
Not given in the sources.

**Scaling:**
Not given in the sources.

**Strengths and Limitations:**
*   **Strengths:** It produces a more transparent look than maximum gradient magnitude measures, allowing more samples along the viewing ray to contribute to the final result [S1]. It can better reveal sharp features of datasets compared to direct volume rendering [S1].
*   **Limitations:** It tends to have a wider dynamic range, requiring tone-mapping for display [S1].

**When to use it and when not:**
Use it when a more transparent visualization is desired and sharp features need to be highlighted without the opacity saturation of maximum gradient magnitude projection. It is suitable for fast data exploration when combined with tone-mapping [S1].

## Background
The method was introduced in the context of importance-based accumulated transparency modulation for fast volumetric data exploration. It is derived from the observation that calculating importance difference by two adjacent samples provides distinct visual characteristics compared to maximum gradient magnitude projection [S1].

## Sources
[S1] Fast Volumetric Data Exploration with Importance-Based Accumulated Transparency Modulation, https://doi.org/10.2312/vg/vg10/061-068, fetched 2026-10-26
