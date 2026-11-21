# Two-Pass Volume Rendering with Shadows

Also called: GPU Volumetric Shadows

## Overview
A two-pass algorithm that renders volumetric shadows on the GPU by modifying proxy geometry slicing to render from both the eye and the light's point of view [S1, unofficial]. It improves depth perception and amplifies surface structure by simulating light attenuation without the cost of secondary raymarching [S1, unofficial]. It sets apart from standard slice-based rendering by decoupling the lighting calculation into a separate pass that computes light transmissivity, allowing for interactive hard and translucent shadows [S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: 3D volume texture, slice geometry (proxy triangles/quads), view matrix, light position/direction.
- Output: Final color buffer with shadows.

**Data Structures:**
- **3D Texture:** Stores scalar volume data.
- **Light Buffer:** A texture (framebuffer object) storing light color/transmissivity values computed in the second pass.
- **Slice Geometry:** Planar polygons intersecting the volume bounding box, with 3D texture coordinates mapped to the volume [S1, unofficial][S2, unofficial].

**Steps:**
1. **Slice Axis Calculation:** Set the slice axis half-way between the view direction and the light direction [S1, unofficial]. This orientation allows each slice to be rendered effectively from both the eye and light points of view.
2. **First Pass (Eye View):**
   - Render slices from the eye's point of view.
   - Render order: Front-to-back or back-to-front [S1, unofficial].
   - In the fragment shader:
     - Sample the 3D texture to get material color/opacity.
     - Look up the light color from the **Light Buffer** (bound as a texture).
     - Multiply material color by light color: $C_{final} = C_{material} \times C_{light}$.
     - Composite using alpha blending (e.g., `GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA`) [S1, unofficial][S2, unofficial].
3. **Second Pass (Light View):**
   - Render slices from the light's point of view.
   - Render order: Front-to-back [S1, unofficial].
   - In the fragment shader:
     - Sample the 3D texture to get opacity.
     - Only blend alpha values to compute light transmissivity (shadowing).
     - Store the result in the **Light Buffer**.
     - This pass simulates light attenuation from the light's perspective [S1, unofficial].

**Parameters:**
- Slice axis: Bisector of view and light directions [S1, unofficial].
- Light render order: Front-to-back [S1, unofficial].
- Eye render order: Front-to-back or back-to-front [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Large fragment/pixel operations and texture access can lead to precision issues in the light buffer if not handled with sufficient bit depth [S1, unofficial].
- **Memory Usage:** Requires additional framebuffer objects for the light buffer, increasing memory usage [S1, unofficial].
- **Degenerate Input:** If the view and light directions are parallel, the slice axis calculation may become ambiguous; however, the bisector remains well-defined unless vectors are identical or opposite.

## Subtopics

### Slice-based shadow maps
A technique for simulating shadows in texture-based volume rendering that avoids expensive lighting computations by computing shadow maps per volume slice and rendering them with a specific blending function [S2, unofficial]. It computes a shadow map for each slice, renders these into the framebuffer, and applies a specific blending function to modulate shading based on the shadow maps [S2, unofficial]. This approach is distinct from the two-pass global light buffer method by localizing shadow computation to individual slices.

## Operating characteristics
**Time and Memory Complexity:**
- Time: $O(N \cdot S)$, where $N$ is the number of slices and $S$ is the number of fragments per slice. Two passes double the rasterization cost compared to single-pass volume rendering.
- Memory: Requires storage for the 3D volume texture plus an additional framebuffer/texture for the light buffer [S1, unofficial].

**Measured Performance:**
- Performance is limited by the huge amount of fragment/pixel operations, texture access, lighting calculation, and blending [S1, unofficial].
- It maintains interactivity compared to full raymarching but is more expensive than standard slice-based rendering due to the second pass [S2, unofficial].

**Strengths and Limitations:**
- Strengths: Interactive performance, improved depth perception, simulation of hard and translucent shadows [S2, unofficial].
- Limitations: High fragment/pixel operation count, large memory usage, potential for artifacts if light buffer precision is insufficient [S1, unofficial].

**When to Use:**
- Use when interactive volumetric shadows are needed and full raymarching is too slow.
- Avoid when memory is extremely constrained or when soft shadows require complex scattering models not captured by simple alpha blending [S1, unofficial].

## Background
Originating from research on GPU-based volume rendering, this method modifies proxy geometry slicing to handle illumination [S1, unofficial]. It addresses the complexity of general scattering effects, which are too complex for interactive rendering, by focusing on incoming light from the direction of the light source [S1, unofficial]. The approach is part of a broader effort to improve perception in volume visualization through illumination techniques [S2, unofficial]. Related acceleration techniques for GPU-based volume rendering were discussed by Krüger and Westermann in 2003 [S1, unofficial].

## Sources
[S1] GPU volume rendering, http://www.slideserve.com/hilliard/volume-rendering-using-graphics-hardware, fetched 2026-10-26 (unofficial)
[S2] Enhanced perception in volume visualization, https://doi.org/10.5821/dissertation-2117-94929, fetched 2026-10-26 (unofficial)
