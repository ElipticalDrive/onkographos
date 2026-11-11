# Motion-Blurred Splats

Also called: Motion Blur in Volume Rendering, Temporal Antialiasing Technique for Splatting

## Overview
Motion-Blurred Splats extends standard splatting to handle temporal aliasing by approximating the integration of voxel energy over a shutter interval [S1, unofficial]. It solves the flickering and scintillation artifacts caused by rapid movement or spinning in volume rendering by smoothing temporal variations, though it does not address visibility changes or popping artifacts caused by sorting order changes [S1, unofficial]. Unlike standard splatting, which suffers from aliasing during rapid motion, this method integrates energy over time by stretching the spatial footprint of each splat along its projected motion vector [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of voxels, each defined by position $\mathbf{p}_k$, density/extinction coefficient $\rho_k$, a static radial reconstruction kernel radius $s$, and a linear motion vector $\mathbf{v}_k$ in volume space.
*   **Output:** A 2D image where each pixel accumulates the integrated energy of motion-blurred splats.

**Data Structures:**
Each voxel $k$ is associated with a linear motion vector. The algorithm requires the static footprint area $A_1$ and the projected motion parameters.

**Numbered Steps:**

1.  **Vector Transformation:** For each voxel $k$, associate a linear motion vector $\mathbf{v}_k$. Transform this vector into eye space and project it onto the viewing plane to determine the projected vector direction and length $l$ [S1, unofficial].
2.  **Footprint Construction:** For radially symmetric footprint functions, approximate the motion by smearing (stretching) the footprint function in the direction of the projected velocity vector [S1, unofficial].
    *   The motion-blurred splat is drawn between the start position $t_{i-1}$ and end position $t_i$ of the shutter interval.
    *   The original circular footprint (radius $s$) is conceptually cut in half and stretched between $t_{i-1}$ and $t_i$ [S1, unofficial].
    *   This results in a composite shape consisting of:
        *   Two half-circles (A and B) at the endpoints, each with radial profile $s$.
        *   A rectangle (C) connecting the half-circles, with length $l$ (distance between $t_{i-1}$ and $t_i$) and width $w = 2s$ [S1, unofficial].
    *   The profile of the rectangle $C$ is $s$ along any line perpendicular to the motion vector [S1, unofficial].
3.  **Area Calculation:** Calculate the area of the motion-blurred splat footprint $A_2$ using the formula:
    $$ A_2 = A_1 + 2ls $$
    where:
    *   $A_1$ is the area of the static footprint extent ($\pi s^2$ for a circle).
    *   $l$ is the distance between $t_{i-1}$ and $t_i$ (projected motion length).
    *   $s$ is the radial splat radius [S1, unofficial].
4.  **Opacity Adjustment:** Adjust the splat opacity to maintain constant energy contribution. The integrated intensity of the motion-blurred splat must equal the integrated energy of a single stationary splat. Scale the opacity $\alpha_{mb}$ based on the ratio of the static footprint area to the motion-blurred footprint area:
    $$ \alpha_{mb} = \alpha_{static} \cdot \frac{A_1}{A_2} = \alpha_{static} \cdot \frac{A_1}{A_1 + 2ls} $$
    where $\alpha_{static}$ is the opacity used for the non-blurred splat [S1, unofficial].
5.  **Rotation:** For arbitrary projected velocity vector directions, rotate the motion-blurred splat in the proper direction using a scheme similar to Crawfis and Max [S1, unofficial].
6.  **Rasterization and Compositing:** Rasterize the rotated, scaled motion-blurred splats onto the image plane and composite them using standard alpha blending [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_motion_blurred_splats(voxels, camera, shutter_start, shutter_end):
    """
    Render volume using motion-blurred splats.
    
    Args:
        voxels: List of voxel objects with attributes:
            - pos: (x, y, z) position in volume space
            - density: float, density/extinction coefficient
            - alpha: float, base opacity
            - radius: float, s, radial splat radius
            - velocity: (vx, vy, vz) linear motion vector in volume space
        camera: Camera object with methods:
            - world_to_eye(point): transforms point to eye space
            - project(point): projects eye-space point to 2D screen coordinates
            - get_projection_matrix(): returns MVP matrix
        shutter_start: float, t_{i-1}
        shutter_end: float, t_i
        
    Returns:
        image: 2D array of pixel values
    """
    image = initialize_image()
    
    # Pre-calculate time delta
    dt = shutter_end - shutter_start
    
    for voxel in voxels:
        # 1. Transform motion vector to eye space
        # Assume velocity is constant over the shutter interval
        vel_eye = camera.world_to_eye(voxel.velocity)
        
        # Project the motion vector to screen space to get length and direction
        # We need the displacement on the screen plane
        start_pos_eye = camera.world_to_eye(voxel.pos)
        end_pos_eye = camera.world_to_eye(voxel.pos + voxel.velocity * dt)
        
        start_screen = camera.project(start_pos_eye)
        end_screen = camera.project(end_pos_eye)
        
        # Calculate projected length l
        dx = end_screen.x - start_screen.x
        dy = end_screen.y - start_screen.y
        l = sqrt(dx*dx + dy*dy)
        
        # Calculate direction angle for rotation
        angle = atan2(dy, dx)
        
        # 2. Calculate areas
        s = voxel.radius
        A1 = pi * s * s  # Area of static circular footprint
        
        # 3. Calculate motion-blurred area A2
        # Formula: A2 = A1 + 2*l*s
        A2 = A1 + 2 * l * s
        
        # 4. Adjust opacity
        # Scale opacity to maintain constant energy
        alpha_mb = voxel.alpha * (A1 / A2)
        
        # 5. Rasterize the motion-blurred splat
        # The splat is a capsule shape (rectangle with two half-circles)
        # Center of the capsule is the midpoint between start and end screen positions
        mid_screen_x = (start_screen.x + end_screen.x) / 2
        mid_screen_y = (start_screen.y + end_screen.y) / 2
        
        # Rasterize the capsule shape
        # Note: In a real implementation, this would involve scan-converting
        # the capsule and blending with the image buffer.
        rasterize_capsule(
            image,
            center=(mid_screen_x, mid_screen_y),
            radius=s,
            length=l,
            angle=angle,
            density=voxel.density,
            alpha=alpha_mb
        )
        
    return image

def rasterize_capsule(image, center, radius, length, angle, density, alpha):
    """
    Rasterize a capsule (motion-blurred splat) into the image.
    This is a placeholder for the actual scan-conversion logic.
    The capsule consists of:
    - A rectangle of width 2*radius and length 'length'
    - Two half-circles of radius 'radius' at the ends
    """
    # Implementation details for scan-converting a rotated capsule
    # would go here. This function iterates over pixels covered by the shape
    # and blends the contribution based on the kernel profile and opacity.
    pass
```

**Edge Cases and Pitfalls:**
*   **Visibility Artifacts:** The algorithm does not address the visibility problem (changing hidden surfaces) or popping artifacts caused by voxel sorting changes [S1, unofficial]. It only addresses integrated energy across the time domain.
*   **High Velocity:** As velocity increases, the footprint area $A_2$ increases, causing opacity to decrease. If $l$ is very large, the opacity may become negligible, potentially leading to under-sampling if not handled by the compositing order or coverage checks.
*   **Degenerate Input:** If the motion vector is zero ($l=0$), $A_2 = A_1$, and opacity remains unchanged, correctly falling back to standard splatting.

## Subtopics

### [Motion Blur via Splat Smearing](1593-motion-blur-via-splat-smearing.md)
Motion Blur via Splat Smearing is a technique to approximate motion blur in voxel buffers by distributing each sample's value evenly across the voxels it touches along its motion vector, preserving the total sum of values. It differs from correct temporal integration by folding the temporal domain into the spatial domain, which allows for efficient rasterization but results in loss of lighting detail and inability to cancel out camera motion relative to primitives.

### [Motion Blur via Vertex Displacement](11526-motion-blur-via-vertex-displacement.md)
Motion blur for fluid meshes is obtained by shifting each vertex toward the velocity by the shutter time, using the velocity recorded by the simulation, rather than tracing vertices along a transformation path between frames.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels, as each voxel is processed once to compute its motion-blurred footprint and rasterized [S1, unofficial].
*   **Memory Complexity:** Linear with respect to the number of voxels to store motion vectors and parameters [S1, unofficial].
*   **Measured Performance:** Applied to a 512 x 512 x 76 nerve cell dataset from confocal microscopy, motion blur dominated the visual result, making the difference between antialiased and motion-blurred splats less visible for objects with high movement [S1, unofficial].
*   **Strengths:** Smooths temporal aliasing artifacts (flickering, scintillation) caused by rapid movement or spinning [S1, unofficial]. Maintains constant energy contribution by adjusting opacity [S1, unofficial].
*   **Limitations:** Does not address visibility changes or popping artifacts [S1, unofficial]. Approximates motion by stretching the footprint, which may not accurately represent complex non-linear motion or lighting changes over time [S1, unofficial].
*   **When to Use:** When temporal aliasing is the primary artifact and the motion can be approximated as linear over the shutter interval.
*   **When Not to Use:** When visibility changes (occlusion) are significant or when high-fidelity lighting integration over time is required.

## Background
The method was introduced by Mueller et al. in 1998 as an extension to splatting for handling motion blur and antialiasing [S1, unofficial]. It builds on the work of Max [7] and Crawfis et al. [2] regarding compositing order and rotation schemes [S1, unofficial]. The paper "Splatting Errors and Antialiasing" presented initial work on extending splatting to handle motion blur, noting that motion blurring in volume rendering had not yet been addressed and presented a fruitful area for future research [S1, unofficial].

## Sources
[S1] DTIC ADA606764: Splatting Errors and Antialiasing, https://archive.org/details/DTIC_ADA606764, fetched 2026-10-26 (unofficial)
