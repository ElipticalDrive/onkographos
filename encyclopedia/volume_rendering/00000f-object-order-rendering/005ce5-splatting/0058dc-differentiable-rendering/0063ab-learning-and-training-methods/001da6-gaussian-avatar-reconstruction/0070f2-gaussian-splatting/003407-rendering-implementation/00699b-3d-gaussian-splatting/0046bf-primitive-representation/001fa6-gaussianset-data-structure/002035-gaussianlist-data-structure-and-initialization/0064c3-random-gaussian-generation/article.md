# Random Gaussian Generation

Also called: Gaussian Generation Dataset, Random Gaussian Cloud Generation, Random Gaussian Sampling

## Overview
Random Gaussian Generation is the initialization procedure for creating a set of 3D Gaussians from scratch, typically to serve as the starting point for differentiable rendering optimization in methods like 3D Gaussian Splatting [S2, unofficial]. It generates a fixed number of Gaussian primitives with random positions, scales, orientations, opacities, and spherical harmonics coefficients [S2, unofficial]. It differs from [LinPrim Population Control](2295-linprim-population-control.md) which manages the density of existing Gaussians during training, and from [Gaussian Splatting Initialization and Parameterization](2313-gaussian-splatting-initialization-and-parameterization.md) which may use structured or data-driven initialization strategies [G]. This method provides a uniform, uninformative prior over the parameter space [G].

## How it works
The method generates a set of $N$ Gaussian objects [S2, unofficial]. Each Gaussian is defined by a centroid, a scale vector, a rotation quaternion, an opacity scalar, and spherical harmonics (SH) coefficients for color [S2, unofficial].

### Inputs and Outputs
**Inputs:**
- `num_samples`: Total number of Gaussians to generate. Default: 1000 [S2, unofficial].
- `num_points`: Number of points per Gaussian (used for internal representation or visualization, not stored in the final Gaussian object). Default: 144 ($12 \times 12$) [S2, unofficial].
- `min_scale`: Lower bound for the uniform distribution of scale components. Default: -12 [S2, unofficial].
- `max_scale`: Upper bound for the uniform distribution of scale components. Default: -2 [S2, unofficial].
- `min_opacity`: Lower bound for the uniform distribution of opacity. Default: -5 [S2, unofficial].
- `max_opacity`: Upper bound for the uniform distribution of opacity. Default: 10 [S2, unofficial].
- `sh_degree`: Maximum spherical harmonics degree to generate. Default: 3 [S2, unofficial].
- `return_type`: Output format. Options: `'gaussian'`, `'param'`, or `'both'`. Default: `'gaussian'` [S2, unofficial].
- `bounds`: Tuple of 3 floats defining the spatial range $[-b, b]$ for each dimension. Default: $(2.0, 2.0, 2.0)$ [S1, unofficial].

**Outputs:**
- A list or tensor of Gaussian objects/parameters [S2, unofficial]. If `return_type` is `'gaussian'`, returns full Gaussian objects [S2, unofficial]. If `'param'`, returns raw parameter tensors [S2, unofficial]. If `'both'`, returns both [S2, unofficial].

### Data Structures
Each Gaussian object contains:
- `centroid`: 3D vector representing the initial position [S1, unofficial][S2, unofficial].
- `scale`: 3D vector $[s_x, s_y, s_z]$ [S2, unofficial].
- `rotation`: Quaternion $q$ derived from a rotation matrix [S2, unofficial].
- `opacity`: Scalar $\alpha$ [S2, unofficial].
- `feat_dc`: 3D vector containing DC SH coefficients for R, G, B [S2, unofficial].
- `feat_extra`: Concatenated higher-order SH coefficients [S2, unofficial].

### Steps
1. **Initialize Position**: Sample $x, y, z$ independently from a uniform distribution over the bounding box defined by `bounds` [S1, unofficial]. (Note: Some implementations fix the centroid at $[0, 0, 0]$ and apply transformations later [S2, unofficial]).
2. **Initialize Scale**: Sample $s_x, s_y, s_z$ independently from a uniform distribution $U(\text{min\_scale}, \text{max\_scale})$ [S2, unofficial]. (Alternative fixed distribution: $s = |\mathcal{N}(0, 0.03) \times 0.03 + 0.04|$ [S1, unofficial]).
3. **Initialize Rotation**:
   - Generate a $3 \times 3$ matrix $M$ with entries drawn from a standard normal distribution $\mathcal{N}(0, 1)$ [S2, unofficial].
   - Compute the QR decomposition of $M$: $M = QR$ [S2, unofficial].
   - Take the $Q$ factor as the rotation matrix $R_{mat}$ [S2, unofficial].
   - Convert $R_{mat}$ to a quaternion $q$ using a standard conversion function (e.g., `rotmat2qvec`) [S2, unofficial].
