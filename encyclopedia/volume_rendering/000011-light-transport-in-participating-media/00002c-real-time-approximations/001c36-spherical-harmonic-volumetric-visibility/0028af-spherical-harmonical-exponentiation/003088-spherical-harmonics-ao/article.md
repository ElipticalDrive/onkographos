# Spherical Harmonics AO

Also called: Ren et al.'s Spherical Harmonics AO, SH visibility, Sphere Approximation SH AO, Spherical Harmonics Visibility Approximation, Visibility in Spherical Harmonics Space, sphere blocker visibility

## Overview
Spherical Harmonics AO approximates scene geometry with a collection of sphere blockers to model visibility in spherical harmonics (SH) space, enabling efficient computation of soft shadows and ambient occlusion [S1, unofficial][S2]. Unlike traditional rasterization which ignores global visibility or Monte Carlo ray tracing which is computationally expensive due to sampling, this method uses fast SH multiplications (via logarithmic summation) to combine visibility functions, making it suitable for real-time rendering and differentiable inverse problems [S2].

## How it works
**Inputs:**
- A scene geometry mesh.
- A set of sphere blockers $S_i$ attached to the mesh.
- Incident illumination represented as SH coefficients.

**Outputs:**
- Combined visibility SH vector $V(x)$ for each point $x$.
- Final rendered radiance combining illumination and visibility.

**Data Structures:**
- **Sphere Blocker:** Defined by center position $c_i$, radius $r_i$, and orientation (if non-spherical, though the base method assumes spheres).
- **SH Coefficients:** Vectors storing coefficients for bands $l=0$ to $L_{max}$. First-order coefficients ($l=0,1$) contain ambient occlusion factors and bent normals; higher-order coefficients ($l>1$) capture environment map shadowing [S1, unofficial].

**Steps:**

1.  **Geometry Approximation:**
    Approximate the scene geometry surface with a collection of sphere blockers $S_i$ rigged to the underlying mesh [S2]. This allows the mesh to deform and pose through the sphere representation [S2].

2.  **Illumination Modeling:**
    Model scene illumination using spherical harmonics representation [S2]. Let $L$ be the SH vector of the incident light [S2].

3.  **Single Sphere Visibility Modeling:**
    For a point $x$ and a sphere blocker $S_i$, define the binary visibility function $V_i(\omega, x)$ which is $0$ if $S_i$ blocks direction $\omega$ and $1$ otherwise [S2]. To enable logarithmic computation, approximate this with a soft boundary:
    $$
    V^i_0(\omega, x) = \begin{cases} 
    e^{-\epsilon} & \text{if } S_i \text{ blocks in direction } \omega \\
    1 & \text{otherwise}
    \end{cases}
    $$
    where $\epsilon = 3$ is set such that $e^{-3} \approx 0.05$ to avoid infinite logarithms for zero values [S2].

4.  **Logarithmic Projection:**
    Compute the logarithm of the approximated visibility function $\log(V^i_0(\omega, x))$. Orient this function to the z-axis and project it to SH space. This projection has an analytical form with respect to the distance from $x$ to the sphere center and the sphere's radius [S2].

5.  **Rotation:**
    Apply SH rotation to the projected logarithmic coefficients using the vector pointing from $x$ to the sphere center [S2].

6.  **Aggregation:**
    Sum the rotated logarithmic SH vectors of all sphere blockers. The global visibility $V(x)$ is computed by exponentiating the sum:
    $$
    V(x) = \exp\left( \sum_i \text{RotatedLog}(V^i(x)) \right)
    $$
    This avoids the computational expense of direct SH multiplication (Eq. 12 in [S2]) by leveraging the property that the logarithm of a product leads to a summation [S2].

