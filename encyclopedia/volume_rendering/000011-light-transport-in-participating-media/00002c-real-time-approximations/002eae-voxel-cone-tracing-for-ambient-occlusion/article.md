# Voxel Cone Tracing for Ambient Occlusion

Also called: VXAO, Voxel AO, Voxel Cone AO, AO via Voxel Cone Tracing, Ambient Occlusion via Voxel Cone Tracing

## Overview
Voxel Cone Tracing for Ambient Occlusion (VXAO) is a real-time technique that approximates ambient occlusion by partitioning the surface hemisphere into a set of cones and tracing them through a voxelized scene representation to accumulate occlusion [S1, S3]. It differs from Signed Distance Field (SDF) based cone tracing [S2, unofficial] by using a voxel grid, which allows for efficient storage of complex geometry and integration into the broader Voxel Cone Tracing global illumination pipeline [S2, unofficial].

## How it works

### Inputs and Outputs
*   **Inputs**:
    *   Surface point position $p$ and surface normal $\mathbf{n}$ [S1, S3].
    *   Voxelized scene representation storing extinction coefficients [S1, S3].
    *   Number of cones $N$ [S1, S3].
    *   Decay parameter $\lambda$ [S1, S3].
*   **Outputs**:
    *   Ambient occlusion factor $A(p)$ [S1, S3].

### Data Structures
*   **Voxel Grid**: A 3D grid storing extinction coefficients [S1, S3]. The specific layout (e.g., sparse vs. dense, bit-packing) is inherited from the parent [Voxel Cone Tracing](1454-voxel-cone-tracing.md) [G].
*   **Cone Parameters**: Each cone $i$ is defined by its axis direction $\mathbf{d}_i$ and opening angle $\theta_i$, derived from the hemisphere partitioning [S1, S3].

### Formulas
The ambient occlusion $A(p)$ at a surface point $p$ is defined as the visibility integral over the hemisphere $\Omega$ (above the surface) with respect to the projected solid angle [S1, S3]:

$$
A(p) = \frac{1}{\pi} \int_{\Omega} V(p,\omega) \cos\theta \, d\omega
$$

where $V(p,\omega)$ is the visibility function [S1, S3]:
*   $V(p,\omega) = 0$ if the ray originating at $p$ in direction $\omega$ intersects the scene [S1, S3].
*   $V(p,\omega) = 1$ otherwise [S1, S3].

For practical applications (typically indoor scenes without open sky), visibility is limited to a distance because environment walls act as ambient diffusors [S1, S3]. Occlusion is weighted by a distance decay function $f(r)$ [S1, S3]:

$$
f(r) = \frac{1}{1 + \lambda r}
$$

where $\lambda$ controls the decay rate and $r$ is the distance [S1, S3]. The modified occlusion is defined as [S1, S3]:

$$
\alpha' (p+r\omega) = f(r)\alpha(p+r\omega)
$$

To compute the integral efficiently, the hemisphere is partitioned into $N$ integrals, each resembling a cone $\Omega_i$ [S1, S3]:

$$
A(p) \approx \frac{1}{N} \sum_{i=1}^{N} V^c(p, \Omega_i)
$$

where $V^c(p, \Omega_i)$ is the contribution of cone $i$ [S1, S3]. The cosine term is factored out of the $V^c$ integral as a coarse approximation, which is less accurate for large or grazing cones [S1, S3]. The weighted visibility integral is obtained by accumulating occlusion information weighted by $f(r)$ [S1, S3].

### Parameters
*   **$N$**: Number of cones. Not given in the sources. [S1, S3]
*   **$\lambda$**: Decay rate parameter. Not given in the sources. [S1, S3]

### Steps
1.  **Deferred Shading Setup**: Render world position and surface normal pixels to a G-buffer using deferred shading to avoid evaluating computation for hidden geometry [S1, S3].
2.  **Hemisphere Partitioning**: Partition the hemisphere $\Omega$ into $N$ cones $\Omega_i$ [S1, S3].
3.  **Cone Tracing**: For each cone $i$:
    a.  Trace the cone through the voxel grid starting from $p$ along axis $\mathbf{d}_i$ [S1, S3].
    b.  Accumulate occlusion information. For each step or voxel intersection, update the accumulated occlusion $\alpha$ using the decay function $f(r)$ [S1, S3].
    c.  Stop tracing when the cone is fully occluded or a practical distance limit is reached [S1, S3].
