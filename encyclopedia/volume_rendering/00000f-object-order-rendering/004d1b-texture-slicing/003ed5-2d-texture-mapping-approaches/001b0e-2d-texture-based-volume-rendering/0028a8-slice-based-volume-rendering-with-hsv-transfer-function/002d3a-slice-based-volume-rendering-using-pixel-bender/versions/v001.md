# Slice-based Volume Rendering using Pixel Bender

Also called: Pixel Bender Slice Rendering

## Overview
A real-time volume rendering technique that uses view-aligned slices and a static 3D texture (set of cross sections), implemented via Pixel Bender shaders [S1, unofficial]. It differs from surface rendering methods (e.g., marching cubes) by rendering volume data directly without building an isosurface, allowing for translucent cross-sections [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A static 3D texture representing volume data as a set of 2D cross-sections (slices). The texture is typically packed into a 2D texture array or sprite sheet.
- Output: A 2D image where each pixel corresponds to a sample from the volume along the view direction.

**Data Structures:**
- **3D Texture:** Stored as a 2D texture with dimensions $W \times H$, where $H$ represents the number of slices $N_z$ and $W$ represents the width $N_x$ of each slice. The height of each slice is $N_y$.
  - Example dimensions:
    - High quality: $4096 \times 128$ texture for a $128 \times 128 \times 32$ volume [S1, unofficial].
    - Low quality: $2048 \times 64$ texture [S1, unofficial].
    - Ultra quality: $4096 \times 128$ texture with 50 visible slices [S1, unofficial].
- **Slice Indexing:** The slice index $z$ is derived from the texture coordinate $v$ (vertical) by mapping $v \in [0, 1]$ to $z \in [0, N_z - 1]$.

**Numbered Steps:**
1. **View-Aligned Slice Generation:** Generate view-aligned slices perpendicular to the camera view direction.
2. **Pixel-to-World Transformation:** For each pixel on every slice, calculate the point $p$ on the view-aligned slice.
3. **Change of Basis:** Determine the position of $p$ in the texture's 3D space using a change of basis transformation. Since eye space equals world space in this implementation, the transformation is:
   $$ p_{texture} = M_{object\_inverse} \cdot p_{eye} $$
   where $M_{object\_inverse}$ is the inverse of the object's delta transformation matrix [S1, unofficial].
4. **Interpolation:** The transformed point $p_{texture}$ usually lies between two adjacent texture slices. Sample both adjacent texture slices using constant $x$ and $y$ coordinates and interpolate the color values [S1, unofficial].
5. **Culling:** Cull half of the back slices to improve performance, as they contribute minimally to the final image [S1, unofficial].
6. **Rendering:** Perform these calculations for every pixel on every slice using Pixel Bender shaders [S1, unofficial].

**Parameters:**
- Number of cross sections: 32 (primary example) [S1, unofficial].
- Visible slices: 16 (high/low quality), 50 (ultra quality) [S1, unofficial].

**Pitfalls:**
- Interpolation is not aligned to the view direction, which is technically incorrect but accepted as a performance trade-off [S1, unofficial].
- Pixel Bender in FP10 is slow on conditionals, limiting alpha-testing capabilities [S1, unofficial].

## Operating characteristics
- **Performance:** Real-time, but dependent on the number of visible slices and texture resolution. Ultra quality (50 slices) is slow [S1, unofficial].
- **Memory:** Depends on texture size. High quality uses $4096 \times 128$ texture [S1, unofficial].
- **Strengths:** Fast compared to surface rendering methods; allows translucent cross-sections [S1, unofficial].
- **Limitations:** Interpolation inaccuracy; slow on conditionals in Pixel Bender FP10 [S1, unofficial].

## Background
Originated by Der Schmale in 2009 as an alternative to sparse voxel octree ray casting using Alchemy [S1, unofficial]. It builds on earlier experiments with object-aligned slices [S1, unofficial].

## Sources
[S1] Slice-based volume rendering using Pixel Bender | Der Schmale, https://www.derschmale.com/2009/07/20/slice-based-volume-rendering-using-pixel-bender/, fetched 2026-10-27 (unofficial)
