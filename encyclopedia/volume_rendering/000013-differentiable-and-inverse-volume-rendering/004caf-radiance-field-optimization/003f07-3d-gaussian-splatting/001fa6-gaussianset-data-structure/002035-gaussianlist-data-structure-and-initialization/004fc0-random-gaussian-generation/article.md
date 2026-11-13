# Random Gaussian Generation

Also called: Gaussian Generation Dataset, Random Gaussian Sampling

## Overview
Random Gaussian Generation is the initialization routine for the `GaussianList` data structure in 3D Gaussian Splatting, creating a set of $N$ isotropic or anisotropic Gaussian primitives with random spatial parameters before scene-specific optimization begins [S1, unofficial]. It differs from sibling methods like LinPrim Population Control, which manages density during training, and Octree-GS, which uses hierarchical spatial structures for initialization; instead, this method provides a uniform, unstructured starting point for radiance-field optimization [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Inputs**:
  - `num_samples` (int): Total number of Gaussians to generate. Default: 1000 [S1, unofficial].
  - `num_points` (int): Number of points per Gaussian (used in some variants, default 144) [S1, unofficial].
  - `min_scale` (float): Lower bound for the uniform distribution of scale logarithms. Default: -12 [S1, unofficial].
  - `max_scale` (float): Upper bound for the uniform distribution of scale logarithms. Default: -2 [S1, unofficial].
  - `min_opacity` (float): Lower bound for the uniform distribution of opacity logits. Default: -5 [S1, unofficial].
  - `max_opacity` (float): Upper bound for the uniform distribution of opacity logits. Default: 10 [S1, unofficial].
  - `sh_degree` (int): Spherical Harmonics degree. Default: 3 [S1, unofficial].
  - `return_type` (str): Output format ('gaussian', 'param', or 'both'). Default: 'gaussian' [S1, unofficial].
- **Outputs**: A collection of Gaussian objects or parameter tensors containing centroid, scale, rotation, opacity, and SH coefficients [S1, unofficial].

**Data Structures**
Each Gaussian primitive is defined by:
1.  **Centroid**: 3D vector $\mathbf{c} = [0, 0, 0]$ (local origin, transformed later) [S1, unofficial].
2.  **Scale**: 3D vector $\mathbf{s} = [s_x, s_y, s_z]$ [S1, unofficial].
3.  **Rotation**: Quaternion $\mathbf{q}$ derived from a random rotation matrix [S1, unofficial].
4.  **Opacity**: Scalar $\alpha$ (logit space) [S1, unofficial].
5.  **SH Coefficients**: Vector $\mathbf{c}_{SH}$ containing DC and higher-order coefficients [S1, unofficial].

**Steps**
1.  **Generate Scale**: For each Gaussian $i$, sample $s_x, s_y, s_z$ independently from a uniform distribution $U(\text{min\_scale}, \text{max\_scale})$ [S1, unofficial].
2.  **Generate Rotation**:
    - Create a $3 \times 3$ matrix $M$ with entries drawn from a standard normal distribution $\mathcal{N}(0, 1)$ [S1, unofficial].
    - Compute the QR decomposition of $M$: $M = QR$ [S1, unofficial].
    - Take the $Q$ factor as the rotation matrix $R$ [S1, unofficial].
    - Convert $R$ to a quaternion $\mathbf{q}$ using `rotmat2qvec(R)` [S1, unofficial].
3.  **Generate Opacity**: Sample opacity logit $\alpha$ from $U(\text{min\_opacity}, \text{max\_opacity})$ [S1, unofficial].
4.  **Generate SH Coefficients**:
    - For each color channel (R, G, B) and each degree $d$ from 0 to `sh_degree`:
        - Calculate variance $\sigma_d^2 = \frac{1}{4^d}$ [S1, unofficial].
        - Sample $2d+1$ coefficients from $\mathcal{N}(0, \sigma_d^2)$ [S1, unofficial].
    - If `sh_degree` < max possible degree, fill remaining coefficients with noise from $\mathcal{N}(0, 0.05^2)$ [S1, unofficial].
    - Concatenate coefficients into $\mathbf{c}_{SH}$ [S1, unofficial].
5.  **Construct Gaussian**: Assemble the primitive with centroid at origin, scale $\mathbf{s}$, rotation $\mathbf{q}$, opacity $\alpha$, and SH coefficients $\mathbf{c}_{SH}$ [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch

def random_gaussian_generation(num_samples=1000, min_scale=-12, max_scale=-2,
                               min_opacity=-5, max_opacity=10, sh_degree=3):
    """
    Generates random Gaussian primitives for initialization.
    """
    gaussians = []
    
    for _ in range(num_samples):
        # 1. Scale: Uniform in log-space
        scale = torch.rand(3) * (max_scale - min_scale) + min_scale
        
        # 2. Rotation: QR decomposition of standard normal matrix
        R_matrix = torch.randn(3, 3)
        Q, _ = torch.linalg.qr(R_matrix)
        # Convert rotation matrix to quaternion [G]
        # Standard conversion assuming column-major or row-major consistency
        # Here we assume a helper rotmat2qvec exists or implement standard conversion
        q = rotmat2qvec(Q)
        
        # 3. Opacity: Uniform in logit space
        opacity = torch.rand(1) * (max_opacity - min_opacity) + min_opacity
        
        # 4. SH Coefficients
        sh_coeffs = []
        for d in range(sh_degree + 1):
            variance = 1.0 / (4 ** d)
            num_coeffs = 2 * d + 1
            # Sample from normal distribution with mean 0 and variance variance
            band_coeffs = torch.randn(num_coeffs) * np.sqrt(variance)
            sh_coeffs.append(band_coeffs)
        
        # Fill remaining if needed (assuming max degree is higher than sh_degree)
        # [G] Implementation detail: if sh_degree < max, fill with std 0.05 noise
        # This step depends on the specific max degree expected by the renderer
        
        feat_dc = torch.stack([sh_coeffs[0][0] for _ in range(3)]) # Simplified for R,G,B DC
        # In practice, feat_dc is [coeff_r[0], coeff_g[0], coeff_b[0]]
        # feat_extra is concatenated higher-order coefficients
        
        # 5. Construct Gaussian
        # Centroid is [0,0,0] locally
        centroid = torch.zeros(3)
        
        gaussian = {
            'centroid': centroid,
            'scale': scale,
            'rotation': q,
            'opacity': opacity,
            'feat_dc': feat_dc,
            'feat_extra': torch.cat(sh_coeffs[1:]) if len(sh_coeffs) > 1 else torch.tensor([])
        }
        gaussians.append(gaussian)
        
    return gaussians

def rotmat2qvec(R):
    """
    Converts a 3x3 rotation matrix to a quaternion [w, x, y, z].
    Standard implementation [G].
    """
    m00, m01, m02 = R[0, 0], R[0, 1], R[0, 2]
    m10, m11, m12 = R[1, 0], R[1, 1], R[1, 2]
    m20, m21, m22 = R[2, 0], R[2, 1], R[2, 2]
    
    trace = m00 + m11 + m22
    if trace > 0:
        s = 0.5 / torch.sqrt(trace + 1.0)
        w = 0.25 / s
        x = (m21 - m12) * s
        y = (m02 - m20) * s
        z = (m10 - m01) * s
    elif (m00 > m11) and (m00 > m22):
        s = 2.0 * torch.sqrt(1.0 + m00 - m11 - m22)
        w = (m21 - m12) / s
        x = 0.25 * s
        y = (m01 + m10) / s
        z = (m02 + m20) / s
    elif m11 > m22:
        s = 2.0 * torch.sqrt(1.0 + m11 - m00 - m22)
        w = (m02 - m20) / s
        x = (m01 + m10) / s
        y = 0.25 * s
        z = (m12 + m21) / s
    else:
        s = 2.0 * torch.sqrt(1.0 + m22 - m00 - m11)
        w = (m10 - m01) / s
        x = (m02 + m20) / s
        y = (m12 + m21) / s
        z = 0.25 * s
        
    return torch.tensor([w, x, y, z])
```

**Edge Cases and Pitfalls**
- **Precision**: The QR decomposition may produce non-orthogonal matrices if $M$ is singular or near-singular, though probability is low with standard normal entries [S1, unofficial].
- **Boundary Conditions**: The uniform distributions for scale and opacity are inclusive of bounds [S1, unofficial].
- **Degenerate Input**: If `num_samples` is 0, return empty list [G].

## Subtopics

### LinPrim Population Control
Manages Gaussian density during training by splitting/duplicating primitives based on gradient magnitude. Choose when dynamic resolution is needed. [Link](2295-linprim-population-control.md)

### Gaussian Splatting Initialization and Parameterization
Covers broader initialization strategies and parameter encoding (e.g., log-space vs. direct). Choose when comparing different starting distributions. [Link](2313-gaussian-splatting-initialization-and-parameterization.md)

### Octree-GS / Scaffold-GS
Uses hierarchical spatial structures for initialization. Choose when scene complexity requires adaptive density. [Link](6142-octree-gs-scaffold-gs.md)

## Operating characteristics
- **Time Complexity**: $O(N \cdot D^2)$ where $N$ is `num_samples` and $D$ is `sh_degree` (for SH coefficient generation) [S1, unofficial]. The QR decomposition is $O(1)$ per Gaussian (fixed 3x3 matrix).
- **Memory Complexity**: $O(N \cdot D^2)$ to store SH coefficients [S1, unofficial].
- **Measured Performance**: Not given in the sources.
- **Strengths**: Simple, fast, and provides a uniform starting distribution for optimization [S1, unofficial].
- **Limitations**: Does not account for scene geometry, potentially leading to inefficient initial coverage [S1, unofficial].
- **When to Use**: Standard initialization for 3D Gaussian Splatting when no prior scene information is available [S1, unofficial].
- **When Not to Use**: When scene-specific initialization (e.g., from point clouds) or hierarchical structures are required [S1, unofficial].

## Background
The method originates from the implementation details of 3D Gaussian Splatting, specifically within the `gs-embedding` repository by cilix-ai [S1, unofficial]. It serves as a baseline initialization for radiance-field optimization, allowing the Gaussian primitives to be optimized towards a target image or scene [S1, unofficial].

## Sources
[S1] cilix-ai/gs-embedding, https://github.com/cilix-ai/gs-embedding, fetched 2026-10-27 (unofficial)
