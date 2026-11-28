# Weighted Normal Accumulation

Also called: Normals Rendering, NormalsRenderer, Transmittance-Weighted Normal Estimation

## Overview
Weighted Normal Accumulation estimates surface normals for volume rendering by aggregating per-sample gradient normals along a ray, weighted by their transmittance contributions [S2, unofficial].

## How it works
**Inputs:**
- `normals`: A tensor of shape `(num_samples, 3)` containing the estimated normal vector for each sample along the ray [S1, S2, unofficial].
- `weights`: A tensor of shape `(num_samples,)` containing the transmittance-based weight for each sample [S1, S2, unofficial].

**Outputs:**
- `n`: A tensor of shape `(3,)` representing the accumulated normal vector [S1, unofficial].

**Data Structures:**
- `normals`: Float32 array, 3 components per sample [S1, unofficial].
- `weights`: Float32 array, 1 component per sample [S1, unofficial].

**Steps:**
1. Initialize the accumulated normal vector $n$ to zero [G].
2. Compute $n = \sum (weights \times normals)$ [S1, unofficial].
3. If `normalize` is True, normalize the resulting vector $n$ [S1, unofficial].

**Formula:**
The accumulated normal $n$ is computed as:
$$
n = \sum_{i=0}^{N-1} w_i \mathbf{n}_i
$$
Where:
- $w_i$ is the transmittance weight for sample $i$ [S1, unofficial].
- $\mathbf{n}_i$ is the normal vector for sample $i$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def weighted_normal_accumulation(normals, weights, normalize=True):
    acc = [0.0, 0.0, 0.0]
    for w, n in zip(weights, normals):
        acc[0] += w * n[0]
        acc[1] += w * n[1]
        acc[2] += w * n[2]
    if normalize:
        length = (acc[0]**2 + acc[1]**2 + acc[2]**2)**0.5
        if length > 1e-8:
            acc[0] /= length
            acc[1] /= length
            acc[2] /= length
    return acc
```

**Parameters:**
- `normalize`: Boolean, default `True`. Determines if the result is unit-length [S1, unofficial].

**Edge Cases:**
- If all weights are zero, the accumulated normal is zero. Normalization should handle this by returning a default or leaving it as zero [G].
- Precision issues may arise if weights vary significantly in magnitude; using higher precision for accumulation is recommended [G].

## Subtopics

### Transmittance-Weighted Normal Accumulation
This variant computes the rendered normal map $N_{rendered}(u)$ for a pixel $u$ by summing transmittance-weighted opacity and normals along the ray: $N_{rendered}(u) = \sum_{i=1}^{N_{ray}} T_i \alpha_i n_i$, where $T_i$ is transmittance, $\alpha_i$ is opacity, and $n_i$ is the normal at sample $i$ [S3, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] nerfstudio-project/nerfstudio, https://github.com/nerfstudio-project/nerfstudio, fetched 2026-10-26 (unofficial)
[S2] autonomousvision/sdfstudio, https://github.com/autonomousvision/sdfstudio, fetched 2026-10-26 (unofficial)
[S3] SVRecon: Sparse Voxel Rasterization for Surface Reconstruction, https://doi.org/10.48550/arxiv.2511.17364, fetched 2026-10-26 (unofficial)