4. **Initialize Opacity**: Sample opacity $\alpha_i$ from a uniform distribution $U(\text{min\_opacity}, \text{max\_opacity})$ [S2, unofficial]. (Alternative clipped distribution: $\alpha = \text{clip}(\mathcal{U}(0, 1) \times 0.5 + 0.5, 0, 1)$ [S1, unofficial]). Note: The parameterized range $[-5, 10]$ indicates logit space, requiring a sigmoid activation if interpreted as a probability [G].
5. **Initialize Spherical Harmonics Coefficients**:
   - For each color channel (R, G, B) and each degree $d$ from $0$ to $\text{sh\_degree}$:
     - Calculate variance $\sigma_d^2 = \frac{1}{4^d}$ [S2, unofficial].
     - Sample $(2d+1)$ coefficients from a normal distribution $\mathcal{N}(0, \sigma_d)$ [S2, unofficial].
   - If $\text{sh\_degree} < \text{max\_SH\_degree}$, fill the remaining coefficients with normal noise $\mathcal{N}(0, 0.05)$ [S2, unofficial].
   - The DC coefficients ($d=0$) for R, G, B form `feat_dc` [S2, unofficial].
   - Higher-order coefficients ($d > 0$) are concatenated to form `feat_extra` [S2, unofficial].
