# Adaptive Multi-resolution LOD for Vertex Features

Also called: Adaptive Vertex Feature LOD, Adaptive Virtual Vertex Allocation, Multi-resolution Surface Learnable Feature

## Overview
This method dynamically refines mesh-based surface features during training by allocating virtual vertices in regions with high reconstruction loss, enabling the capture of high-frequency illumination details in initially sparse mesh areas [S1, unofficial]. It sets itself apart from manual subdivision by achieving superior detail preservation using the original sparse mesh as input while requiring less memory, and differs from static multi-resolution grids by concentrating computation only on actual scene geometry rather than unnecessary space [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A mesh with initial vertex features, training loss statistics per triangle, and triangle radiance values [S1, unofficial].
- Output: Updated per-triangle Level of Detail (LOD) factors $k_i$ that control the density of virtual vertices [S1, unofficial].

**Data Structures:**
- Per-triangle data:
  - $L(i)$: Average training loss for triangle $i$ over a training interval [S1, unofficial].
  - $R(i)$: Radiance of triangle $i$ [S1, unofficial].
  - $k_i$: Current LOD factor for triangle $i$ [S1, unofficial].
  - $L'_{mean}$: Mean of adjusted loss values across all triangles [S1, unofficial].
  - $L'_{std}$: Standard deviation of adjusted loss values across all triangles [S1, unofficial].

**Steps:**
1. **Compute Average Loss:** Calculate the average loss $L(i)$ for each triangle $i$ over a training interval [S1, unofficial].
2. **Adjust Loss for Radiance:** Compute the adjusted loss $L(i)'$ for each triangle to account for radiance intensity, using the formula:
   $$ L(i)' = L(i) \times \log(R(i) + 1) $$
   where $R(i)$ is the radiance of triangle $i$ [S1, unofficial].
3. **Calculate Statistics:** Compute the mean $L'_{mean}$ and standard deviation $L'_{std}$ of the adjusted loss values $L(i)'$ across all triangles [S1, unofficial].
4. **Update LOD Factor:** For each triangle $i$, update the LOD factor $k_i$ if the adjusted loss exceeds the mean plus two standard deviations:
   $$ k_i = k_i + \frac{L(i)' - L'_{mean}}{L'_{std}}, \quad \text{if } L(i)' > L'_{mean} + 2 \cdot L'_{std} $$
   Otherwise, $k_i$ remains unchanged [S1, unofficial].
5. **Allocate Virtual Vertices:** Use the updated $k_i$ to determine the number of virtual vertices to allocate within triangle $i$ for feature encoding [S1, unofficial].

**Parameters:**
- Threshold multiplier: 2 (used in the condition $L'_{mean} + 2 \cdot L'_{std}$) [S1, unofficial].
- Training interval: Not specified in the sources [G].

**Edge Cases and Pitfalls:**
- **Discontinuities:** When adjacent triangles use different LOD levels, features across shared edges may introduce visible seams, particularly in early training stages (e.g., steps 200 and 500) [S1, unofficial]. These seams become less noticeable as training progresses (e.g., steps 5,000 and 10,000) [S1, unofficial].
- **Precision:** The logarithmic function $\log(R(i) + 1)$ ensures that triangles with zero radiance do not distort the loss adjustment [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def update_lod_factors(triangles):
    """
    triangles: List of triangle objects, each with attributes:
               - loss: float (L(i), already averaged over the training interval)
               - radiance: float (R(i))
               - lod_factor: float (k_i)
    Returns: Updated list of triangles with new lod_factors.
    """
    adjusted_losses = []
    for tri in triangles:
        l_prime = tri.loss * math.log(tri.radiance + 1)
        adjusted_losses.append(l_prime)

    if not adjusted_losses:
        return triangles

    mean_l_prime = sum(adjusted_losses) / len(adjusted_losses)
    variance = sum((x - mean_l_prime) ** 2 for x in adjusted_losses) / len(adjusted_losses)
    std_l_prime = math.sqrt(variance)

    for i, tri in enumerate(triangles):
        l_prime = adjusted_losses[i]
        threshold = mean_l_prime + 2 * std_l_prime

        if l_prime > threshold:
            delta = (l_prime - mean_l_prime) / std_l_prime
            tri.lod_factor += delta

    return triangles
```

## Subtopics

### Adaptive LOD Update
This variant refers to the specific mechanism of computing the average loss $L(i)$ for each triangle $i$ over a training interval and adjusting the LOD factor based on the adjusted loss formula $L(i)' = L(i) \times \log(R(i) + 1)$ [S1, unofficial]. It is the core update rule for the method [S1, unofficial].

## Operating characteristics
- **Memory Efficiency:** The method achieves significant memory savings [S1, unofficial]. In the Dining Room scene, it uses 4.8 MB compared to 6.5 MB for a dense mesh with no LOD and 4.2 MB for a sparse mesh with no LOD [S1, unofficial]. It achieves over a 5× reduction in memory overhead compared to state-of-the-art methods while maintaining comparable image quality [S1, unofficial].
- **Performance:** The method demonstrates superior detail preservation using the original sparse mesh as input with less memory than manual subdivision [S1, unofficial].
- **Strengths:** Concentrates computation on actual scene geometry, reducing memory overhead compared to multiresolution grids [S1, unofficial]. Effectively identifies underrepresented areas (e.g., low initial mesh density) and adaptively allocates virtual vertices [S1, unofficial].
- **Limitations:** Visible seams may appear along triangle boundaries when adjacent triangles have different LOD levels, especially in early training stages [S1, unofficial].

## Background
This method was introduced in the paper "Vertex Features for Neural Global Illumination" by Rui Su, Honghao Dong, Haojie Jin, Yisong Chen, Guoping Wang, and Sheng Li in 2026 (unofficial preprint) [S1, unofficial]. It is used in neural rendering applications such as Neural Radiosity, dynamic neural radiosity (DNR), and neural path guiding [S1, unofficial]. The approach builds on the concept of neural vertex features, which store spatial information at mesh vertices to offer a memory-efficient solution for neural rendering [S1, unofficial].

## Sources
[S1] [2508.07852] Vertex Features for Neural Global Illumination, https://arxiv.org/abs/2508.07852, fetched 2026-10-27 (unofficial)
