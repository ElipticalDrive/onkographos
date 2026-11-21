# Two-Pass Volume Rendering with Shadows

Also called: GPU Volumetric Shadows

## Overview
A two-pass algorithm that renders volumetric shadows on the GPU by modifying proxy geometry slicing to render from both the eye and the light's point of view [S1, unofficial]. It improves depth perception and amplifies surface structure by simulating light attenuation without the cost of secondary raymarching [S1, unofficial]. It differs from standard slice-based rendering by computing light transmissivity in a dedicated pass, enabling interactive hard and translucent shadows [S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: 3D volume texture, slice geometry (proxy triangles/quads), view matrix, light position/direction [G].
- Output: Final color buffer with shadows [G].

**Data Structures:**
- **3D Texture:** Stores scalar volume data [G].
- **Light Buffer:** A texture (framebuffer object) storing light color/transmissivity values computed in the second pass [S1, unofficial].
- **Slice Geometry:** Planar polygons intersecting the volume bounding box, with 3D texture coordinates mapped to the volume [G].

**Steps:**
1. **Slice Axis Calculation:** Set the slice axis half-way between the view direction and the light direction [S1, unofficial]. This orientation allows each slice to be rendered effectively from both the eye and light points of view [G].
2. **First Pass (Eye View):**
   - Render slices from the eye's point of view [G].
   - Render order: Front-to-back or back-to-front [S1, unofficial].
   - In the fragment shader:
     - Sample the 3D texture to get material color/opacity [G].
     - Look up the light color from the **Light Buffer** (bound as a texture) [S1, unofficial].
     - Multiply material color by light color: $C_{final} = C_{material} \times C_{light}$ [S1, unofficial].
     - Composite using standard alpha blending [G].
3. **Second Pass (Light View):**
   - Render slices from the light's point of view [G].
   - Render order: Front-to-back [S1, unofficial].
   - In the fragment shader:
     - Sample the 3D texture to get opacity [G].
     - Only blend alpha values to compute light transmissivity (shadowing) [S1, unofficial].
     - Store the result in the **Light Buffer** [S1, unofficial].
     - This pass simulates light attenuation from the light's perspective [S1, unofficial].

**Parameters:**
- Slice axis: Bisector of view and light directions [S1, unofficial].
- Light render order: Front-to-back [S1, unofficial].
- Eye render order: Front-to-back or back-to-front [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Large fragment/pixel operations and texture access can lead to precision issues in the light buffer if not handled with sufficient bit depth [S1, unofficial].
- **Memory Usage:** Requires additional framebuffer objects for the light buffer, increasing memory usage [S1, unofficial].
- **Degenerate Input:** If the view and light directions are parallel, the slice axis calculation may become ambiguous; however, the bisector remains well-defined unless vectors are identical or opposite [G].

## Subtopics

### Slice-based shadow maps
A technique for simulating shadows in texture-based volume rendering that avoids expensive lighting computations by computing shadow maps per volume slice and rendering them with a specific blending function [S2, unofficial]. It computes a shadow map for each slice, renders these into the framebuffer, and applies a specific blending function to modulate shading based on the shadow maps [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- Time: $O(N \cdot S)$, where $N$ is the number of slices and $S$ is the number of fragments per slice [G]. Two passes double the rasterization cost compared to single-pass volume rendering [G].
- Memory: Requires storage for the 3D volume texture plus an additional framebuffer/texture for the light buffer [S1, unofficial].

**Measured Performance:**
- Performance is limited by the huge amount of fragment/pixel operations, texture access, lighting calculation, and blending [S1, unofficial].
- It maintains interactivity compared to full raymarching [S2, unofficial]. The second pass increases computational cost over standard slice-based rendering [G].

**Strengths and Limitations:**
- Strengths: Interactive performance, improved depth perception, simulation of hard and translucent shadows [S2, unofficial].
- Limitations: High fragment/pixel operation count, large memory usage, potential for artifacts if light buffer precision is insufficient [S1, unofficial].

**When to Use:**
- Use when interactive volumetric shadows are needed and full raymarching is too slow [G].

## Background
Originating from research on GPU-based volume rendering, this method modifies proxy geometry slicing to handle illumination [S1, unofficial]. It addresses the complexity of general scattering effects, which are too complex for interactive rendering, by focusing on incoming light from the direction of the light source [S1, unofficial]. The approach is part of a broader effort to improve perception in volume visualization through illumination techniques [S2, unofficial]. Related acceleration techniques for GPU-based volume rendering were discussed by Krüger and Westermann in 2003 [S1, unofficial].

## Sources
[S1] GPU volume rendering, http://www.slideserve.com/hilliard/volume-rendering-using-graphics-hardware, fetched 2026-10-26 (unofficial)
[S2] Enhanced perception in volume visualization, https://doi.org/10.5821/dissertation-2117-94929, fetched 2026-10-26 (unofficial)
