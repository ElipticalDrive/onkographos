# Angle-pose Cuboid Uniform Initialization (ACUI)

## Overview
Angle-pose Cuboid Uniform Initialization (ACUI) is a deterministic initialization strategy for 3D Gaussian Splatting in X-ray novel view synthesis that replaces Structure-from-Motion (SfM) by computing camera parameters directly from scanner specifications and uniformly sampling Gaussian centers within a bounding cuboid [S1]. It solves the problem of feature detection failure in low-contrast, overlapping X-ray projections, offering a significant reduction in training time compared to SfM-based initialization [S1].

## How it works
**Inputs:**
- Scanner parameters: azimuth angle $\phi$, source-detector distance $L_{SD}$, detector width $W$, detector height $H$ [S1].
- Bounding cuboid dimensions: $S_1, S_2, S_3$ (enclosing the object, centered at world origin) [S1].
- Grid resolution: $M_1, M_2, M_3$ (number of voxels along each axis) [S1].
- Sampling interval: $d$ [S1].

**Outputs:**
- Extrinsic matrix $\mathbf{M}_{ext}$ [S1].
- Intrinsic matrix $\mathbf{M}_{int}$ [S1].
- Set of 3D point positions $\mathcal{P}$ [S1].
- Number of points $N_p$ and initial means $\boldsymbol{\mu}_i$ derived from $\mathcal{P}$ [S1].

**Data Structures:**
- $\mathbf{M}_{ext}$: $3 \times 4$ matrix (rotation and translation) [S1].
- $\mathbf{M}_{int}$: $3 \times 4$ matrix (intrinsic parameters) [S1].
- $\mathcal{P}$: Set of 3D coordinates $(x, y, z)$ [S1].

**Steps:**
1. **Compute Extrinsic Matrix:** Calculate $\mathbf{M}_{ext}$ using the azimuth angle $\phi$. The elevation angle is fixed at zero [S1].
   $$
   \mathbf{M}_{ext} = \begin{bmatrix} -\sin \phi & \cos \phi & 0 & 0 \\ 0 & -\cos \phi & -\sin \phi & 0 \\ -\cos \phi & -\sin \phi & 0 & 0 \end{bmatrix}
   $$
   [S1]

2. **Compute Intrinsic Matrix:** Calculate $\mathbf{M}_{int}$ using the source-detector distance $L_{SD}$ and detector dimensions $W, H$ [S1].
   $$
   \mathbf{M}_{int} = \begin{bmatrix} L_{SD} & 0 & W/2 & 0 \\ 0 & L_{SD} & H/2 & 0 \\ 0 & 0 & 1 & 0 \end{bmatrix}
   $$
   [S1]

3. **Define Bounding Cuboid:** Establish a cuboid of size $S_1 \times S_2 \times S_3$ centered at the world origin $(0,0,0)$ that completely encloses the scanned object [S1].

4. **Uniform Sampling:** Divide the cuboid into a grid of $M_1 \times M_2 \times M_3$ voxels. Generate the set of sampled points $\mathcal{P}$ using integer indices $n_i$ and sampling interval $d$ [S1]:
   $$
   \mathcal{P} = \left\{ \left( \frac{n_1 S_1}{M_1}, \frac{n_2 S_2}{M_2}, \frac{n_3 S_3}{M_3} \right) \mid - \left[ \frac{2}{3} d \right] - 1 \leq n_i \leq \left[ \frac{2}{3} d \right] + 1, i = 1, 2, 3 \right\}
   $$
   where $n_i \in \mathbb{Z}$ and $[\cdot]$ denotes the floor function or integer part [S1].

