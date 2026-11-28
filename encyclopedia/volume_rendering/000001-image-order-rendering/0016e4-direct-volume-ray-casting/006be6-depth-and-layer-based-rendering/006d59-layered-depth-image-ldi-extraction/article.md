# Layered Depth Image (LDI) extraction

Also called: LDI extraction

## Overview
LDI extraction is a view-dependent method that reduces the ray traversal count to the scene's depth complexity by generating a buffer of depth layers per pixel [S1, unofficial]. Unlike [Two-Pass Depth Ray Casting](3377-two-pass-depth-ray-casting.md) which uses geometric bounding boxes, or [Multi-layer Volume Ray Casting](9441-multi-layer-volume-ray-casting.md) which uses pre-computed static layers, LDI extraction derives intervals dynamically from rendered depth images, making it sensitive to the specific model and view [S1, unofficial].

## How it works
**Inputs:** 3D volume data; camera parameters (position, orientation, projection).
**Outputs:** A Layered Depth Image (LDI) buffer, typically structured as a list of depth values (and optionally color/opacity) for each pixel.

**Data Structures:**
The LDI buffer is an array of lists, one per pixel $(u, v)$. Each list contains entries sorted by depth $t$. An entry typically contains:
- $t$: depth value along the ray.
- $color$: RGBA or density value (optional, depending on implementation).
- $opacity$: $\alpha$ value (optional).

**Steps:**
1. **Render Depth Pass:** Render the volume's bounding geometry or a simplified representation to a depth buffer (Z-buffer) to obtain the visible surface depths. [S1, unofficial]
2. **Extract Depths:** For each pixel $(u, v)$, read the depth values from the depth buffer. If multiple layers are required (e.g., for internal structures), this step may involve multiple render passes or specialized hardware queries to extract all front-to-back depth intersections. [S1, unofficial]
3. **Sort Depths:** The extracted depth values for each pixel must be sorted in front-to-back order ($t_1 < t_2 < \dots < t_n$). This sorting is performed on the CPU. [S1, unofficial]
4. **Construct LDI:** Populate the LDI buffer with the sorted depth lists. Each list represents the non-overlapping ray segments for that pixel. [S1, unofficial]
5. **Ray Casting (Downstream):** Use the LDI buffer to define integration intervals $[t_{start}, t_{end}]$ for the volume rendering equation, as described in [Depth and Layer-Based Rendering](20975-depth-and-layer-based-rendering.md).

**Pitfalls:**
- **CPU Bottleneck:** The method requires downloading depth images from video memory and resorting depth lists pixel-by-pixel on the CPU, which significantly impacts efficiency, especially at high volume resolutions [S1, unofficial].
- **View Dependence:** The LDI must be regenerated for every new view, making it unsuitable for static scenes with camera movement unless caching strategies are employed.

## Subtopics

### [Volumetric Depth Sprite (VDS)](23422-volumetric-depth-sprite-vds.md)
Extends LDI by storing depth values in a 3D texture for fast point-in-volume tests, optimized for volumetric parity rather than just rendering.

## Operating characteristics
- **Time Complexity:** Dominated by the depth extraction and CPU sorting steps. Efficiency is heavily affected by volume resolution and the number of depth layers per pixel [S1, unofficial].
- **Memory Complexity:** Proportional to the number of pixels times the average depth complexity (number of layers per pixel).
- **Strengths:** Reduces traversal to actual depth complexity; simple and robust implementation [S1, unofficial].
- **Limitations:** High CPU overhead due to data transfer and sorting; view-dependent; not suitable for real-time if depth complexity is high or resolution is very large [S1, unofficial].

## Background
LDI extraction was presented as part of a scheme for real-time voxelization of complex models, aiming to represent resultant volumes as 2D textures for efficient implementation on PC graphics hardware [S1, unofficial].

## Sources
[S1] Real-time Voxelization for Complex Models, https://www.researchgate.net/publication/252533918_Real-time_Voxelization_for_Complex_Models, fetched 2026-11-03 (unofficial)