6. **Construct Gaussian Object**: Assemble the components into a Gaussian object with centroid at $[0,0,0]$, scale $[s_x, s_y, s_z]$, rotation $q$, opacity $\alpha$, `feat_dc`, and `feat_extra` [S2, unofficial].
7. **Repeat**: Steps 1-6 are repeated for $N = \text{num\_samples}$ Gaussians [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def generate_random_gaussians(num_samples=1000, min_scale=-12, max_scale=-2,
                              min_opacity=-5, max_opacity=10, sh_degree=3,
                              bounds=(2.0, 2.0, 2.0), return_type='gaussian'):
    """
    Generates a set of random 3D Gaussians for initialization.
    Follows parameterized generation from [S2, unofficial] with bounds support from [S1, unofficial].
    """
    gaussians = []
    
    for _ in range(num_samples):
        # 1. Generate Position
        pos = (np.random.rand(3) - 0.5) * 2.0 * np.array(bounds)
        
        # 2. Generate Scale
        scale = np.random.uniform(min_scale, max_scale, 3)
        
        # 3. Generate Rotation
        M = np.random.randn(3, 3)
        Q, _ = np.linalg.qr(M)
        # Ensure proper rotation matrix (handle reflection if det(Q) == -1)
        if np.linalg.det(Q) < 0:
            Q[:, 0] *= -1
        rotation = rotmat2qvec(Q)
        
        # 4. Generate Opacity
        opacity = np.random.uniform(min_opacity, max_opacity)
        
        # 5. Generate Spherical Harmonics Coefficients
        feat_dc = []
        feat_extra = []
        
        for d in range(sh_degree + 1):
            variance = 1.0 / (4 ** d)
            std = np.sqrt(variance)
            num_coeffs = 2 * d + 1
            
            # Sample coefficients for R, G, B
            coeffs_r = np.random.normal(0, std, num_coeffs)
            coeffs_g = np.random.normal(0, std, num_coeffs)
            coeffs_b = np.random.normal(0, std, num_coeffs)
            
            if d == 0:
                feat_dc = [coeffs_r[0], coeffs_g[0], coeffs_b[0]]
            else:
                feat_extra.extend(coeffs_r)
                feat_extra.extend(coeffs_g)
                feat_extra.extend(coeffs_b)
        
        # Fill remaining coefficients if sh_degree < max possible (assumed 3)
        max_sh_degree = 3
        if sh_degree < max_sh_degree:
            for d in range(sh_degree + 1, max_sh_degree + 1):
                num_coeffs = 2 * d + 1
                feat_extra.extend(np.random.normal(0, 0.05, num_coeffs))
                feat_extra.extend(np.random.normal(0, 0.05, num_coeffs))
                feat_extra.extend(np.random.normal(0, 0.05, num_coeffs))
        
        # 6. Construct Gaussian Object
        gaussian = {
            'centroid': pos,
            'scale': scale,
            'rotation': rotation,
            'opacity': opacity,
            'feat_dc': np.array(feat_dc),
            'feat_extra': np.array(feat_extra)
        }
        gaussians.append(gaussian)
        
    return gaussians

def rotmat2qvec(R):
    """
    Converts a 3x3 rotation matrix to a quaternion [w, x, y, z].
    Standard conversion algorithm [G].
    """
    m00, m01, m02 = R[0, 0], R[0, 1], R[0, 2]
    m10, m11, m12 = R[1, 0], R[1, 1], R[1, 2]
    m20, m21, m22 = R[2, 0], R[2, 1], R[2, 2]
    
    trace = m00 + m11 + m22
    if trace > 0:
        s = 0.5 / np.sqrt(trace + 1.0)
        w = 0.25 / s
        x = (m21 - m12) * s
        y = (m02 - m20) * s
        z = (m10 - m01) * s
    elif (m00 > m11) and (m00 > m22):
        s = 2.0 * np.sqrt(1.0 + m00 - m11 - m22)
        w = (m21 - m12) / s
        x = 0.25 * s
        y = (m01 + m10) / s
        z = (m02 + m20) / s
    elif m11 > m22:
        s = 2.0 * np.sqrt(1.0 + m11 - m00 - m22)
        w = (m02 - m20) / s
        x = (m01 + m10) / s
        y = 0.25 * s
        z = (m12 + m21) / s
    else:
        s = 2.0 * np.sqrt(1.0 + m22 - m00 - m11)
        w = (m10 - m01) / s
        x = (m02 + m20) / s
        y = (m12 + m21) / s
        z = 0.25 * s
        
    return np.array([w, x, y, z])
```

### Edge Cases and Pitfalls
- **Reflection in Rotation**: The QR decomposition of a random normal matrix can produce a reflection (determinant -1) instead of a rotation [G]. The pseudocode includes a check to flip the first column if `det(Q) < 0` to ensure a proper rotation matrix [G].
- **Opacity Range**: The default opacity range $[-5, 10]$ suggests logit space [S2, unofficial]. If used directly as probability, it will cause numerical issues [G]. It should be passed through a sigmoid function if interpreted as probability [G].
- **SH Variance**: The variance decreases exponentially with degree ($1/4^d$) [S2, unofficial]. This ensures higher-frequency details have smaller initial amplitudes, stabilizing optimization [G].
- **Missing SH Coefficients**: If `sh_degree` is less than the maximum supported degree, remaining coefficients must be filled with noise to maintain tensor shape consistency [S2, unofficial].

## Subtopics

### [Consistent Anchor Guided Gaussian Splatting](14982-consistent-anchor-guided-gaussian-splatting.md)
Consistent Anchor Guided Gaussian Splatting (CAG-GS) is a large-scale scene rendering method that employs a divide-and-conquer strategy with semantic guidance to ensure global consistency across spatial blocks, thereby avoiding artifacts such as floaters and brightness inconsistencies. It distinguishes itself from sibling methods like Momentum-GS by directly incorporating consistent semantic information from a pre-trained segmentation model (SAM2) to guide feature learning, rather than relying solely on shared MLP decoders for regularization.

## Operating characteristics
- **Time Complexity**: $O(N \cdot D)$, where $N$ is the number of samples and $D$ is the number of SH coefficients [G]. The QR decomposition is $O(1)$ per Gaussian (fixed 3x3 matrix) [G].
- **Memory Complexity**: $O(N \cdot D)$ to store the generated Gaussians [G].
- **Performance**: The method is fast and deterministic in its randomness [S2, unofficial]. It does not depend on scene geometry, making it suitable for initial exploration or when no point cloud is available [G].
- **Strengths**: Simple to implement, no external data required, provides a diverse initial set of orientations and scales [G].
- **Limitations**: The Gaussians are not aligned with any scene structure, leading to slow convergence if the scene is complex [G]. The random positions require subsequent transformation or optimization to match the scene [G].
- **When to Use**: When initializing a new scene with no prior geometric information, or as a baseline for comparison [G].
- **When Not to Use**: When a point cloud or mesh is available, structured initialization (e.g., [Gaussian Splatting Initialization and Parameterization](2313-gaussian-splatting-initialization-and-parameterization.md)) is preferred for faster convergence and better initial quality [G].

## Background
The method is described in the context of the `gs-embedding` project, which explores embedding-based representations for 3D Gaussians [S2, unofficial]. The specific parameters and generation steps are derived from the implementation details provided in the source code and documentation of this project [S2, unofficial]. The use of QR decomposition for random rotations is a standard technique in computer graphics for generating uniform random rotations [G]. The exponential decay of SH variance is a common practice in neural rendering to stabilize training [G].

## Sources
[S1] Laityperfect7/GaussianScan-Recon, https://github.com/Laityperfect7/GaussianScan-Recon, fetched 2026-10-27 (unofficial)
[S2] cilix-ai/gs-embedding, https://github.com/cilix-ai/gs-embedding, fetched 2026-10-27 (unofficial)