5. **Initialize Gaussians:** Use the size of $\mathcal{P}$ to set the number of points $N_p$ and assign the elements of $\mathcal{P}$ as the initial mean positions $\boldsymbol{\mu}_i$ for the Gaussian point clouds [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_acui_init(phi, L_SD, W, H, S1, S2, S3, M1, M2, M3, d):
    """
    Computes ACUI initialization for X-ray Gaussian Splatting.
    
    Args:
        phi: Azimuth angle in radians.
        L_SD: Source-detector distance.
        W: Detector width.
        H: Detector height.
        S1, S2, S3: Cuboid dimensions along x, y, z axes.
        M1, M2, M3: Grid resolution (voxel counts) along x, y, z axes.
        d: Sampling interval.
        
    Returns:
        M_ext: 3x4 extrinsic matrix.
        M_int: 3x4 intrinsic matrix.
        P: List of 3D points (tuples).
        Np: Number of points.
    """
    # Step 1: Compute Extrinsic Matrix
    sin_phi = math.sin(phi)
    cos_phi = math.cos(phi)
    
    # M_ext is 3x4. The source defines a 3x3 rotation part and zeros for translation.
    # Note: The source formula shows a 3x3 matrix embedded in a 3x4 context implicitly 
    # or as a rotation matrix. We construct the 3x4 matrix as per source Eq. 12.
    M_ext = [
        [-sin_phi, cos_phi, 0, 0],
        [0, -cos_phi, -sin_phi, 0],
        [-cos_phi, -sin_phi, 0, 0]
    ]
    
    # Step 2: Compute Intrinsic Matrix
    # M_int is 3x4. Source Eq. 12.
    M_int = [
        [L_SD, 0, W / 2, 0],
        [0, L_SD, H / 2, 0],
        [0, 0, 1, 0]
    ]
    
    # Step 3 & 4: Uniform Sampling
    # Calculate bounds for n_i
    # The source uses [2/3 d]. Assuming floor or integer truncation.
    # We use math.floor for consistency with integer indexing.
    limit = math.floor((2 / 3) * d)
    n_min = -limit - 1
    n_max = limit + 1
    
    P = []
    
    # Iterate over all integer indices within the calculated bounds
    for n1 in range(n_min, n_max + 1):
        for n2 in range(n_min, n_max + 1):
            for n3 in range(n_min, n_max + 1):
                # Calculate coordinates
                x = (n1 * S1) / M1
                y = (n2 * S2) / M2
                z = (n3 * S3) / M3
                P.append((x, y, z))
                
    Np = len(P)
    
    return M_ext, M_int, P, Np
```

**Parameters:**
- $d$: The interval for uniform sampling within the grid [S1]. Typical value not given in the sources.
- $\phi$: Azimuth angle [S1].
- $L_{SD}$: Source-detector distance [S1].
- $W, H$: Detector dimensions [S1].
- $S_1, S_2, S_3$: Cuboid size [S1].
- $M_1, M_2, M_3$: Grid voxel counts [S1].

**Edge Cases and Pitfalls:**
- Standard SfM initialization fails for X-ray images because they are grayscale with low contrast and suffer from overlapping layers due to transmission imaging, which degrades feature detection and matching accuracy [S1]. ACUI avoids this by bypassing feature detection entirely [S1].
- The sampling bounds depend on $d$. If $d$ is not provided, the range of $n_i$ cannot be determined. The source does not specify how $d$ is chosen relative to $S_i$ or $M_i$ [S1].

## Subtopics

### Spherical Harmonics Initialization
Initializes SH coefficients for view-dependent appearance. ACUI focuses on geometry and camera pose initialization for isotropic radiative models, whereas SH initialization handles appearance. Choose ACUI for X-ray/CT scenarios where camera geometry is known from scanner specs. See [Spherical Harmonics Initialization](3163-spherical-harmonics-initialization.md).

## Operating characteristics
- **Time Complexity**: The initialization is deterministic and scales with the number of sampled points $N_p$, which is proportional to $(2 \lfloor \frac{2}{3} d \rfloor + 3)^3$. It avoids the iterative and computationally expensive SfM pipeline [S1].
- **Memory Complexity**: Requires storage for $N_p$ Gaussian centers and the fixed-size camera matrices [S1].
- **Measured Performance**: In X-ray novel view synthesis, ACUI-enabled X-Gaussian achieved 73× inference speed and 7× training speed compared to SOTA methods, with a 6.5 dB PSNR improvement [S1].
- **Strengths**: Eliminates the need for feature detection in low-contrast, overlapping X-ray projections; significantly reduces training time; leverages known scanner geometry [S1].
- **Limitations**: Assumes a circular cone beam X-ray scanning scenario with equal angular intervals; requires accurate scanner parameters ($\phi, L_{SD}, W, H$); does not account for view-dependent effects (isotropic model) [S1].
- **When to use**: X-ray novel view synthesis, sparse-view CT reconstruction, scenarios with known scanner geometry and low-contrast grayscale images [S1].
- **When not to use**: RGB-based novel view synthesis where SfM is robust; scenarios with unknown or inaccurate camera parameters; view-dependent radiative models [S1].

## Background
ACUI was proposed in the context of X-Gaussian, a 3D Gaussian Splatting-based framework for X-ray novel view synthesis [S1]. It was developed to address the limitations of Structure-from-Motion (SfM) in X-ray imaging, where low contrast and overlapping layers degrade feature matching [S1]. The method exploits the isotropic property of X-ray radiodensity and the known geometry of circular cone beam scanners [S1].

## Sources
[S1] Radiative Gaussian Splatting for Efficient X-Ray Novel View Synthesis, https://doi.org/10.1007/978-3-031-73232-4_16, fetched 2026-10-26
