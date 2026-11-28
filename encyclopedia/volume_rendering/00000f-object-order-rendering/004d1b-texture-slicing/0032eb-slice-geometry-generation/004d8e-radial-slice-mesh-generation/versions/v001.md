# Radial Slice Mesh Generation

Also called: Slice Scanning Geometry

## Overview
Radial Slice Mesh Generation creates a mesh of quadrilaterals representing 2D screens rotating through a volumetric volume for slice-scanning display simulation [S1, unofficial]. It differs from its siblings by generating geometry for a continuous rotational sweep rather than static planar cuts or arbitrary directional slices. It builds on [Texture slicing](25-texture-slicing.md) [G].

## How it works
**Inputs:**
- `sim_geometry.slice_count`: Number of rotational slices (default 360) [S1, unofficial].
- `sim_geometry.screen_width`: Panel width used to calculate rotation radius [S1, unofficial].
- `VOXELS_X`, `VOXELS_Z`: Volume dimensions in X and Z axes [S1, unofficial].

**Outputs:**
- A vertex array `vertices` of shape `[slice_count][screen_count][12]` [S1, unofficial].

**Data Structures:**
- `vertices`: A 3D array where the first two dimensions index the slice and screen position, and the third dimension holds 12 floats (likely 4 vertices per quad, 3 coordinates each) [S1, unofficial].

**Steps:**
1. Allocate the vertex array `vertices` with dimensions `[sim_geometry.slice_count][sim_geometry.screen_count][12]` [S1, unofficial].
2. Generate a slice scatter order using `slicemap_ebr(scatter, count_of(scatter))` [S1, unofficial].
3. Calculate the aspect ratio $z = \frac{\text{VOXELS\_Z}}{\text{VOXELS\_X}}$ and the radius $r = \text{sim\_geometry.screen\_width} \times 0.5$ [S1, unofficial].
4. For each slice $s$ from $0$ to $\text{sim\_geometry.slice\_count} - 1$:
   a. Compute the angle $\theta = \frac{s \times \pi \times 2}{\text{sim\_geometry.slice\_count}}$ [S1, unofficial].
   b. Calculate the slope vector:
      $$ \text{slope}.x = \cos(\theta) $$
      $$ \text{slope}.y = \sin(\theta) $$
   c. [G] The source code cuts off at a comment `// throttle the inner columns`, indicating missing implementation details for inner column scaling or culling during radial projection [S1, unofficial].

**Parameters:**
- `sim_geometry.slice_count`: Default 360 [S1, unofficial].
- `sim_geometry.screen_width`: Panel width [S1, unofficial].

**Pitfalls:**
- The implementation details for handling inner columns during radial projection are incomplete in the source, specifically regarding scaling or culling [S1, unofficial].

## Subtopics

### [Polar Slice Generation for Rotating Displays](22521-polar-slice-generation-for-rotating-displays.md)
Generates discrete volumetric data in 3D polar coordinates $(r, \theta, z)$ for a rotating volumetric display by rendering orthographic camera views of a boolean-intersected slice through a 3D object at 24 evenly spaced rotation angles.

## Operating characteristics
Not given in the sources.

## Background
Origin: This method is described in the "Multivox" project, an unofficial implementation of a volumetric display [S1, unofficial].

## Sources
[S1] Multivox: Volumetric Display, https://github.com/AncientJames/multivox, fetched 2026-10-26 (unofficial)
