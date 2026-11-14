# Sliced Ray Casting with On-Demand Volume Reconstruction

Also called: On-Demand Density Sampling, Sliced Ray Casting

## Overview
Sliced Ray Casting with On-Demand Volume Reconstruction is a direct volume rendering technique that reconstructs particle density on-demand in screen space at viewport resolution and sampling depths, combining splatting, texture slicing, and ray casting to avoid aliasing artifacts and large memory footprints without pre-computation [S1, unofficial]. It sets itself apart from traditional object-space volume ray casting by performing density reconstruction in image space, thereby moving perspective correction from the rendering stage to the data reconstruction stage [S1, unofficial].

## How it works
The method executes an iterative loop for `#s` times containing three inner steps: sampling position update, density reconstruction through splatting, and volume ray-casting [S1, unofficial].

### Inputs and Outputs
*   **Inputs**: Particle positions $\mathbf{p}_j$, particle masses $m_j$, kernel support radius $h$, volume sampling distance $\Delta \lambda$, eye position $\mathbf{e}$, ray direction $\mathbf{v}$, maximum ray length $\lambda_{max}$, and number of iterations $\#s$ [S1, unofficial].
*   **Outputs**: Final composited color and opacity for each pixel [S1, unofficial].

### Data Structures
*   **Particle Data**: List of particles with fields $\mathbf{p}_j$ (position) and $m_j$ (mass) [S1, unofficial].
*   **Ray Parameters**:
    *   $\mathbf{e}$: Eye position [S1, unofficial].
    *   $\mathbf{v}$: Ray direction [S1, unofficial].
    *   $\lambda_{min}$: Minimal distance between $\mathbf{e}$ and the particle bounding box [S1, unofficial].
    *   $\lambda_{max}$: Maximum ray length, calculated based on the back-side of the particle bounding box [S1, unofficial].
    *   $\Delta \lambda$: Volume sampling distance [S1, unofficial].
    *   $\#s$: Number of iterations for the inner loop [S1, unofficial].

### Steps
1.  **Ray Setup**: Determine $\lambda_{min}$ as the minimal distance between the eye $\mathbf{e}$ and the bounding box of the particles. Determine $\lambda_{max}$ based on the back-side of the particle bounding box. Store ray direction $\mathbf{v}$ and start position at the front face of the bounding box [S1, unofficial].
2.  **Shell Subdivision**: Subdivide the viewing volume in the viewing direction into concentric spherical shells centered at the eye $\mathbf{e}$. Place the shells equally separated by the volume sampling distance $\Delta \lambda$. Position the first slice at $\lambda_{min}$ to skip empty space in front of the camera [S1, unofficial].
3.  **Iterative Loop**: Execute a loop for $\#s$ times. Each iteration performs three inner steps:
    a.  **Sampling Position Update**: Update the sampling position along the ray based on the current shell distance [S1, unofficial].
    b.  **Density Reconstruction**: Reconstruct the density field from particles via perspectively correct splatting of particle density footprints at the sampling depths used by volume ray casting. This is performed in screen space at viewport resolution [S1, unofficial].
    c.  **Volume Ray-Casting**: Perform the standard ray casting steps (sampling, classification, compositing) using the reconstructed density [S1, unofficial].

### Density Reconstruction Formula
The density $\bar{\rho}(\mathbf{p})$ at a point $\mathbf{p}$ is reconstructed using a radial symmetric kernel function $W \in C^2[0, h]$ with support radius $h$, particle positions $\mathbf{p}_j$, and particle masses $m_j$ [S1, unofficial]:

$$ \bar{\rho}(\mathbf{p}) = \frac{1}{N} \sum_{j} m_j W(\|\mathbf{p} - \mathbf{p}_j\|) $$

