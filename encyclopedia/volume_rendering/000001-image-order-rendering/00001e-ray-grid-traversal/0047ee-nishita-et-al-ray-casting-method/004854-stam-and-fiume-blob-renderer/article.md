# Stam and Fiume Blob Renderer
Also called: Blob Renderer, Blob-Warping, Volumetric Blob Rendering
## Overview
A volumetric rendering approach for gaseous phenomena that models non-uniformly modified density blobs convected by wind fields and diffusion processes [S2, unofficial]. It differs from standard particle systems by treating blobs as spherical particles with normally distributed gas volumes, rendered using diffusion equations [S2, unofficial]. It applies a front-to-back strategy to handle turbulent fields [S1].
## How it works
Inputs: A set of spherical blobs, each defined by position, velocity, and density parameters; a wind field; and diffusion parameters. Outputs: A rendered image of the gaseous phenomenon.

Data Structures:
- Blob: A structure containing position vector $\mathbf{p}$, velocity vector $\mathbf{v}$, radius $r$, and density distribution parameters.
- Wind Field: A function or grid returning velocity $\mathbf{u}(\mathbf{x}, t)$ at position $\mathbf{x}$ and time $t$.

Steps:
1. Initialize blob positions and velocities based on physical properties (convection, pressure, drag) [S2, unofficial].
2. For each frame, update blob positions using Newton’s Laws of Motion, accounting for convection, drag, and thermal buoyancy [S2, unofficial].
3. Model the gas density within each blob as normally distributed over its spherical volume [S2, unofficial].
4. Apply diffusion equations to modify the density distribution over time [S2, unofficial].
5. Render the scene using a front-to-back ray casting strategy [S1].
6. For each pixel, cast a ray and integrate the density contributions of intersecting blobs, considering the normally distributed gas volume [S2, unofficial].
7. Compute lighting and scattering effects based on the integrated density [S1][S2, unofficial].

Formulas:
- Gas density distribution: $\rho(\mathbf{x}) = \rho_0 \exp\left(-\frac{|\mathbf{x} - \mathbf{p}|^2}{2\sigma^2}\right)$, where $\rho_0$ is peak density, $\mathbf{p}$ is blob center, and $\sigma$ is spread parameter [S2, unofficial].
- Position update: $\mathbf{p}_{t+1} = \mathbf{p}_t + \mathbf{v}_t \Delta t$ [S2, unofficial].
- Velocity update: $\mathbf{v}_{t+1} = \mathbf{v}_t + (\mathbf{u}(\mathbf{p}_t, t) - \mathbf{v}_t) \Delta t + \mathbf{a}_{buoyancy} \Delta t$ [S2, unofficial].

Parameters:
- $\sigma$: Spread parameter for density distribution. Not given in the sources.
- $\Delta t$: Time step. Not given in the sources.
- $\rho_0$: Peak density. Not given in the sources.

Edge Cases and Pitfalls:
- Precision issues in density calculation for distant blobs. [G] Use early ray termination if density falls below threshold.
- Boundary handling for wind fields. [G] Clamp or interpolate wind velocities at domain boundaries.
- Degenerate input: Blobs with zero radius or density. [G] Skip such blobs during rendering.

Pseudocode:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_blob_renderer(blobs, wind_field, diffusion_params, camera, image_width, image_height):
    image = zeros((image_height, image_width, 3))
    for y in range(image_height):
        for x in range(image_width):
            ray = camera.cast_ray(x, y)
            color = vec3(0.0)
            transmittance = 1.0
            for blob in blobs:
                if ray_intersects_blob(ray, blob):
                    density = compute_density(ray, blob, diffusion_params)
                    absorption = density * ray.step_size
                    color += (1.0 - transmittance) * lighting(ray, blob)
                    transmittance *= exp(-absorption)
                    if transmittance < 0.01:
                        break
            image[y, x] = color
    return image

def compute_density(ray, blob, diffusion_params):
    # Compute intersection point and distance
    # Apply diffusion equation to modify density
    # Return density value
    pass

def lighting(ray, blob):
    # Compute lighting based on blob properties and wind field
    pass
```
## Operating characteristics
Time complexity: $O(N \cdot M)$, where $N$ is the number of pixels and $M$ is the number of blobs. Memory complexity: $O(M)$ for storing blob data. Performance depends on the number of blobs and the complexity of the wind field. Strengths: Realistic modeling of turbulent gases. Limitations: Computationally expensive for large numbers of blobs. When to use: For realistic gaseous effects like smoke or fire. When not to use: For real-time applications with limited computational resources.
## Background
Origin: Stam and Fiume [S1]. History: Applied to turbulent fields in gaseous phenomena [S1]. Usage: Rendering of gaseous effects like smoke and fire [S2, unofficial].
## Sources
[S1] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
[S2] Realistic Real-Time Rendering of Fire in a Production System, https://clas.iusb.edu/math-compsci/_prior-proposals/YVanzine_Proposal.pdf, fetched 2026-10-28 (unofficial)
