# Cosine-Weighted Hemisphere Sampling

Also called: Cosine-Weighted Hemisphere Sampling for GI, Cosine-Weighted Solid Angle Sampling, Indirect Radiance with Cosine-Weighted Hemisphere Sampling, getSampleCosineWeighted

## Overview
Cosine-Weighted Hemisphere Sampling generates a random direction uniformly distributed within a hemisphere oriented around a given normal vector, weighted by the cosine of the polar angle [S2, S3, unofficial]. It is used to compute the indirect lighting contribution at a surface point by sampling the hemisphere above the surface normal and recursively tracing rays [S2, unofficial]. It sets itself apart from its parent [Random Hemisphere Sampling](2034-random-hemisphere-sampling.md) by biasing samples toward the normal, which reduces variance for diffuse surfaces where the BRDF is proportional to the cosine of the incident angle.

## How it works
**Inputs:**
- Surface normal $\mathbf{n}$ (unit vector).
- Two uniform random numbers $r_1, r_2 \in [0, 1)$ [S1, S2, S3, unofficial].
- Optional: Tangent $\mathbf{t}$ and bitangent $\mathbf{b}$ vectors to form a local coordinate system, or a method to construct one from $\mathbf{n}$ [S1, S2, S3, unofficial].

**Outputs:**
- A unit direction vector $\mathbf{w}_i$ in the hemisphere defined by $\mathbf{n}$ [S1, S2, S3, unofficial].
- The probability density function (PDF) value for the sampled direction [S2, unofficial].

**Data Structures:**
- No specific complex data structures are required. The method operates on 3D vectors and scalar random numbers.

**Steps:**

1.  **Generate Random Numbers:**
    Generate two independent uniform random numbers $r_1$ and $r_2$ in the range $[0, 1)$ [S1, S2, S3, unofficial].
    *Note:* In practical implementations, these are often derived from a hash function of pixel coordinates, seed, and sample index to ensure decorrelation [S1, unofficial].

2.  **Compute Local Spherical Coordinates:**
    Calculate the polar angle $\theta$ and azimuthal angle $\phi$ in the local tangent space using the following formulas [S2, S3, unofficial]:
    $$
    \cos\theta = \sqrt{1 - r_1}
    $$
    $$
    \sin\theta = \sqrt{r_1}
    $$
    $$
    \phi = 2\pi r_2
    $$
    Alternatively, compute the components directly in a local Cartesian frame where the z-axis aligns with the normal [S1, unofficial]:
    $$
    r = \sqrt{r_1}
    $$
    $$
    x_{local} = r \cos(\phi)
    $$
    $$
    y_{local} = r \sin(\phi)
    $$
    $$
    z_{local} = \sqrt{\max(0.0, 1.0 - r_1)}
    $$

3.  **Construct Local Basis:**
    Construct an orthonormal basis $(\mathbf{t}, \mathbf{b}, \mathbf{n})$ where $\mathbf{n}$ is the surface normal.
    *   If $\mathbf{t}$ and $\mathbf{b}$ are provided, use them directly [S1, S2, unofficial].
    *   If not, construct them from $\mathbf{n}$ to avoid degeneracy when $\mathbf{n}$ is parallel to the world up vector (e.g., $\mathbf{z} = (0,0,1)$) [S2, S3, unofficial]. A common robust method is:
        1.  Choose an arbitrary vector $\mathbf{a}$ not parallel to $\mathbf{n}$ (e.g., if $|n_x| < |n_z|$, $\mathbf{a} = (1,0,0)$, else $\mathbf{a} = (0,1,0)$).
        2.  $\mathbf{t} = \text{normalize}(\mathbf{a} \times \mathbf{n})$.
        3.  $\mathbf{b} = \mathbf{n} \times \mathbf{t}$.

4.  **Transform to World Space:**
    Combine the local components with the basis vectors to obtain the world-space direction $\mathbf{w}_i$ [S1, S2, S3, unofficial]:
    $$
    \mathbf{w}_i = \mathbf{t} \cdot x_{local} + \mathbf{b} \cdot y_{local} + \mathbf{n} \cdot z_{local}
    $$
    Or equivalently using the spherical components:
    $$
    \mathbf{w}_i = \mathbf{t} \cdot (\sin\theta \cos\phi) + \mathbf{b} \cdot (\sin\theta \sin\phi) + \mathbf{n} \cdot (\cos\theta)
    $$
    Normalize the result to ensure it is a unit vector [S1, S3, unofficial].

