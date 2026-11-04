# Image-Parallel Ray Casting via 2D Texture Mapping

Also called: Slice-based Volume Rendering, Texture-Mapped Volume Rendering (2D)

## Overview
Image-Parallel Ray Casting via 2D Texture Mapping is a hardware-accelerated direct volume rendering technique that replaces per-ray processing with per-slice processing. It projects parallel slices of the volumetric dataset onto the image plane and composites them using standard graphics blending operations. Unlike standard ray casting which processes pixels independently, this method processes all pixels in a slice in parallel, leveraging 2D texture mapping hardware for resampling and hardware blending for compositing. It is distinct from its parent, the Planes Method, which relies on 3D texture hardware; this method uses only 2D texture units, making it compatible with older graphics architectures while maintaining equivalence to standard ray casting results.

## How it works
The method transforms the volume rendering integral into a sequence of 2D texture mapping and blending operations.

### Inputs and Outputs
- **Input**: A volumetric dataset $V$ (scalar field), a viewing direction $\vec{v}$, and a projection matrix $P$.
- **Output**: A 2D image (color and opacity buffer) representing the volume.

### Data Structures
1.  **Slice Stack**: A list of 2D texture images. Each image represents a planar cross-section of the volume.
    -   **Layout**: Each slice is a 2D array of texels. Each texel contains interpolated scalar values from $V$.
    -   **Generation**: Slices are either pre-computed orthogonal to a primary axis (static) or generated dynamically orthogonal to $\vec{v}$ (dynamic).
2.  **Projection Geometry**:
    -   For static slices: A set of projection matrices mapping the fixed slice planes to the image plane.
    -   For dynamic slices: A single projection matrix mapping the view-orthogonal slice plane to the image plane.

### Steps
1.  **Slice Generation**:
    -   Divide the volume into $N$ parallel planes orthogonal to the viewing direction $\vec{v}$.
    -   If using static slices (pre-computed), these planes are orthogonal to a primary axis (e.g., $z$-axis). If using dynamic slices, they are orthogonal to $\vec{v}$.
    -   For each slice plane, extract the scalar values from $V$ at the intersection points.
    -   [G] If the slice geometry is non-rectangular (common in dynamic view-orthogonal slicing), generate a polygon mesh defining the slice boundary.

2.  **Projection and Texture Mapping**:
    -   For each slice $i$ from $1$ to $N$:
        -   Project the slice geometry onto the image plane using the appropriate projection matrix.
        -   Bind the slice data as a 2D texture.
        -   Render the projected slice geometry. During rasterization, for each covered pixel, the hardware performs bi-linear interpolation of the texture coordinates $(s, t)$ to retrieve the scalar value $s_i$.
        -   Apply the transfer function to map $s_i$ to color $C_i$ and opacity $\alpha_i$.

3.  **Compositing**:
    -   Composite the rendered slices into the final image buffer.
    -   Use the standard front-to-back or back-to-front compositing operator provided by the graphics API (e.g., OpenGL blending).
    -   For front-to-back compositing, the accumulated color $C_{acc}$ and opacity $\alpha_{acc}$ are updated for each new slice with color $C_i$ and opacity $\alpha_i$:
        $$ \alpha_{new} = \alpha_{acc} + (1 - \alpha_{acc}) \alpha_i $$
        $$ C_{new} = C_{acc} + (1 - \alpha_{acc}) C_i \alpha_i $$
    -   [G] Initialize $C_{acc} = 0$ and $\alpha_{acc} = 0$ before the first slice.

### Formulas
The method is exactly equivalent to standard ray casting, provided that precision is maintained during the compositing step. The equivalence holds because the set of sample points along all rays is identical to the set of sample points on the parallel slices.

### Parameters
-   **Slice Spacing**: The distance between adjacent slices. Must be small enough to avoid visible gaps (aliasing) along the viewing direction. Typically equal to the voxel size or smaller.
-   **Interpolation**: Bi-linear interpolation is standard for texture sampling.

### Edge Cases and Pitfalls
-   **View Direction Parallel to Slices**: If pre-computed slices are orthogonal to a primary axis and the view direction is parallel to those slices, the projection degenerates. The slice appears as a line or has zero area, resulting in no image generation or severe distortion [S1].
-   **Arbitrary Viewpoints with Static Slices**: Using pre-computed slices for arbitrary viewpoints causes geometric distortion because the projection of a planar slice onto the image plane is not a simple affine transformation of the texture coordinates if the view is oblique. This degrades image quality [S1].
-   **Dynamic Slice Complexity**: Generating slices orthogonal to the viewing direction for each frame ensures optimal image quality but requires regenerating the slice stack every frame. The resulting slice geometry can be complex (triangles to hexagons) rather than rectangular, requiring more complex polygon rendering [S1].

## Operating characteristics
-   **Time Complexity**: $O(N \cdot W \cdot H)$, where $N$ is the number of slices, and $W \times H$ is the image resolution. The complexity is dominated by the rasterization and blending of $N$ slices.
-   **Memory Complexity**: $O(N \cdot W_{slice} \cdot H_{slice})$ for storing the slice textures, where $W_{slice}, H_{slice}$ are the dimensions of the slice projections.
-   **Performance**: Highly efficient on hardware supporting 2D texture mapping and blending. It offloads interpolation and compositing to fixed-function pipeline units.
-   **Strengths**: Compatible with older graphics hardware lacking 3D texture support. Simple implementation using standard graphics APIs.
-   **Limitations**: Image quality degrades with static slices at oblique angles. Dynamic slices incur high CPU overhead for slice generation and geometry management.

## Background
This method was developed as a way to accelerate volume rendering on graphics hardware that supported 2D texture mapping but not 3D texture mapping. It was described in the context of interactive volume rendering techniques, specifically as a precursor to 3D texture mapping methods. The approach leverages the fact that ray casting samples can be grouped into parallel planes, allowing the use of standard texture mapping hardware for resampling.

## Sources
[S1] Interactive Volume Rendering Using 3D Texture-Mapping ..., https://web.cs.ucdavis.edu/~okreylos/PhDStudies/Winter2000/TextureMapping.html, fetched 2026-10-27
