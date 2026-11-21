# Scanline A-buffer rendering

Also called: Ebert and Parent method

## Overview
A rendering algorithm for surface-defined objects in hybrid scenes, creating an A-buffer for a scanline containing a list of all fragments that partially or fully cover each pixel [S2, unofficial]. It sets itself apart by explicitly managing fragment lists per pixel to support hybrid volume rendering, unlike standard scanline renderers that discard depth information early [S1].

## How it works
**Inputs and Outputs:**
- Input: A scanline of pixels, geometry data for surface-defined objects.
- Output: An A-buffer data structure for the scanline, containing fragment lists for each pixel.

**Data Structures:**
- **A-buffer**: An array indexed by pixel column. Each entry contains a linked list of fragments.
- **Fragment**: A data structure representing a surface patch covering part of a pixel. Fields include:
  - `z_min`, `z_max`: Depth range of the fragment.
  - `color`: Surface color/opacity.
  - `next`: Pointer to the next fragment in the list.
  - `pixel_coverage`: Bitmap or count of pixels covered by this fragment.

**Steps:**
1. For each pixel in the scanline, initialize an empty fragment list in the A-buffer.
2. For each surface-defined object intersecting the scanline:
   a. Determine all fragments that partially or fully cover the pixel.
   b. For each fragment, compute its depth range (`z_min`, `z_max`) and coverage.
   c. Insert the fragment into the pixel's list in the A-buffer, sorted by depth (back-to-front).
3. After processing all objects, the A-buffer contains all fragments for the scanline, ready for volume rendering compositing.

**Formulas:**
- Fragment coverage: $C = \frac{\text{Area of intersection}}{\text{Pixel Area}}$
- Depth sorting: Fragments are sorted such that $z_{i} \le z_{i+1}$ for back-to-front rendering.

**Parameters:**
- Not given in the sources.

**Edge cases and pitfalls:**
- Precision: Depth values must be precise to avoid z-fighting in fragment sorting.
- Boundaries: Fragments covering partial pixels must be handled correctly to avoid artifacts.
- Degenerate input: Objects with no intersection with the scanline should be skipped.

## Subtopics

### Hybrid Scanline A-Buffer Volume Rendering
Combines this method with volume rendering for procedural gases, using Z-sorted A-buffer fragments [Parent].

## Operating characteristics
- **Time complexity**: $O(N \cdot M)$, where $N$ is the number of pixels in the scanline and $M$ is the average number of fragments per pixel.
- **Memory complexity**: $O(N \cdot M)$ for storing fragment lists.
- **Measured performance**: Not given in the sources.
- **Scaling**: Performance degrades with high fragment density per pixel.
- **Strengths**: Explicit management of fragments allows for hybrid rendering with volume data.
- **Limitations**: High memory usage for scenes with many overlapping surfaces.
- **When to use**: Hybrid scenes with surface-defined objects and volumetric data.
- **When not to use**: Scenes with only volumetric data or simple surface rendering.

## Background
Originated by Ebert and Parent in the pre-shader era, combining scanline A-buffer rendering with volume rendering to handle hybrid scenes [S1]. It was used for visualizing surface-defined objects in conjunction with procedural gases like clouds [S1][S2, unofficial].

## Sources
[S1] A survey of modeling, rendering and animation of clouds in computer graphics, https://doi.org/10.1007/s00371-020-01953-y, fetched 2026-10-26
[S2] Volume Rendering Methods for Computational Fluid Dynamics Visualization, https://www.researchgate.net/profile/David-Ebert-4/publication/220943451_Volume_Rendering_Methods_for_Computational_Fluid_Dynamics_Visualization/links/00b7d51a73e2892e23000000/Volume-Rendering-Methods-for-Computational-Fluid-Dynamics-Visualization.pdf, fetched 2026-10-27 (unofficial)
