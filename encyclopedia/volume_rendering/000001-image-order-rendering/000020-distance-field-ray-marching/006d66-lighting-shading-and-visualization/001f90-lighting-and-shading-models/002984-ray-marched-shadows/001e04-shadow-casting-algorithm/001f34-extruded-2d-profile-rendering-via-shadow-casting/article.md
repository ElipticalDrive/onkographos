# Extruded 2D Profile Rendering via Shadow Casting

Also called: sculpted shafts of light technique

## Overview
A strategy to render geometric objects defined by the extrusion of 2D profiles using the shadow casting algorithm to simulate volumetric light shafts [S1, unofficial]. It constructs a mesh of shadow volumes from a shadow map to produce radial blur around light sources, light shafts, and volumetric shadows without ray marching [S2, unofficial]. It is distinct from [Texture Projection](2002-texture-projection.md) by explicitly generating shadow volume meshes for volumetric occlusion effects rather than projecting textures onto surfaces.

## How it works
**Inputs:**
*   A 2D profile geometry defining the cross-section of the light beam or object [S1, unofficial].
*   Volumetric data representing the medium (e.g., dust-filled room, radiation field) [S1, unofficial].
*   Light source parameters [S2, unofficial].

**Outputs:**
*   A rendered image showing sculpted shafts of light passing through the volumetric data [S1, unofficial].

**Data Structures:**
*   **Array of Initial Light Strengths:** A data structure storing the intensity values for the volumetric grid [S1, unofficial].
*   **Volumetric Data:** A grid of voxels representing the medium [S1, unofficial].
*   **Shadow Volume Mesh:** A mesh constructed from the shadow map to define occluded regions [S2, unofficial].

**Steps:**
1.  **Initialize Volumetric Opacity:** Assign all voxels in the volumetric data a slightly non-zero opacity [S1, unofficial]. This ensures the medium is visible even in areas not directly illuminated by the primary beam.
2.  **Load Profile Geometry:** Load the 2D profile geometry into the array of initial light strengths [S1, unofficial]. This defines the shape and intensity distribution of the light source or beam.
3.  **Construct Shadow Volumes:** Construct a mesh of shadow volumes from the shadow map [S2, unofficial]. This step identifies regions occluded from the light source.
4.  **Render Shadow Effects:** Render the scene using the shadow volume mesh to produce radial blur around light sources, light shafts, and volumetric shadows [S2, unofficial]. This is achieved without ray marching [S2, unofficial].
5.  **Optional: Non-Attenuating Beam:** If the beam is assumed to pass through the volumetric data without attenuation, render it without a separate shadow pass and without the accompanying 3D representation [S1, unofficial].

**Parameters:**
*   **Voxel Opacity:** "Slightly non-zero" [S1, unofficial]. The exact value is not specified.
*   **Attenuation:** Can be enabled or disabled. If disabled, the beam passes through without attenuation [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Overhead:** The algorithm suffers from quite large overhead despite providing speed up and quality images [S2, unofficial].
*   **Aliasing:** While not explicitly detailed for this specific method, related techniques using sampling hulls aim to reduce aliasing at intersections between virtual planes and bounding boxes [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Provides a speed up and quality images compared to ray marching [S2, unofficial]. However, it suffers from quite large overhead [S2, unofficial].
*   **Strengths:** Avoids the computational cost of ray marching; produces radial blur and volumetric shadows effectively [S2, unofficial].
*   **Limitations:** Large overhead; may not handle multiple scattering effects well (as noted in related methods [S2, unofficial]).
*   **When to use:** When rendering light shafts and volumetric shadows for extruded 2D profiles is required, and ray marching is too slow, but overhead is acceptable.
*   **When not to use:** When memory or computational overhead is a critical constraint.

## Background
The technique is described in the context of rendering mixtures of geometric and volumetric data [S1, unofficial]. It is mentioned as a strategy for treating geometric objects defined by extrusion of 2D profiles within a hybrid ray tracer framework [S1, unofficial]. The concept of constructing shadow volumes from shadow maps for light shafts is attributed to Billeter [S2, unofficial].

## Sources
[S1] DTIC ADA208085: Rendering Mixtures of Geometric and Volumetric Data, https://archive.org/details/DTIC_ADA208085, fetched 2026-10-26 (unofficial)
[S2] A Review On Light Shafts Rendering For Indoor Scenes, https://doi.org/10.5281/zenodo.1125510, fetched 2026-10-26 (unofficial)
