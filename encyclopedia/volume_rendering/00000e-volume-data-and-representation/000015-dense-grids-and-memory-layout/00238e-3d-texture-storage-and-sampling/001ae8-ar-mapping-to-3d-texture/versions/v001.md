# AR mapping to 3D texture

Also called: Linear alpha blending mapping, SIV texture coordinate mapping

## Overview
AR mapping to 3D texture is a technique for volume visualization that maps scalar data and associated attributes (color and opacity) into a 3D texture using a linear coordinate system, thereby avoiding non-linear transformation formulas during rendering [S1, unofficial]. It solves the performance bottleneck of recalculating texture coordinates for every rotation of parallel polygons by defining coordinates once and storing the geometry in a display list [S1, unofficial]. It is distinguished from Wilson's depth integration by its reliance on linear estimation, which is noted to be less accurate but sufficient for effective visualization [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A set of parallel polygons representing the volume or isosurface, and a 3D texture space.
- Output: A rendered image where color and opacity are composited using linear alpha blending.

**Data Structures:**
- **3D Texture:** A volumetric texture object capable of storing RGBA values. The mapping places color and opacity directly into this texture [S1, unofficial].
- **Display List:** A precomputed group of OpenGL commands cached for execution, used to store the rotated polygon set [S1, unofficial].

**Steps:**
1. **Define Texture Coordinates:** Calculate the linear mapping from the 3D spatial coordinates of the volume to the 3D texture coordinates $(u, v, w)$. This mapping is defined once for the entire set of parallel polygons [S1, unofficial].
   - Let the volume bounds be $[x_{min}, x_{max}] \times [y_{min}, y_{max}] \times [z_{min}, z_{max}]$.
   - For a point $P = (x, y, z)$, the texture coordinates are:
     $$ u = \frac{x - x_{min}}{x_{max} - x_{min}} $$
     $$ v = \frac{y - y_{min}}{y_{max} - y_{min}} $$
     $$ w = \frac{z - z_{min}}{z_{max} - z_{min}} $$
   - [G] Clamp $u, v, w$ to $[0, 1]$ to handle boundary cases.
2. **Populate Texture:** Fill the 3D texture with the scalar data converted to color and opacity values based on the transfer function.
3. **Rotate Polygons:** Apply the desired rotation to the parallel polygons. Because the texture coordinates are defined in the local space of the linear mapping, they do not need to be recalculated for each rotation step [S1, unofficial].
4. **Render:** Use linear alpha blending to composite the fragments. The hardware interpolates the pre-stored color and opacity values from the 3D texture [S1, unofficial].

**Parameters:**
- Not given in the sources.

**Edge Cases and Pitfalls:**
- **Precision:** The linear estimation method is noted to be likely worse than Wilson's depth integration [S1, unofficial].
- **Performance:** Recalculating texture coordinates at each rotation is slow; this method avoids that cost [S1, unofficial].
- **Visual Quality:** Despite the estimation error, visualizations remain effective, especially given the context of the goals [S1, unofficial].

## Subtopics

### [Gradient-Isosurface Texture Packing](889-gradient-isosurface-texture-packing.md)
A data layout strategy where gradients are stored in the RGB components of a 3D volume texture, and the original isovalue is stored in the Alpha component. This allows using a single texture for both data and shading.

### [Röttger et al. Rendering](938-r-ttger-et-al-rendering.md)
Röttger et al. Rendering is a technique that mitigates the high computational cost of volume integration by approximating the rendering integral and moving the computation to a pre-process. It computes a 3D lookup table of colors and opacities parameterized by front scalar value, back scalar value, and distance, which is stored in a 3D texture for hardware-accelerated lookups during rendering. This method builds on AR mapping to 3D texture by using the 3D texture not just for static data storage, but to evaluate the volume composition operator directly via texture interpolation. It is distinguished from standard ray casting by replacing per-voxel integration with texture lookups, and from pure pre-classification by supporting arbitrary transfer functions including isosurface effects through the parameterization of the lookup table.

### Uniform Grid
A variant implementation detail where the underlying data structure is a uniform grid.
- **Step:** `set_values` accepts a $(X, Y, Z)$ float32 tensor to update scalar values.
- **Parameters:**
  - `aabb_min`: The minimum corner of the bounding box. Default is $[-1, -1, -1]$.
  - `aabb_max`: The maximum corner of the bounding box. Default is $[1, 1, 1]$.
  - `default_value`: Initial scalar value for all points. Default is float max.
- **Layout:**
  - `get_points` returns a $(X, Y, Z, 3)$ float32 tensor containing 3D coordinates of all grid points.
  - `get_values` returns a $(X, Y, Z)$ float32 tensor containing scalar values.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Avoids non-linear formulae; allows texture coordinates to be defined once; enables use of display lists for performance improvement [S1, unofficial].
- **Limitations:** The estimation method is likely worse than Wilson's depth integration [S1, unofficial].
- **When to use:** When rendering parallel polygons and performance is critical, and high precision is secondary to effective visualization [S1, unofficial].

## Background
The method originates from Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration [S1, unofficial]. It was developed to improve the performance of interactive volume visualization by leveraging OpenGL display lists and linear texture mapping.

## Sources
[S1] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
[S2] GuangyanCai/isoext, https://github.com/GuangyanCai/isoext, fetched 2026-10-27 (unofficial)
