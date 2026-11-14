# Motion-Blurred Splats

Also called: Motion Blur in Volume Rendering, Temporal Antialiasing Technique for Splatting

## Overview
Motion-Blurred Splats is a technique for approximating temporal integration in direct volume ray casting by stretching the spatial footprint of each splat along its projected motion vector, thereby smoothing flickering and scintillation artifacts caused by rapid movement or spinning [S1, unofficial]. It differs from [Splatting for Anti-aliasing Small Cells](829-splatting-for-anti-aliasing-small-cells.md) and [Prefiltering](2928-prefiltering.md) by addressing temporal aliasing through spatial footprint deformation rather than spatial frequency filtering or pre-computation, and it differs from [Motion Blur via Splat Smearing](1593-motion-blur-via-splat-smearing.md) by operating on the continuous footprint geometry level rather than discretizing the smear into a voxel buffer [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A voxel dataset with associated linear motion vectors for each voxel, a static radial splat radius $s$, and a static footprint area $A_1$.
*   **Output:** A rendered image where moving objects exhibit motion blur, with energy conserved relative to stationary splats.

**Data Structures:**
*   **Voxel Data:** Each voxel contains position, opacity, color, and a 3D linear motion vector $\mathbf{v}$.
*   **Splat Footprint:** Defined by a radial profile $s$ and a geometric shape (circle for static, composite for motion).

**Steps:**
1.  **Transform Motion Vector:** For each voxel, associate its linear motion vector $\mathbf{v}$. Transform $\mathbf{v}$ to eye space. Project this vector onto the viewing plane to determine the projected direction and length $l$ [S1, unofficial].
2.  **Determine Footprint Geometry:**
    *   If the projected length $l$ is zero, draw a standard circular splat with radius $s$ and static opacity.
    *   If $l > 0$, construct a motion-blurred footprint. The original circular footprint is conceptually cut in half and stretched between the start position $t_{i-1}$ and end position $t_i$ of the motion interval. This results in a composite shape consisting of:
        *   Two half-circles (A and B) of radius $s$ at the endpoints.
        *   A rectangle (C) connecting the half-circles, with length $l$ and width $2s$ [S1, unofficial].
3.  **Rotate Footprint:** For arbitrary projected velocity vector directions, rotate the motion-blurred splat footprint in the proper direction using a rotation scheme similar to Crawfis and Max [S1, unofficial].
4.  **Adjust Opacity for Energy Conservation:** To maintain constant energy contribution, the integrated intensity of the motion-blurred splat must equal that of a single stationary splat. Calculate the area of the motion-blurred footprint $A_2$ and scale the splat opacity $\alpha$ accordingly [S1, unofficial].
5.  **Rasterize:** Draw the motion-blurred splat using the rotated composite footprint and the scaled opacity.

**Formulas:**
*   **Motion-Blurred Footprint Area:**
    $$A_2 = A_1 + 2ls$$
    Where:
    *   $A_1$ is the area of the static circular footprint ($\pi s^2$).
    *   $l$ is the distance between $t_{i-1}$ and $t_i$ (projected motion length).
    *   $s$ is the radial splat radius.
*   **Opacity Scaling:**
    $$\alpha_{new} = \alpha_{static} \times \frac{A_1}{A_2}$$
    Where:
    *   $\alpha_{static}$ is the original opacity of the voxel.
    *   $\alpha_{new}$ is the adjusted opacity for the motion-blurred splat [S1, unofficial].

**Parameters:**
*   **Splat Radius ($s$):** Not given in the sources.
*   **Motion Vector:** Provided per voxel.

**Edge Cases and Pitfalls:**
*   **Visibility/Popping:** The algorithm does not address the visibility problem (changing hidden surfaces) or popping artifacts caused by voxel sorting changes; it only addresses integrated energy across the time domain [S1, unofficial].
*   **High Velocity:** At very high velocities, the motion blur dominates the visual result, potentially masking the benefits of spatial anti-aliasing [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_motion_blurred_splat(voxel, camera, time_interval):
    """
    Renders a single voxel as a motion-blurred splat.
    
    Args:
        voxel: Object containing position, opacity (alpha), color, and motion_vector (3D).
        camera: Object containing projection matrix and eye-space transformation.
        time_interval: Tuple (t_start, t_end) representing the time step.
        
    Returns:
        A list of rasterization commands or a modified splat geometry.
    """
    # 1. Transform motion vector to eye space
    # Assuming voxel.motion_vector is in world or object space
    eye_space_vector = camera.transform_to_eye_space(voxel.motion_vector)
    
    # 2. Project to viewing plane to get length and direction
    # This step assumes a standard perspective or orthographic projection
    projected_vector = camera.project_vector(eye_space_vector)
    l = length(projected_vector)
    direction = normalize(projected_vector)
    
    # Static parameters
    s = voxel.splat_radius # Not given in sources, assumed known
    A1 = pi * (s ** 2)
    alpha_static = voxel.opacity
    
    if l == 0:
        # Standard static splat
        return draw_circular_splat(voxel.position, s, alpha_static, voxel.color)
    
    # 3. Calculate motion-blurred footprint area
    # A2 = A1 + 2 * l * s
    A2 = A1 + 2 * l * s
    
    # 4. Adjust opacity for energy conservation
    alpha_new = alpha_static * (A1 / A2)
    
    # 5. Construct and rotate footprint
    # The footprint is a capsule: two half-circles connected by a rectangle
    # Center of the capsule is the midpoint of the motion
    start_pos = voxel.position - (projected_vector / 2)
    end_pos = voxel.position + (projected_vector / 2)
    
    # Create capsule geometry
    capsule = create_capsule(start_pos, end_pos, s)
    
    # Rotate capsule to align with projected direction
    # (Assuming create_capsule creates an axis-aligned capsule, rotation is needed)
    rotated_capsule = rotate_geometry(capsule, direction)
    
    # 6. Rasterize
    return draw_capsule_splat(rotated_capsule, alpha_new, voxel.color)
```

## Subtopics

### [Motion Blur via Splat Smearing](1593-motion-blur-via-splat-smearing.md)
Approximates motion blur by distributing sample values evenly across voxels touched by the motion vector in a voxel buffer. Choose this when working with voxel-based rendering pipelines that support buffer accumulation; it is less accurate for lighting but efficient.

### [Motion Blur via Vertex Displacement](11526-motion-blur-via-vertex-displacement.md)
Generates motion blur by shifting vertices along velocity vectors on the mesh geometry level. Choose this for procedurally generated fluid meshes with changing topology; it differs from footprint stretching by operating on geometry rather than rasterization.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels, as each voxel requires a constant number of transformations and footprint calculations.
*   **Memory Complexity:** Linear with respect to the number of voxels to store motion vectors.
*   **Measured Performance:** Applied to a 512 x 512 x 76 nerve cell dataset from confocal microscopy. Motion blur dominated the visual result, making the difference between antialiased and motion-blurred splats less visible for objects with high movement [S1, unofficial].
*   **Strengths:** Smooths temporal artifacts (flickering, scintillation, strobing) caused by rapid movement or spinning. Simple to implement as a modification to existing splatting pipelines.
*   **Limitations:** Does not address visibility changes or popping artifacts from sorting. Approximates motion as linear, which may be inaccurate for complex non-linear motion.
*   **When to Use:** When temporal aliasing is the primary artifact and motion is relatively linear and smooth.
*   **When Not to Use:** When visibility changes (occlusion) are frequent or when high-fidelity lighting integration over time is required.

## Background
Originating from work by Mueller et al. (1998), this technique extends splatting to handle motion blur by approximating the motion of a volume defined on a raster grid as a linear motion of the reconstruction basis functions at each voxel [S1, unofficial]. It was developed to address the lack of motion blurring techniques in volume rendering at the time, presenting a fruitful area for future research [S1, unofficial].

## Sources
[S1] DTIC ADA606764: Splatting Errors and Antialiasing, https://archive.org/details/DTIC_ADA606764, fetched 2026-10-26 (unofficial)
