# Extinction-based Shadow and Scattering Approximation

Also called: Cone approximation via SAT, Extinction SAT queries

## Overview
A technique to approximate directional shadows and light scattering by computing the extinction of light intensity for voxels using a 3D Summed Area Table (SAT), avoiding the propagation of illumination from outside the volume [S1, unofficial]. It differs from half-angle slicing by handling any light type and multiple sources, and from Ropinski et al.'s method by computing voxel extinction rather than propagating external illumination [S1, unofficial].

## How it works
**Inputs:** Scalar volume $V$, transfer function $TF$, light source positions $\mathbf{L}_j$, cone angle $\Psi$, sampling frequency $f$, growth rate $g$, weighting function $h$ [S1, unofficial].
**Outputs:** Light cache texture containing shadow and ambient occlusion (AO)/color bleeding terms [S1, unofficial].

**Data Structures:**
1. **Extinction SAT:** 3D texture of size $N_x \times N_y \times N_z$ storing the cumulative sum of extinction coefficients $\sigma_t$ along the three axes [S1, unofficial].
2. **Light Cache Texture:** 3D texture storing precomputed shadow and AO terms per voxel [S1, unofficial].

**Steps:**
1. **Construct Extinction SAT:** Compute the 3D SAT of the extinction coefficient field $\sigma_t(\mathbf{x})$ [S1, unofficial].
2. **Determine Primary Axis:** For each light source, identify the coordinate axis with the smallest angle to the light direction vector [S1, unofficial].
3. **Sample Primary Axis:** Define sampling points along the primary axis using the user-defined sampling frequency $f$ and growth rate $g$ [S1, unofficial].
4. **Derive Cuboid Queries:** For each sample point, project the query cone (defined by angle $\Psi$) onto the primary-secondary axis planes [S1, unofficial].
5. **Compute Extinction via SAT:** Perform SAT lookups to sum extinction coefficients for the derived cuboids [S1, unofficial].
6. **Store in Light Cache:** Store the computed shadow and AO/color bleeding terms in the light cache texture to avoid duplicate queries [S1, unofficial].
7. **Raycasting Application:** During the raycasting pass, apply illumination terms using one additional texture lookup per sample from the light cache [S1, unofficial].

**Formulas:**
The primary axis is chosen as the coordinate axis $\mathbf{e}_i$ minimizing the angle to the light direction $\mathbf{L}_j$:
$$ \text{axis} = \arg\min_{i \in \{x,y,z\}} \angle(\mathbf{e}_i, \mathbf{L}_j) $$
The extinction over a cuboid query is computed via the SAT lookup, weighted by $h$:
$$ E = \sum_{\text{cuboids}} \text{SAT\_Lookup}(\text{cuboid}) \cdot h $$

**Parameters:**
* Cone angle $\Psi$: Typical values 1.0 to 5.0 degrees [S1, unofficial].
* Sampling frequency: A few dozen lookups per cone and voxel are sufficient [S1, unofficial].
* Growth rate: Adjusts sampling density based on distance [S1, unofficial].
* Weighting function $h$: Strengthens or weakens shadow effect [S1, unofficial].

**Pitfalls:**
* Using very few cone samples (e.g., 12) results in clearly visible cuboid artifacts [S1, unofficial].
* SAT resolution can be a fraction of the volume dataset size without exposing conspicuous artifacts, but too low resolution causes blurred shadows [S1, unofficial].

## Subtopics

### Approximate Shadowing
An algorithm to compute approximate shadows for volumetric rendering by baking axis-aligned light extinction into textures and interpolating, avoiding full shadow raycasts per voxel or raymarch step [S2]. It computes axis-aligned extinction via cumulative products of extinction along each axis, writing each axis to a different texture concurrently [S2]. It collapses these into two interpolated directional extinction values to reduce bandwidth cost during raymarching [S2]. Precomputation cost is <1ms across tested scenarios [S2].

## Operating characteristics
**Time Complexity:** SAT construction is $O(N)$ where $N$ is the number of voxels [S1, unofficial]. Shadow computation is performed per frame for dynamic light sources, with measured times of 0.0079s/frame for a $64^3$ texture with 50 samples [S1, unofficial].
**Memory Complexity:** Requires storage for the Extinction SAT and Light Cache textures, both $O(N)$ [S1, unofficial].
**Performance:** SAT construction for $256^3$ volume takes 0.311s; for $64^3$ it takes 0.029s [S1, unofficial]. AO/color bleeding computation for $256^3$ texture with 3 shells takes 0.0307s, compared to 3.0818s for explicit sampling of radius 7 [S1, unofficial].
**Strengths:** Fast, flexible, handles any light type and multiple sources, handles light sources within or on the border of the dataset without additional effort [S1, unofficial]. Interactive performance is maintained even with dynamic transfer functions [S1, unofficial].
**Limitations:** Approximation artifacts if cone samples are too few [S1, unofficial]. Requires recomputation of SAT and AO terms when transfer function or light source position changes [S1, unofficial].

## Background
Originated in GPU volume ray-casting for interactive rendering with dynamic transfer functions and light sources [S1, unofficial]. It was developed to provide realistic looking images with improved depth and occlusion effects at interactive frame rates [S1, unofficial].

## Sources
[S1] Extinction-based Shading and Illumination in GPU Volume Ray-Casting, https://www.ifi.uzh.ch/dam/jcr:ffffffff-9e7f-2af5-ffff-ffffcf9259bb/FastShadows.pdf, fetched 2026-10-30 (unofficial)
[S2] Ignis: Eulerian Fluid Simulation and Rendering at VR Frame Rates, https://doi.org/10.1145/3610542.3626120, fetched 2026-10-30
