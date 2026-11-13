# Real-time shell space rendering

Also called: Real-time shell space rendering of volumetric geometry

## Overview
A technique for real-time rendering of arbitrary volumetric geometry into a polygonal mesh's shell space, which is a layer of variable thickness on top or below the polygonal mesh [S1, abstract only]. It differs from its parent [Shell Method for Volume Rendering](10496-shell-method-for-volume-rendering.md) by explicitly defining the shell volume via a pre-computed tetrahedra mesh and using a 3D distance map to optimize ray traversal, overcoming the high storage requirements of [Wang et al. 2004] and the mapping distortions of [Policarpo and Oliveira 2006] [S1, abstract only].

## How it works
**Inputs:**
1. A polygonal base mesh defining the surface geometry.
2. A target volumetric geometry to be rendered within the shell space.

**Outputs:**
1. A rendered image of the volumetric geometry within the shell space, including local lighting effects.

**Data Structures:**
1. **Pre-computed Tetrahedra Mesh:** A mesh where tetrahedra subdivide prisms extruded from the polygonal base mesh. This structure defines the shell volume boundaries.
2. **Geometry Volume Texture:** A 3D texture representing the arbitrary volumetric geometry within the shell space.
3. **3D Distance Map:** A 3D texture storing distance values to skip empty space in the shell volume during ray tracing.

**Steps:**
1. **Pre-computation:**
   - Generate the geometry volume texture and the corresponding 3D distance map from the polygonal mesh and the target volumetric data [S1, abstract only].
   - Construct the pre-computed tetrahedra mesh by extruding prisms from the polygonal base mesh and subdividing them into tetrahedra [S1, abstract only].

2. **Vertex Shader Processing:**
   - For each vertex of the tetrahedra mesh, compute the distances to the tetrahedron face planes [S1, abstract only].
   - These distances are used to generate per-pixel tetrahedron thickness using non-linear interpolation across the primitive [S1, abstract only].

3. **Pixel Shader Processing (Ray Tracing):**
   - For each pixel, compute the view ray intersection with the shell geometry defined by the tetrahedra mesh [S1, abstract only].
   - Use the 3D distance map to skip empty space in the shell volume, minimizing the number of ray tracing steps required to traverse the volume [S1, abstract only].
   - Sample the geometry volume texture along the ray to determine color and opacity.
   - Apply local lighting calculations to the sampled volume data [S1, abstract only].

**Parameters:**
- Shell thickness: Variable, determined by the extrusion and tetrahedra subdivision process. Not given in the sources.
- Ray step size: Adapted by the 3D distance map skips. Not given in the sources.

**Edge Cases and Pitfalls:**
- Shallow view angles: Rendering artifacts common to many related techniques are reduced by this method [S1, abstract only].
- Empty space: The 3D distance map is critical for performance; without it, ray tracing steps would be excessive [S1, abstract only].

## Subtopics

### [Shell Textures](14361-shell-textures.md)
Applies specific texture sampling strategies to the shell space to simulate volumetric microgeometry (such as fur) rather than just solid volume.

## Operating characteristics
- **Time Complexity:** Optimized by the 3D distance map which skips empty space, minimizing ray tracing steps [S1, abstract only].
- **Memory Complexity:** Uses a pre-computed tetrahedra mesh and 3D textures (volume texture and distance map) [S1, abstract only].
- **Strengths:** Handles arbitrary volumetric geometry and height-field shell geometry; reduces artifacts at shallow view angles; includes local lighting [S1, abstract only].
- **Limitations:** Requires pre-computation of the tetrahedra mesh and 3D textures [S1, abstract only].
- **Comparison:** Overcomes high storage requirements and per-vertex computations of [Wang et al. 2004] and low shell geometry depth complexity and mapping distortions of [Policarpo and Oliveira 2006] [S1, abstract only].

## Background
This technique was introduced to address shortcomings in previous shell space rendering approaches [S1, abstract only]. It builds on the general shell idea but implements it via a tetrahedra-based volume definition and distance-map-accelerated ray tracing for real-time performance [S1, abstract only].

## Sources
[S1] Real-time shell space rendering of volumetric geometry, https://doi.org/10.1145/1174429.1174477, fetched 2026-10-26 (abstract only)