5.  **Compute PDF:**
    The probability density function for this sampling strategy over the hemisphere is [S2, unofficial]:
    $$
    p(\mathbf{w}_i) = \frac{\cos\theta}{\pi}
    $$
    where $\cos\theta = \mathbf{n} \cdot \mathbf{w}_i$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def cosine_weighted_hemisphere_sampling(normal, r1, r2):
    """
    Generates a cosine-weighted direction in the hemisphere defined by normal.
    
    Args:
        normal (tuple): (nx, ny, nz) unit normal vector.
        r1 (float): Uniform random number in [0, 1).
        r2 (float): Uniform random number in [0, 1).
        
    Returns:
        tuple: (direction, pdf)
            direction (tuple): (dx, dy, dz) unit direction vector.
            pdf (float): Probability density value.
    """
    # Step 1: Compute spherical coordinates
    # cos_theta = sqrt(1 - r1)
    # sin_theta = sqrt(r1)
    # phi = 2 * pi * r2
    
    cos_theta = math.sqrt(max(0.0, 1.0 - r1))
    sin_theta = math.sqrt(max(0.0, r1))
    phi = 2.0 * math.pi * r2
    
    # Step 2: Construct local basis
    # Robustly construct tangent and bitangent from normal
    # Avoid degeneracy if normal is parallel to world up (0,0,1)
    if abs(normal[0]) < abs(normal[2]):
        # Normal is closer to Z axis, use X axis as reference
        tangent = (1.0, 0.0, 0.0)
    else:
        # Normal is closer to X or Y axis, use Z axis as reference
        tangent = (0.0, 0.0, 1.0)
        
    # Tangent = normalize(cross(tangent_ref, normal))
    tx = tangent[1] * normal[2] - tangent[2] * normal[1]
    ty = tangent[2] * normal[0] - tangent[0] * normal[2]
    tz = tangent[0] * normal[1] - tangent[1] * normal[0]
    len_t = math.sqrt(tx*tx + ty*ty + tz*tz)
    if len_t == 0:
        # Fallback if normal is exactly parallel to reference, use arbitrary orthogonal
        tangent = (1.0, 0.0, 0.0)
        tx, ty, tz = tangent
        len_t = 1.0
    tangent = (tx/len_t, ty/len_t, tz/len_t)
    
    # Bitangent = cross(normal, tangent)
    bx = normal[1] * tangent[2] - normal[2] * tangent[1]
    by = normal[2] * tangent[0] - normal[0] * tangent[2]
    bz = normal[0] * tangent[1] - normal[1] * tangent[0]
    bitangent = (bx, by, bz)
    
    # Step 3: Compute local direction components
    cos_phi = math.cos(phi)
    sin_phi = math.sin(phi)
    
    local_x = sin_theta * cos_phi
    local_y = sin_theta * sin_phi
    local_z = cos_theta
    
    # Step 4: Transform to world space
    # dir = tangent * local_x + bitangent * local_y + normal * local_z
    dx = tangent[0] * local_x + bitangent[0] * local_y + normal[0] * local_z
    dy = tangent[1] * local_x + bitangent[1] * local_y + normal[1] * local_z
    dz = tangent[2] * local_x + bitangent[2] * local_y + normal[2] * local_z
    
    # Normalize (should be unit length theoretically, but good for safety)
    len_d = math.sqrt(dx*dx + dy*dy + dz*dz)
    if len_d > 0:
        dx /= len_d
        dy /= len_d
        dz /= len_d
        
    direction = (dx, dy, dz)
    
    # Step 5: Compute PDF
    # pdf = cos_theta / pi
    # cos_theta is dot(normal, direction), which should be local_z
    pdf = cos_theta / math.pi
    
    return direction, pdf
```

**Parameters:**
- $r_1, r_2$: Uniform random numbers in $[0, 1)$.
- $\pi$: Mathematical constant, approximately $3.14159265$.

**Edge Cases and Pitfalls:**
- **Degenerate Basis:** If the normal is parallel to the world up vector, the cross product used to generate the tangent may result in a zero vector. The pseudocode above handles this by checking the components of the normal and selecting an appropriate reference vector [S2, S3, unofficial].
- **Precision:** Ensure $1 - r_1$ is non-negative before taking the square root to avoid NaNs due to floating-point errors [S1, unofficial].
- **PDF Calculation:** The PDF must be calculated using the cosine of the angle between the normal and the sampled direction, which corresponds to the z-component in the local frame [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample. The computation involves basic arithmetic operations and square roots.
- **Memory Complexity:** $O(1)$. No additional memory is required beyond the input vectors and random numbers.
- **Strengths:**
    - Reduces variance for diffuse surfaces because the sampling distribution matches the cosine lobe of the Lambertian BRDF [S2, unofficial].
    - Simple to implement and computationally inexpensive.
- **Limitations:**
    - Not optimal for specular or glossy materials where the energy is concentrated in a narrow cone.
    - Biased towards the normal, which may not be ideal for all surface types.
- **When to Use:** Ideal for rendering diffuse (Lambertian) surfaces in global illumination algorithms [S2, unofficial].
- **When Not to Use:** Not suitable for specular highlights or anisotropic materials without modification.

## Background
Cosine-weighted hemisphere sampling is a standard technique in computer graphics for importance sampling diffuse reflections. It originates from the need to efficiently estimate the integral of radiance over a hemisphere for Lambertian surfaces. The method is widely used in path tracing and other Monte Carlo rendering algorithms.

## Sources
[S1] mstampfli/voxelG, https://github.com/mstampfli/voxelG, fetched 2026-10-24 (unofficial)
[S2] ainagutierrez01/Volumetric-Path-Tracing-, https://github.com/ainagutierrez01/Volumetric-Path-Tracing-, fetched 2026-10-26 (unofficial)
[S3] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