7.  **Rendering:**
    Combine the soft shadow rendering (using SH multiplication of illumination $L$ and visibility $V(x)$) with a diffuse spherical harmonics based shading model to obtain the final rendering [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_sh_log_visibility(x, spheres, epsilon=3.0):
    """
    Compute the logarithm of the combined visibility function for point x.
    
    Args:
        x: Point position (3,)
        spheres: List of dicts with keys 'center' (3,), 'radius' (float)
        epsilon: Softening factor for visibility (default 3.0)
        
    Returns:
        log_V: SH vector of the logarithm of visibility
    """
    log_V_accumulator = None
    
    for sphere in spheres:
        center = sphere['center']
        radius = sphere['radius']
        
        # Vector from point to sphere center
        vec_to_center = center - x
        dist = np.linalg.norm(vec_to_center)
        
        # 1. Compute analytical SH coefficients for the oriented log-visibility
        # The function is oriented to the z-axis.
        # V_0 is e^(-epsilon) inside the cone blocked by the sphere, 1 outside.
        # The SH projection of log(V_0) depends on dist and radius.
        # Note: The exact analytical formula for SH coefficients of a cone/occlusion
        # is complex. In practice, this is often precomputed or approximated.
        # Here we assume a helper function exists that returns SH coeffs for 
        # a z-aligned blocker given dist and radius.
        log_V_z = get_analytical_sh_log_blocker(dist, radius, epsilon)
        
        # 2. Rotate to align with the actual direction to the sphere
        # Create rotation matrix/quaternion from z-axis to vec_to_center
        rot_matrix = rotation_matrix_from_z_to_vec(vec_to_center)
        
        # Apply SH rotation to the coefficients
        log_V_rotated = apply_sh_rotation(log_V_z, rot_matrix)
        
        # 3. Accumulate
        if log_V_accumulator is None:
            log_V_accumulator = log_V_rotated
        else:
            log_V_accumulator += log_V_rotated
            
    return log_V_accumulator

def get_analytical_sh_log_blocker(dist, radius, epsilon):
    """
    Computes SH coefficients for log(V_0) oriented along Z.
    V_0 is e^(-epsilon) if blocked, 1 otherwise.
    This is a placeholder for the specific analytical derivation found in [S2].
    The derivation involves integrating the log function against SH basis functions
    over the solid angle blocked by the sphere.
    """
    # Implementation detail: The sources state this has an analytical form.
    # A common approximation for the SH coefficients of a cone of half-angle theta
    # where the function is constant inside and constant outside can be derived.
    # theta = arcsin(radius / dist)
    # However, the exact coefficients are not explicitly listed in the provided text.
    # We return a zero vector as a placeholder, noting that [G] requires implementation.
    # In a real implementation, one would compute the integral:
    # c_l = int_{Omega} log(V_0(w)) * Y_l(w) dw
    # where V_0(w) = exp(-epsilon) if dot(w, z) > cos(theta), else 1.
    return np.zeros(9) # Assuming 3rd order (l=0,1,2) for example

def apply_sh_rotation(coeffs, rot_matrix):
    """
    Rotates SH coefficients using a rotation matrix.
    """
    # Standard SH rotation matrix application
    # Wigner D-matrices are typically used.
    # This is a placeholder for the standard SH rotation algorithm.
    return coeffs

def render(x, illumination_sh, spheres):
    """
    Renders the point x using SH visibility and illumination.
    """
    # 1. Get log visibility
    log_V = compute_sh_log_visibility(x, spheres)
    
    # 2. Exponentiate to get visibility SH vector
    # Note: Exponentiating SH coefficients is non-trivial.
    # The method in [S2] implies computing the exponential of the summed log SH.
    # This usually requires a lookup table or approximation as mentioned in [S2].
    V_sh = exponential_of_sh(log_V)
    
    # 3. Multiply illumination and visibility in SH space
    # SH multiplication (convolution)
    radiance_sh = sh_multiply(illumination_sh, V_sh)
    
    # 4. Evaluate radiance at the surface normal or integrate for diffuse shading
    # This step combines with the diffuse SH shading model mentioned in [S2]
    return radiance_sh
```

**Parameters:**
- **SH Bandwidth:** A limited subset of basis functions is used to obtain a low-frequency approximation [S2].
- **Epsilon ($\epsilon$):** Set to 3 to ensure $e^{-3} \approx 0.05$, preventing infinite logarithms [S2].

**Pitfalls:**
- **Small Details:** Occlusion from small details and creases is not accounted for because geometry is approximated through spheres [S1, unofficial].
- **Differentiability:** Traditional rasterization-based differentiable renderers do not model light source visibility, leading to artifacts like baked-in shadows in recovered texture/geometry [S2]. The method must ensure the logarithm and exponential steps are differentiable for inverse problems [S2].

## Operating characteristics
- **Time Complexity:** Significantly more efficient than Monte Carlo ray tracing due to the use of SH multiplications and logarithmic summation [S2]. The cost is dominated by the number of sphere blockers and the SH bandwidth.
- **Memory Complexity:** Proportional to the number of sphere blockers and the size of the SH coefficient vectors.
- **Strengths:** Efficient soft shadow computation; fully differentiable, enabling optimization of scene parameters (pose, deformation, illumination, texture) [S2].
- **Limitations:** Loss of high-frequency geometric detail due to sphere approximation [S1, unofficial]. Accuracy depends on the number and placement of sphere blockers.

## Background
The method was originally proposed by Ren et al. [26] (cited as [S1, unofficial] and [S2]) as a way to approximate scene geometry with spheres to model visibility using spherical harmonics. It was implemented in Nvidia's VXGI package as VXAO (Voxel Accelerated Ambient Occlusion) [S1, unofficial]. A later work by [S2] extended this formulation to be fully differentiable for inverse problems, providing a novel algorithmic approach to differentiate through the rendering process, including the exponential of logarithms of SH functions.

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S2] Efficient and Differentiable Shadow Computation for Inverse Problems, https://doi.org/10.1109/iccv48922.2021.01286, fetched 2026-11-01
