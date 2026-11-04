# Motion-Blurred Splats

Also called: Motion Blur in Volume Rendering, Temporal Antialiasing Technique for Splatting

## Overview
Motion-blurred splats approximate temporal integration in volume rendering by stretching the spatial footprint of each voxel splat along its projected motion vector, thereby smoothing high-frequency temporal artifacts such as flickering and scintillation during rapid camera or object movement [S1, unofficial]. Unlike standard splatting which suffers from aliasing in animation, or [Threshold Slicing and Anti-Aliasing](1447-threshold-slicing-and-anti-aliasing.md) which addresses spatial frequency, this method specifically targets the time domain by distributing energy over the motion path, though it does not resolve visibility changes or popping artifacts caused by sorting order variations [S1, unofficial].

## How it works
The method extends object-order splatting by associating a linear motion vector with each voxel and modifying the splat's geometric footprint to represent the path traveled during a frame interval.

**Inputs and Outputs**
*   **Input:** A volume dataset defined on a raster grid; for each voxel, a linear motion vector $\vec{v}$ (typically derived from camera velocity or object transformation); a static radial splat radius $s$; and a base opacity $\alpha$.
*   **Output:** A rendered image where moving objects exhibit temporal blur proportional to their velocity.

**Data Structures**
Each voxel sample carries:
*   Position $(x, y, z)$
*   Value (opacity/density)
*   Motion vector $\vec{v}_{world}$ in world space.

**Steps**
1.  **Transform Motion Vector:** For each voxel, transform its linear motion vector $\vec{v}_{world}$ into eye space. Project this vector onto the viewing plane to determine the 2D direction and length $l$ of the motion blur [S1, unofficial].
2.  **Construct Footprint Geometry:** For radially symmetric footprint functions, approximate the motion by stretching the footprint in the direction of the projected velocity vector. The motion-blurred splat is drawn between positions $t_{i-1}$ and $t_i$. The original circular footprint is conceptually cut in half and stretched between these positions, resulting in a shape composed of:
    *   Two half-circles (A and B) at the endpoints, each with radius $s$.
    *   A rectangle (C) connecting the two half-circles, with length $l$ (distance between $t_{i-1}$ and $t_i$) and width $2s$ [S1, unofficial].
3.  **Rotate Footprint:** For arbitrary projected velocity vector directions, rotate the constructed motion-blurred splat (the combination of half-circles and rectangle) to align with the velocity direction, using a rotation scheme similar to that of Crawfis and Max [S1, unofficial].
4.  **Adjust Opacity for Energy Conservation:** To maintain constant energy contribution, the integrated intensity of the motion-blurred splat must equal that of a single stationary splat. Since the area increases, the opacity is scaled down.
    *   Let $A_1$ be the area of the static circular footprint ($\pi s^2$).
    *   Let $A_2$ be the area of the motion-blurred footprint.
    *   The area is calculated as:
        $$ A_2 = A_1 + 2ls $$
        where $l$ is the distance between $t_{i-1}$ and $t_i$, and $s$ is the radial splat radius [S1, unofficial].
    *   The new opacity $\alpha'$ is scaled by the ratio of the static area to the blurred area:
        $$ \alpha' = \alpha \cdot \frac{A_1}{A_2} = \alpha \cdot \frac{\pi s^2}{\pi s^2 + 2ls} $$
5.  **Rasterize:** Draw the rotated, opacity-scaled motion-blurred splat onto the frame buffer.

**Parameters**
*   $s$: Radial splat radius. Typical values depend on the voxel size relative to the pixel size, often set to 1.0 or slightly larger to cover neighboring voxels [S1, unofficial].
*   $l$: Projected motion length. Determined by the frame rate and velocity.

**Edge Cases and Pitfalls**
*   **Visibility/Popping:** The algorithm does not address the visibility problem (changing hidden surfaces) or popping artifacts caused by voxel sorting changes; it only addresses integrated energy across the time domain [S1, unofficial].
*   **High Velocity:** At very high velocities, the footprint becomes a long thin rectangle. The linear approximation assumes constant velocity within the frame.
*   **Precision:** The opacity scaling factor approaches zero as $l$ increases, potentially leading to underflow or very faint rendering for extremely fast-moving objects [S1, unofficial].

## Subtopics

### [Motion Blur via Splat Smearing](1593-motion-blur-via-splat-smearing.md)
A variant that distributes each sample's value evenly across the voxels it touches along its motion vector. It differs by folding the temporal domain into the spatial domain for efficient rasterization, but loses lighting detail and cannot cancel out camera motion relative to primitives.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels $N$, $O(N)$, as each voxel requires a constant number of geometric transformations and a single rasterization pass of an elongated shape.
*   **Memory Complexity:** $O(N)$ to store the motion vectors for each voxel.
*   **Measured Performance:** Applied to a 512 x 512 x 76 nerve cell dataset from confocal microscopy, motion blur dominated the visual result, making the difference between antialiased and motion-blurred splats less visible for objects with high movement [S1, unofficial].
*   **Strengths:** Effectively reduces flickering and scintillation artifacts in animations of spinning or rapidly moving volumes, such as terrain datasets from satellite data [S1, unofficial].
*   **Limitations:** Does not solve visibility sorting issues; the approximation of smearing a circular footprint into a capsule shape is an approximation and may not perfectly integrate energy for non-radial kernels or complex motion paths [S1, unofficial].
*   **When to use:** When temporal aliasing (flickering) is the primary artifact in animated volume rendering and performance constraints prevent full temporal integration or supersampling.
*   **When not to use:** When accurate visibility sorting is critical or when the motion is non-linear/rotational in a way that the linear smearing approximation fails to capture accurately.

## Background
The technique was introduced by Mueller et al. in their work on splatting errors and antialiasing, published in the IEEE Transactions on Visualization and Computer Graphics (Vol. 4, No. 2, April-June 1998) [S1, unofficial]. It builds on earlier work by Max [7] and Crawfis et al. [2] regarding compositing order for X-ray-like or emission volume models, where compositing order is immaterial [S1, unofficial]. The authors noted that motion blurring in volume rendering had not yet been addressed and presented this as initial work in that area [S1, unofficial].

## Sources
[S1] DTIC ADA606764: Splatting Errors and Antialiasing, https://archive.org/details/DTIC_ADA606764, fetched 2026-10-26 (unofficial)
