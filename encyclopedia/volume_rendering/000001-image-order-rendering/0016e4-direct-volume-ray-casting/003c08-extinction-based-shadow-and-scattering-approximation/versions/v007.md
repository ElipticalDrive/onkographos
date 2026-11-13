# Extinction-based Shadow and Scattering Approximation

Also called: Cone approximation via SAT, Extinction SAT queries

## Overview
This method approximates directional shadows and light scattering in volume rendering by computing the extinction of light intensity for voxels using a 3D Summed Area Table (SAT) [S1, unofficial]. It avoids the computational cost of propagating illumination from outside the volume or performing full shadow raycasts per sample [S1, unofficial]. Unlike half-angle slicing methods, it handles any light type and multiple sources; unlike propagation methods, it computes voxel extinction directly [S1, unofficial].

## How it works
**Inputs**: 3D volume dataset (scalar values mapped to extinction coefficients $\sigma$), light source positions and types, transfer function (TF), camera parameters.
**Outputs**: Final image with approximate shadows and ambient occlusion (AO)/color bleeding.

**Data Structures**:
1.  **Extinction SAT**: A 3D texture of size $D_x \times D_y \times D_z$ storing cumulative sums of extinction coefficients. Let $E(x,y,z)$ be the extinction at voxel $(x,y,z)$. The SAT value $S(x,y,z)$ is:
    $$ S(x,y,z) = \sum_{i=0}^{x} \sum_{j=0}^{y} \sum_{k=0}^{z} E(i,j,k) $$
    (Assuming 0-based indexing and inclusive bounds).
2.  **Light Cache Texture**: Stores precomputed shadow and AO terms to avoid duplicate queries.

**Steps**:
1.  **SAT Construction**: Compute the 3D SAT of the extinction field. This allows $O(1)$ retrieval of the sum of extinction over any axis-aligned cuboid [S1, unofficial].
2.  **Light Cache Precomputation (AO/Color Bleeding)**:
    *   For each voxel, approximate the neighborhood using cuboid shells.
    *   Sum extinction coefficients for these shells using SAT lookups.
    *   Store results in the light cache texture [S1, unofficial].
3.  **Shadow Cone Approximation (Per Frame or on TF/Light Change)**:
    *   **Primary Axis Selection**: For a given light source, define the primary axis as the coordinate axis with the smallest angle to the vector from the voxel to the light source [S1, unofficial].
    *   **Sampling Points**: Define sampling points on the primary axis using a user-defined sampling frequency and growth rate. The growth rate controls the change of frequency over distance, allowing coarser sampling further away [S1, unofficial].
    *   **Cuboid Derivation**: Derive axis-aligned cuboids that cover the shadow cone. The cone is defined by a cone angle $\Psi$. The cuboids are derived from the primary axis sampling and the projection of the query cone onto the primary-secondary axis planes [S1, unofficial].
    *   **SAT Lookup**: Perform SAT lookups to sum extinction coefficients for the derived cuboids. The sum represents the attenuation along the cone [S1, unofficial].
    *   **Weighting**: Apply a weighting function $h$ to the cuboids to strengthen or weaken the shadow effect [S1, unofficial].
    *   **Storage**: Store the computed shadow terms in the light cache texture [S1, unofficial].
4.  **Raycasting Pass**:
    *   For each sample on the primary ray, perform one additional texture lookup to retrieve the precomputed illumination terms (shadow and AO) from the light cache [S1, unofficial].
    *   Apply these terms to the shading model [S1, unofficial].

**Parameters**:
*   **Cone Angle ($\Psi$)**: Defines the angular spread of the shadow cone. Typical values: 1.0 to 5.0 degrees [S1, unofficial]. Larger angles produce smoother shadows [S1, unofficial].
*   **Sampling Frequency**: Number of cuboids for approximation. Typically a few dozen lookups per cone and voxel are sufficient [S1, unofficial].
*   **Growth Rate**: Controls the growth of cuboids (change of frequency over distance) [S1, unofficial].
*   **Weighting Function ($h$)**: Weights cuboids differently [S1, unofficial].

**Pitfalls**:
*   Using very few cone samples (e.g., 12) results in clearly visible cuboid artifacts [S1, unofficial].
*   The SAT size can be a fraction of the volume dataset size without exposing conspicuous artifacts, but too small a size results in blurred shadows [S1, unofficial].

## Subtopics

### Approximate Shadowing
An algorithm to compute approximate shadows for volumetric rendering by baking axis-aligned light extinction into textures and interpolating, avoiding full shadow raycasts per voxel or raymarch step [S2]. It extends the "6-point" or "6-way" lighting technique used in VFX flipbooks to interactive simulation [S2]. Axis-aligned extinction is computed via cumulative products of extinction along each axis, written to different textures concurrently [S2]. These are collapsed down to two interpolated directional extinction values to reduce bandwidth cost during raymarching [S2]. Precomputation cost is <1ms across tested scenarios [S2].

## Operating characteristics
**Time Complexity**:
*   SAT Construction: $O(D_x D_y D_z)$. For a $256^3$ volume, construction takes 0.311s; for $64^3$, 0.029s [S1, unofficial].
*   AO/Color Bleeding Computation: Depends on the number of shells. For a $256^3$ texture with 3 shells, it takes 0.0307s, compared to 3.0818s for explicit sampling of radius 7 [S1, unofficial].
*   Shadow Computation: For a $64^3$ texture with 50 samples, it takes 0.0079s/frame [S1, unofficial]. For a $128^3$ texture with 50 samples, 0.0385s/frame [S1, unofficial].
*   Raycasting Overhead: One additional texture lookup per sample [S1, unofficial].

**Memory Complexity**:
*   Requires storage for the Extinction SAT and the Light Cache Texture [S1, unofficial].

**Performance**:
*   Enables interactive rendering with dynamic transfer functions and light sources [S1, unofficial].
*   The method is fast and flexible, allowing variation of cone angle, number of cuboids, and weighting [S1, unofficial].
*   Compared to classical shadow rays (e.g., 250 samples per unit), this method achieves similar quality with significantly fewer samples (a few dozen) [S1, unofficial].

**Strengths**:
*   Handles any type and multiple light sources, including those inside or on the border of the dataset [S1, unofficial].
*   Avoids costly preprocessing [S1, unofficial].
*   Low performance impact relative to shading effects achieved [S1, unofficial].

**Limitations**:
*   Requires recomputation of SAT and AO terms when the TF changes or light sources move relative to the dataset [S1, unofficial].
*   Cuboid artifacts may be visible if too few samples are used [S1, unofficial].

## Background
Originated in the paper "Extinction-based Shading and Illumination in GPU Volume Ray-Casting" [S1, unofficial]. It was developed to provide realistic directional shadows and ambient occlusion in GPU-based volume ray-casting without the high cost of traditional shadow raycasting or illumination propagation. It is used in GPU ray-casters for interactive volume rendering [S1, unofficial].

## Sources
[S1] Extinction-based Shading and Illumination in GPU Volume Ray-Casting, https://www.ifi.uzh.ch/dam/jcr:ffffffff-9e7f-2af5-ffff-ffffcf9259bb/FastShadows.pdf, fetched 2026-10-30 (unofficial)
[S2] Ignis: Eulerian Fluid Simulation and Rendering at VR Frame Rates, https://doi.org/10.1145/3610542.3626120, fetched 2026-10-30
