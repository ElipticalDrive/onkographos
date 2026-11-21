# Weighted Normal Accumulation

Also called: Normals Rendering, NormalsRenderer, Transmittance-Weighted Normal Estimation

## Overview
Weighted Normal Accumulation estimates surface normals for volume rendering by aggregating per-sample gradient normals along a ray, weighted by their transmittance contributions. Unlike [SDF Normal Estimation via Finite Differences](4619-sdf-normal-estimation-via-finite-differences.md), which computes a single normal per ray intersection, this method accumulates normals from all samples, providing a robust estimate in regions with ambiguous or noisy distance fields. It differs from [Lambertian Shading with Gradient Normals](1229-lambertian-shading-with-gradient-normals.md) by explicitly using transmittance weights rather than just geometric shading factors.

## How it works
**Inputs:**
- `normals`: A tensor of shape `(num_samples, 3)` containing the estimated normal vector for each sample along the ray.
- `weights`: A tensor of shape `(num_samples,)` containing the transmittance-based weight for each sample (typically derived from the volume rendering integral).

**Outputs:**
- `n`: A tensor of shape `(3,)` representing the accumulated normal vector.

**Data Structures:**
- `normals`: Float32 array, 3 components per sample.
- `weights`: Float32 array, 1 component per sample.

**Steps:**
1. Initialize the accumulated normal vector $n$ to zero.
2. For each sample $i$ from $0$ to $N-1$:
   - Compute the contribution: $c_i = w_i \cdot \mathbf{n}_i$, where $w_i$ is the weight and $\mathbf{n}_i$ is the normal.
   - Add $c_i$ to $n$.
3. If normalization is enabled, normalize $n$ to unit length.

**Formula:**
The accumulated normal $n$ is computed as:
$$
n = \sum_{i=0}^{N-1} w_i \mathbf{n}_i
$$
Where:
- $w_i$ is the transmittance weight for sample $i$.
- $\mathbf{n}_i$ is the normal vector for sample $i$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def weighted_normal_accumulation(normals: list, weights: list, normalize: bool = True) -> list:
    """
    Accumulate normals weighted by transmittance.
    
    Args:
        normals: List of 3D vectors (tuples or lists) for each sample.
        weights: List of scalar weights for each sample.
        normalize: Whether to normalize the final normal vector.
        
    Returns:
        A 3D vector representing the accumulated normal.
    """
    # Initialize accumulated normal to zero vector
    acc_normal = [0.0, 0.0, 0.0]
    
    # Iterate over samples and accumulate weighted normals
    for i in range(len(weights)):
        w = weights[i]
        n = normals[i]
        # Ensure n is a 3D vector
        acc_normal[0] += w * n[0]
        acc_normal[1] += w * n[1]
        acc_normal[2] += w * n[2]
        
    # Normalize if requested
    if normalize:
        length = (acc_normal[0]**2 + acc_normal[1]**2 + acc_normal[2]**2)**0.5
        if length > 1e-8: # Avoid division by zero
            acc_normal[0] /= length
            acc_normal[1] /= length
            acc_normal[2] /= length
        else:
            # Fallback to a default normal if accumulation is zero
            acc_normal = [0.0, 1.0, 0.0]
            
    return acc_normal
```

**Parameters:**
- `normalize`: Boolean, default `True`. Determines if the result is unit-length.

**Edge Cases:**
- If all weights are zero, the accumulated normal is zero. Normalization should handle this by returning a default or leaving it as zero [G].
- Precision issues may arise if weights vary significantly in magnitude; using higher precision for accumulation is recommended.

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of samples per ray.
- **Memory Complexity:** $O(1)$ auxiliary space, as it accumulates into a single vector.
- **Strengths:** Robust to noise in individual sample normals; provides a smooth normal estimate across the ray.
- **Limitations:** Computationally more expensive than single-sample methods; may blur sharp features if weights are spread out.
- **When to use:** When smooth, consistent normals are required for shading in volume rendering, especially in noisy or ambiguous regions.
- **When not to use:** When performance is critical and single-sample approximations are sufficient.

## Background
This method is implemented in [SDFStudio](https://github.com/autonomousvision/sdfstudio) [S2, unofficial] and [Nerfstudio](https://github.com/nerfstudio-project/nerfstudio) [S1, unofficial] for normal estimation in differentiable volume rendering. It leverages the transmittance weights from the volume rendering integral to prioritize normals from samples near the surface.

## Sources
[S1] nerfstudio-project/nerfstudio, https://github.com/nerfstudio-project/nerfstudio, fetched 2026-10-26 (unofficial)
[S2] autonomousvision/sdfstudio, https://github.com/autonomousvision/sdfstudio, fetched 2026-10-26 (unofficial)
