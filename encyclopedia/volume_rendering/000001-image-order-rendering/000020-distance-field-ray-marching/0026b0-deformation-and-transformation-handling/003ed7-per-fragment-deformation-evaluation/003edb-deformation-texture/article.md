# Deformation Texture

Also called: Hardware-Accelerated Deformation via Texture Interpolation, Inverse Deformation Texture, Sampled Deformation Function

## Overview
Deformation Texture is a technique for evaluating complex, computationally expensive deformation functions during volume rendering by encoding the inverse deformation mapping into a 3D texture [S2]. It solves the problem of "inverse parameterization," where a shader needs to map a fragment's screen-space or ray-space coordinate back to the deformed object space, a task that is often analytically intractable or too costly for real-time fragment programs [S1]. Unlike direct shader evaluation, which is suitable for simple bends and twists, this method is chosen for complex functions like Hierarchical B-Splines and Thin-Plate Splines, trading memory bandwidth for computational simplicity [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A deformation function $D: \mathbb{R}^3 \to \mathbb{R}^3$ (mapping object space to deformed space) or its inverse $D^{-1}$ (mapping deformed space to object space).
*   **Output:** A 3D texture containing samples of the inverse deformation function, used to retrieve deformed coordinates during rendering.

**Data Structures:**
*   **Deformation Texture:** A 3D texture with resolution $N_x \times N_y \times N_z$.
*   **Texture Coordinates:** Unit volume coordinates $(u, v, w) \in [0, 1]^3$ representing positions in the deformed space.
*   **Stored Values:** Each texel stores a 3D vector $\mathbf{p} = (x, y, z)$ representing the corresponding position in the original object space.

**Steps:**
1.  **Grid Definition:** Define a regular 3D grid of points covering the unit volume of the deformed space. The resolution $N$ is determined by the Nyquist theorem, requiring at least two lattice sections per pixel, though practical resolution depends on curvature, pixel coverage, and fire length, ranging from 1 to several hundred sections [S1].
2.  **Inverse Evaluation:** Evaluate the inverse deformation function $D^{-1}(\mathbf{u})$ for every point $\mathbf{u}$ on the regular 3D grid. If only the forward deformation $D(\mathbf{p})$ is available, compute the inverse using scattered data interpolation on the CPU [S2].
3.  **Texture Population:** Store the resulting object-space coordinates $\mathbf{p}$ into the 3D texture at the corresponding texture coordinates $\mathbf{u}$ [S2].
4.  **Fragment Shader Evaluation:** In the fragment shader, use the current fragment's position in deformed space (or ray origin) as texture coordinates to perform a single 3D texture lookup. The graphics hardware's trilinear interpolation implicitly evaluates the piecewise-linear approximation of $D^{-1}(\mathbf{u})$ [S1, S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pre-rendering phase (CPU)
def create_deformation_texture(forward_deform_func, resolution):
    """
    Creates a 3D texture storing the inverse deformation.
    forward_deform_func: Function D(p) mapping object space to deformed space.
    resolution: Integer N, defining NxNxN grid.
    """
    texture_data = empty_3d_array(resolution, resolution, resolution, dtype=float32, channels=3)
    
    # If inverse is not known analytically, use scattered data interpolation
    # to approximate D_inv from samples of D.
    # For this pseudocode, we assume we have a function inv_deform(u)
    # that computes D_inv(u).
    
    for i in range(resolution):
        for j in range(resolution):
            for k in range(resolution):
                # Normalize to [0, 1]
                u = i / (resolution - 1)
                v = j / (resolution - 1)
                w = k / (resolution - 1)
                
                # Evaluate inverse deformation
                # If only forward is available, this step uses scattered data interpolation
                # to find p such that D(p) ≈ (u, v, w)
                p = inv_deform(u, v, w) 
                
                texture_data[i, j, k] = p
                
    return texture_data

# Rendering phase (Fragment Shader)
def fragment_shader_deformation(texture, ray_origin_deformed_space):
    """
    Retrieves the object-space coordinate for a given deformed-space coordinate.
    """
    # ray_origin_deformed_space is the position in the deformed volume space
    # typically derived from ray marching or screen coordinates
    
    # Single texture lookup with hardware trilinear interpolation
    # This returns the approximate object-space coordinate
    object_space_coord = texture_sample_3d(texture, ray_origin_deformed_space)
    
    return object_space_coord
```

**Gradient Computation:**
Because the gradient of the volume data depends on the deformation, it cannot be precomputed. It is computed "on the fly" in the fragment shader using forward finite differences [S2]:
$$
\nabla v(\mathbf{u}(O)) \approx \left( v(\mathbf{u}(O_x + \Delta O_x, O_y, O_z)) - v(\mathbf{u}(O, O_y, O_z)), \dots \right)
$$
where $\mathbf{u}$ is the deformation function and $O$ is the origin [S2].

**Parameters:**
*   **Lattice Resolution:** Ranges from a single section to several hundred sections. Higher resolution reduces visual artifacts but increases memory usage [S1].
*   **Hardware Interpolation:** Relies on linear (trilinear) interpolation. Discontinuities in the first derivative may occur at lattice section borders [S1].

**Edge Cases and Pitfalls:**
*   **Derivative Discontinuities:** Hardware linear interpolation creates discontinuities in the first derivative of the curve at the borders of each lattice section [S1].
*   **Smoothness Loss:** Piecewise linear approximation may fail to capture high frequencies or desired smoothness unless a large number of samples is used [S2].
*   **Performance Bound:** Fragment programs often become texture-lookup bound [S2].
*   **Inverse Computation:** If only the forward deformation is available, the inverse must be computed via scattered data interpolation, which is expensive [S2].

## Operating characteristics
*   **Time Complexity:** Rendering time is $O(1)$ per fragment for deformation evaluation, as it requires only a single 3D texture lookup [S2]. Pre-processing time is $O(N^3)$ where $N$ is the texture resolution, due to dense evaluation of the deformation function [S2].
*   **Memory Complexity:** $O(N^3)$ for storing the deformation texture [S2].
*   **Performance:** On circa 2005 hardware (NVIDIA GeForce 6800GT), simple deformations (bends, twists) are faster in shaders, while complex deformations (Hierarchical B-Splines, Thin-Plate Splines) benefit from this method [S2]. The method is texture-lookup bound [S2].
*   **Strengths:** Decouples complex deformation logic from the fragment shader, allowing real-time evaluation of functions that are otherwise intractable [S1, S2].
*   **Limitations:** Requires pre-computation on the CPU; piecewise linear approximation may introduce artifacts; memory intensive for high resolutions [S2].
*   **When to Use:** For complex deformation functions (e.g., splines) that are too expensive to evaluate per-fragment [S2].
*   **When Not to Use:** For simple deformations (bends, twists) which are better handled directly in shaders [S2].

## Background
The method was employed in the movie "Shrek" and is based on the model described by Lamorlette and Foster [S1]. The idea of storing a sampled representation of a function in a 3D texture was suggested in [RSSSG01] [S2]. The technique addresses the "inverse parameterization" problem, where the shader operates on the deformed point $P'$ to return the original point $P$, rather than the forward mapping $D(P)$ [S1].

## Sources
[S1] Real-time procedural volumetric fire, https://doi.org/10.1145/1230100.1230131, fetched 2026-11-05
[S2] Integrating Dynamic Deformations into Interactive Volume Visualization, https://doi.org/10.2312/vissym.eurovis06.219-226, fetched 2026-10-31