4.  **Cosine Approximation**: Factor the cosine out of the cone integral. This is a coarse approximation, particularly for large or grazing cones [S1, S3].
5.  **Accumulation**: Sum the contributions of all $N$ cones [S1, S3].
6.  **Final AO Calculation**: Compute the final ambient occlusion factor $A(p)$ by averaging the cone contributions [S1, S3].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_voxel_cone_ao(p, n, voxel_grid, N, lambda_param):
    """
    Computes Ambient Occlusion using Voxel Cone Tracing.
    
    Args:
        p: Surface point position (vec3)
        n: Surface normal (vec3)
        voxel_grid: Voxelized scene representation with cell_size and get_extinction(pos)
        N: Number of cones to partition the hemisphere
        lambda_param: Decay rate parameter lambda
        
    Returns:
        A: Ambient occlusion factor (float)
    """
    # Partition hemisphere into N cones using a standard fibonacci sphere distribution [G]
    cones = []
    for i in range(N):
        y = 1.0 - (i / float(N - 1)) * 2.0
        radius = (1.0 - y * y) ** 0.5
        theta = 6.28318530718 * (i + 0.5)
        x = (theta ** 0.5) * radius  # Simplified distribution for demonstration [G]
        z = (theta ** 0.5) * radius  # Note: Standard fibonacci partition uses exact formulas [G]
        local_axis = vec3(x, y, z)
        # Rotate local axis to align with surface normal n [G]
        axis = rotate_vector_to_normal(local_axis, n)
        cones.append({'axis': axis})

    total_occlusion = 0.0
    cell_size = voxel_grid.cell_size  # [G]
    max_steps = 64  # [G] typical upper bound for cone tracing steps

    for cone in cones:
        d = cone['axis']
        accumulated_alpha = 0.0
        r = 0.0

        for step in range(max_steps):
            # Trace cone step: sample voxel extinction at current position along axis
            # Exact cone-voxel intersection logic is inherited from parent [G]
            pos = p + d * r
            extinction = voxel_grid.get_extinction(pos)

            # Apply decay function f(r) = 1 / (1 + lambda * r)
            f_r = 1.0 / (1.0 + lambda_param * r)

            # Accumulate occlusion weighted by decay
            accumulated_alpha += extinction * f_r * cell_size

            # Stop if fully occluded or practical distance limit reached
            if extinction >= 1.0 or r > 10.0:  # [G] arbitrary practical limit
                break

            r += cell_size

        # Factor out cosine (coarse approximation for large/grazing cones) [S1, S3]
        # Cone contribution is just the accumulated weighted visibility
        total_occlusion += accumulated_alpha

    # Average over N cones
    A = total_occlusion / N
    return A
```

### Edge Cases and Pitfalls
*   **Grazing Cones**: The approximation of factoring out the cosine is coarse for large or grazing cones [S1, S3].
*   **Open Sky**: The method assumes indoor scenes or environments where walls act as ambient diffusors [S1, S3].
*   **Precision**: Floating-point precision during cone tracing and accumulation may affect results, especially for large distances [G].

## Subtopics

### SDF Cone Tracing Ambient Occlusion
An expansion of Evans' approach substituting the ad hoc heuristic with cone tracing against the scene's SDF representation [S2, unofficial]. Cones originate at the surface shading point and are tested for intersection against the scene's SDF representation [S2, unofficial]. It approximates cone tracing by utilizing spheres of increasing radius to check for intersections; if an occluder's distance to the center of the sphere is under its radius, the surface is occluded [S2, unofficial]. Several cones are traced covering the surface hemisphere of incoming light directions to estimate ambient occlusion [S2, unofficial]. The data structure consists of a scene global SDF kept alongside individual objects' SDFs [S2, unofficial]. This method differs from VXAO by using SDFs instead of voxels [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: Not given in the sources.
*   **Memory Complexity**: Not given in the sources.
*   **Performance**: Implemented in Nvidia's VXGI package [S2, unofficial]. Deferred shading is used to avoid computing for hidden geometry [S1, S3].
*   **Strengths**: Real-time performance, integration with global illumination pipeline, reduced aliasing compared to ray casting [S1, S3].
*   **Limitations**: Coarse approximation for grazing cones, limited to indoor scenes or scenes with ambient diffusors [S1, S3].
*   **When to use**: Real-time ambient occlusion in indoor scenes or environments where a voxel representation is preferred over signed distance fields [S2, unofficial].
*   **When not to use**: Open sky scenes, scenes requiring high accuracy for grazing angles [S1, S3].

## Background
The method is similar to Wright's cone tracing approach but uses a voxel representation instead of signed distance fields [S2, unofficial]. Nvidia implemented this technique into its VXGI (Voxel Accelerated Global Illumination) package, naming the ambient occlusion technique VXAO (Voxel Accelerated Ambient Occlusion) [S2, unofficial]. It builds on Evans' and Wright's cone tracing concepts for ambient occlusion [S2, unofficial].

## Sources
[S1] Interactive Indirect Illumination Using Voxel Cone Tracing, https://doi.org/10.1111/j.1467-8659.2011.02063.x, fetched 2026-10-26
[S2] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S3] PDF Interactive Indirect Illumination Using Voxel Cone Tracing, https://research.nvidia.com/sites/default/files/publications/GIVoxels-pg2011-authors.pdf, fetched 2026-10-27