The result is normalized so that the average density evaluates to 1 for easier iso-value evaluation, i.e., $\bar{\rho}(\mathbf{p}) = 1$ [S1, unofficial]. For each particle $j$, a symmetric polynomial density kernel function (the smooth step function) with finite support radius $h$ is used, though any other function with finite support could be used [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sliced_ray_casting(particles, eye, ray_direction, lambda_min, lambda_max, delta_lambda, num_iterations, h):
    """
    Performs sliced ray casting with on-demand volume reconstruction.
    
    Args:
        particles: List of dicts with 'pos' (vector) and 'mass' (float).
        eye: Vector, position of the camera eye.
        ray_direction: Vector, direction of the ray.
        lambda_min: Float, minimal distance to particle bounding box.
        lambda_max: Float, maximum ray length.
        delta_lambda: Float, volume sampling distance.
        num_iterations: Int, number of inner loop iterations.
        h: Float, support radius of the kernel function.
        
    Returns:
        color: Vector, final composited color.
        opacity: Float, final opacity.
    """
    # Initialize compositing variables
    accumulated_color = [0.0, 0.0, 0.0]
    accumulated_opacity = 0.0
    
    # Iterate through sampling depths
    for i in range(num_iterations):
        # 1. Sampling Position Update
        current_lambda = lambda_min + i * delta_lambda
        
        # Check if we have exceeded the maximum ray length
        if current_lambda > lambda_max:
            break
            
        # Sample point on the ray (shells centered at eye)
        sample_point = [
            eye[0] + ray_direction[0] * current_lambda,
            eye[1] + ray_direction[1] * current_lambda,
            eye[2] + ray_direction[2] * current_lambda
        ]
        
        # 2. Density Reconstruction (On-Demand Splatting)
        density = 0.0
        for particle in particles:
            dx = sample_point[0] - particle['pos'][0]
            dy = sample_point[1] - particle['pos'][1]
            dz = sample_point[2] - particle['pos'][2]
            dist = math.sqrt(dx*dx + dy*dy + dz*dz)
            
            # Check if particle is within support radius h
            if dist <= h:
                r_over_h = dist / h
                # Smooth step function kernel
                w = 1.0 - 3.0 * r_over_h * r_over_h + 2.0 * r_over_h * r_over_h * r_over_h
                density += particle['mass'] * w
        
        # Normalize density to average 1
        density = density / len(particles)
        
        # 3. Volume Ray-Casting
        # Classification: Map density to color and opacity [G]
        color_sample = [density, density, density]
        alpha_sample = 1.0 - math.exp(-density * delta_lambda)
        
        # Compositing (front-to-back)
        weight = (1.0 - accumulated_opacity) * alpha_sample
        accumulated_color[0] += weight * color_sample[0]
        accumulated_color[1] += weight * color_sample[1]
        accumulated_color[2] += weight * color_sample[2]
        accumulated_opacity += weight
        
        # Early-z-test culling / Termination [S1, unofficial]
        if accumulated_opacity > 0.99:
            break
            
    return accumulated_color, accumulated_opacity
```

### Parameters
*   $\Delta \lambda$: Volume sampling distance that equally separates the concentric spherical shells [S1, unofficial].
*   $\lambda_{min}$: Minimal distance between the eye $\mathbf{e}$ and the bounding box of the particles; determines the first slice position [S1, unofficial].
*   $\lambda_{max}$: Maximum ray length, calculated based on the back-side of the particles bounding box [S1, unofficial].
*   $\#s$: Number of iterations for the inner loop that performs sampling position update, density reconstruction, and volume ray-casting [S1, unofficial].
*   $h$: Support radius of the kernel function [S1, unofficial].

### Edge Cases and Pitfalls
*   **Perspective Correction**: Perspective correction is handled by moving the correction step from volume rendering to data reconstruction, performing density reconstruction in screen space at viewport resolution [S1, unofficial].
*   **Empty Space**: The first slice is placed at $\lambda_{min}$ to skip empty space in front of the camera [S1, unofficial].
*   **Memory Footprint**: The method avoids large memory footprints because volume data is only partially created on-demand [S1, unofficial].

## Operating characteristics
*   **Performance**: The method maintains comparative rendering performance to traditional methods while reducing memory requirements [S1, unofficial].
*   **Quality**: It yields superior image quality due to image-space evaluation, avoiding aliasing artifacts common in particle visualization and object-space volume ray casting [S1, unofficial].
*   **Memory**: Memory requirements are reduced because the volume data is only partially created on-demand [S1, unofficial].
*   **Hardware**: The process is controlled by the CPU but the inner steps are performed solely on the GPU [S1, unofficial].
*   **Optimization**: A novel method for termination of the ray casting based on inter-frame occlusion queries is employed to optimize overall performance, exploiting frame-to-frame coherency to compensate for overhead and latencies [S1, unofficial].

## Background
The method was proposed in the paper "Interactive Image-Space Volume Visualization for Dynamic Particle Simulations" by Falk, Grottel, and Ertl [S1, unofficial]. It is applied to systems biology (cell signaling propagation), biochemistry (protein surface visualization), and molecular dynamics (laser ablation processes) [S1, unofficial]. The approach combines ideas from texture slicing, ray casting, and splatting. Early implementations blended object-space-aligned texture stacks, and view-aligned texture stacks were superseded by single-pass volume ray casting [S1, unofficial].

## Sources
[S1] Interactive Image-Space Volume Visualization for Dynamic Particle Simulations, https://openalex.org/W2284871597, fetched 2026-10-30 (unofficial)
