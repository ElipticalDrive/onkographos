# Uniform Sphere Sampling

Also called: UniformSphereSampler, Uniform Solid Angle Sampling, sampleUniformSphere, SphereSampler, getSampleUniformSolidAngle

## Overview
Uniform Sphere Sampling generates a random direction vector uniformly distributed over the surface of a unit sphere [S2, S4, unofficial]. It solves the problem of casting rays or sampling directions in 3D space where no preferred orientation exists, ensuring that every solid angle has an equal probability of being selected [S2, S4, unofficial]. It sets itself apart from [Random Hemisphere Sampling](2034-random-hemisphere-sampling.md) by covering the full $4\pi$ steradians rather than restricting samples to a half-space, and from [Rejection Sampling for Uniform Sphere Sampling](3286-rejection-sampling-for-uniform-sphere-sampling.md) by using direct spherical coordinate conversion which avoids the variance in iteration count inherent in rejection-based approaches [S2, S4, S5, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** Two independent uniform random numbers $u, v \in [0, 1)$, typically derived from a seed or pixel coordinate [S1, S2, S4, unofficial].
*   **Output:** A 3D unit vector $\mathbf{d} = (x, y, z)$ representing a direction on the unit sphere [S1, S2, S4, unofficial].

**Data Structures**
No complex data structures are required. The method operates on scalar random values and produces a 3D vector. In batched implementations (e.g., PyTorch), inputs are tensors of shape $(N, 2)$ and outputs are tensors of shape $(N, 3)$ [S5, unofficial].

**Steps**
1.  **Generate Random Inputs:** Obtain two independent uniform random numbers $u$ and $v$ in the range $[0, 1)$ [S1, S2, S4, unofficial].
    *   *Note:* In shader-based implementations, these are often derived from fragment coordinates or a hash function [S1, S4, unofficial].
2.  **Compute Azimuthal Angle ($\theta$):** Calculate the angle in the $xy$-plane:
    $$ \theta = 2\pi u $$
    [S1, S2, S4, unofficial]
3.  **Compute Polar Angle ($\phi$):** Calculate the angle from the positive $z$-axis. To ensure uniform distribution, the cosine of the polar angle must be uniform in $[-1, 1]$. Let $v$ be the second random number:
    $$ \cos\phi = 2v - 1 $$
    $$ \phi = \arccos(2v - 1) $$
    [S1, S2, S4, unofficial]
    *   *Alternative Formulation:* Some sources define $\phi$ directly via $\cos\theta = 2v - 1$ and use $\theta$ for the polar angle [S2, S4, unofficial]. The resulting Cartesian conversion remains consistent if the sine/cosine assignments match the axis definitions.
4.  **Convert to Cartesian Coordinates:** Compute the components of the direction vector $\mathbf{d}$:
    $$ x = \sin\phi \cos\theta $$
    $$ y = \sin\phi \sin\theta $$
    $$ z = \cos\phi $$
    [S1, S2, S4, unofficial]
    *   *Note:* $\sin\phi$ can be computed as $\sqrt{1 - \cos^2\phi} = \sqrt{1 - (2v - 1)^2}$ to avoid calling `asin` or `acos` for the sine term, though `sin(acos(...))` is mathematically equivalent [S2, unofficial].

**Pseudocode**
The following Python-like pseudocode implements the standard spherical coordinate conversion method [S1, S2, S4, S5, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sample_uniform_sphere(u: float, v: float) -> tuple:
    """
    Generates a uniformly distributed random direction on a unit sphere.
    
    Args:
        u: Uniform random number in [0, 1) for azimuthal angle.
        v: Uniform random number in [0, 1) for polar angle cosine.
        
    Returns:
        (x, y, z): Cartesian coordinates of the unit vector.
    """
    # Step 1: Compute azimuthal angle theta
    theta = 2.0 * math.pi * u
    
    # Step 2: Compute cosine of polar angle phi
    cos_phi = 2.0 * v - 1.0
    
    # Step 3: Compute sine of polar angle phi
    # Using sqrt(1 - cos^2) is numerically stable for v in [0,1]
    sin_phi = math.sqrt(1.0 - cos_phi * cos_phi)
    
    # Step 4: Convert to Cartesian coordinates
    x = sin_phi * math.cos(theta)
    y = sin_phi * math.sin(theta)
    z = cos_phi
    
    return (x, y, z)

def sample_uniform_sphere_batch(u_array: list, v_array: list) -> list:
    """
    Batched version for generating multiple samples.
    
    Args:
        u_array: List of u values.
        v_array: List of v values.
        
    Returns:
        List of (x, y, z) tuples.
    """
    samples = []
    for u, v in zip(u_array, v_array):
        samples.append(sample_uniform_sphere(u, v))
    return samples
```

**Edge Cases and Pitfalls**
*   **Poles:** At the poles ($v=0$ or $v=1$), $\sin\phi = 0$, so $x$ and $y$ become 0. The azimuthal angle $\theta$ becomes irrelevant, which is correct for uniform sampling. No special handling is required as the limit is well-defined [S2, unofficial].
*   **Precision:** When $v$ is very close to 0 or 1, `1.0 - cos_phi * cos_phi` may suffer from catastrophic cancellation if computed as `1.0 - (2v-1)**2`. Using `math.sqrt(1.0 - cos_phi * cos_phi)` is standard, but for extreme precision, one might use `math.sqrt(2.0 * (1.0 - cos_phi))` near the poles or rely on hardware `sin`/`cos` accuracy [S2, unofficial].
*   **Random Number Quality:** The uniformity of the output depends critically on the independence and uniformity of $u$ and $v$. Correlated random numbers will produce artifacts on the sphere surface [S1, S4, unofficial].
*   **Volume vs. Surface:** This method samples the *surface* of the sphere. If uniform sampling *within* the volume of a sphere is required, a radius $r$ must be sampled as $r = u^{1/3}$ (cube root of a uniform random number) and multiplied into the Cartesian coordinates [S5, unofficial].

## Subtopics

### [Random Hemisphere Sampling](2034-random-hemisphere-sampling.md)
Generates directions uniformly over a hemisphere. Chosen when rays must point into a specific half-space (e.g., diffuse reflection).

### [Importance Phong Sampling](2224-importance-phong-sampling.md)
Samples according to a Phong lobe. Chosen for glossy/specular surfaces to reduce variance.

### [Rejection Sampling for Uniform Sphere Sampling](3286-rejection-sampling-for-uniform-sphere-sampling.md)
Samples by generating points in a cube and normalizing. Chosen when trigonometric functions are expensive or unavailable, trading potential iteration variance for arithmetic simplicity.

### [Spherical Background Sampling](3991-spherical-background-sampling.md)
Maps existing directions to environment maps. Chosen for background lighting, not for generating new random directions.

### [Anisotropic Scattering via Importance Sampling](4770-anisotropic-scattering-via-importance-sampling.md)
Samples according to a phase function. Chosen for anisotropic media to reduce variance.

### Random Camera Batch Generation
Generates a batch of random camera positions on a sphere for rendering [S6, unofficial]. It samples spherical coordinates (azimuth: $0$ to $2\pi$, elevation: $-\pi/2$ to $\pi/2$) for a given batch size, converts them to Cartesian coordinates at a specified `cam_radius`, and adds a small random offset to the positions [S6, unofficial]. It sets camera look-at directions to the origin and up vectors to the Y-axis [S6, unofficial]. Parameters include `fovy` (default 45 degrees), `iter_res` (default 2048x2048), `cam_near_far` (default 0.1, 1000.0), and `cam_radius` (default 3.0) [S6, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample. Requires two trigonometric functions (`sin`, `cos`) and one square root (or `acos`) [S1, S2, S4, unofficial].
*   **Memory Complexity:** $O(1)$. Only stores the random inputs and the resulting vector [S5, unofficial].
*   **Performance:** Generally faster than rejection sampling for high-quality random numbers because it guarantees termination in constant time [S2, S5, unofficial]. However, it is slower than rejection sampling if trigonometric functions are extremely expensive on the target hardware and low-quality randomness is acceptable [S5, unofficial].
*   **Strengths:** Deterministic runtime; mathematically exact uniform distribution over the sphere surface; simple to implement [S1, S2, S4, unofficial].
*   **Limitations:** Requires trigonometric functions; sensitive to the quality of the underlying random number generator [S1, S4, unofficial].
*   **When to use:** When uniform coverage of all directions is required and performance allows for trigonometric operations [S2, S4, unofficial].
*   **When not to use:** When sampling within the volume of a sphere (use cube root radius scaling) or when hardware lacks efficient trig functions (consider rejection sampling) [S5, unofficial].

## Background
Uniform sphere sampling is a fundamental technique in computer graphics, originating from the need to simulate light scattering, global illumination, and camera ray generation in 3D environments [S1, S2, S4, unofficial]. The method of using spherical coordinates with $\cos\phi$ uniformly distributed in $[-1, 1]$ is a standard result in Monte Carlo integration to avoid the clustering of samples at the poles that occurs if $\phi$ is sampled uniformly [S2, S4, unofficial]. It is widely used in path tracers, voxel ray marchers, and neural rendering pipelines such as VF-NeRF [S1, S4, S5, unofficial].

## Sources
[S1] jamestkiernan/webgpu-voxel-raymarching, https://github.com/jamestkiernan/webgpu-voxel-raymarching, fetched 2026-10-24 (unofficial)
[S2] newDINO/voxel_ray_demo, https://github.com/newDINO/voxel_ray_demo, fetched 2026-10-24 (unofficial)
[S3] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
[S4] viljanvr/VisualNifty, https://github.com/viljanvr/VisualNifty, fetched 2026-10-26 (unofficial)
[S5] albertgassol1/vf-nerf, https://github.com/albertgassol1/vf-nerf, fetched 2026-10-26 (unofficial)
[S6] AlexandreBinninger/TetWeave, https://github.com/AlexandreBinninger/TetWeave, fetched 2026-10-27 (unofficial)
