# Volume Shadow Calculation via Light Strength Buffer

Also called: 3D Shadow Casting, Light Strength Buffer Method

## Overview
This method adds shadows to mixtures of geometric and volumetric data by storing shadow information in a 3D light strength buffer instead of a 2D shadow depth buffer [S1, unofficial]. It sets itself apart from its sibling [Ray Marching with Shadow Mapping](1415-ray-marching-with-shadow-mapping.md) by computing a fractional light strength at every point in space, which allows it to correctly render penumbras and semi-transparent shadows without the aliasing artifacts associated with Williams' 2D shadow depth buffer [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Volumetric data, geometric data (polygons), and a set of light sources [S1, unofficial].
*   **Outputs:** A final composited image with correct volumetric shadows [S1, unofficial].

**Data Structures:**
1.  **Composite Color/Opacity Volumes:** For each light source $i$, a 3D volume storing composite color $C_i$ and composite opacity $O_c$ for each voxel [S1, unofficial].
2.  **3D Light Strength Buffer:** A 3D volume storing the light strength $P_{i,j}$ at each sample position along each ray for each light source $i$ [S1, unofficial].

**Steps:**
1.  Shade and classify the volumetric data, shade and scan-convert the polygons, and use volume matting operators to yield composite colors $C_i$ and opacity $O_c$ for each voxel for each light source $i$ [S1, unofficial].
2.  Cast parallel illumination rays from each light source into the data [S1, unofficial].
3.  For each ray, assume an initial light strength, draw samples at equally spaced positions along the ray, compute the opacity at each location by tri-linearly interpolating from the nearest eight voxels, and attenuate the ray strength [S1, unofficial]. Store the computed light strengths $P_{i,j}$ in the 3D light strength buffer [S1, unofficial].
4.  Compute the total color $c_t$ for each voxel as the weighted sum of light strengths mapped from voxel indices into the coordinate system of each light source $i$, tri-linearly interpolating from the nearest eight light strength values [S1, unofficial].
5.  Ray trace, resample, and composite the arrays of total color $c_t$ and composite opacity $o_c$ [S1, unofficial].

**Formula:**
The strength $P_{i,j+k}$ of a ray as it leaves a sample location is computed from the strength $P_{i,j}$ as it approaches the sample and the opacity $\alpha$ of the sample [S1, unofficial]:

$$ P_{i,j+k} = P_{i,j} \left(1 - \frac{k_{i,j}}{d^2}\right) $$

Where:
*   $P_{i,j}$ is the light strength approaching the sample [S1, unofficial].
*   $P_{i,j+k}$ is the light strength leaving the sample [S1, unofficial].
*   $k_{i,j}$ is a constant for the linear approximation of square-law attenuation [S1, unofficial].
*   $d$ is the distance from the light source to the sample location [S1, unofficial].

The total color $c_t$ for each voxel is computed as [S1, unofficial]:

$$ c_t = \sum_{i=1}^{L} P_i C_i $$

Where $P_i$ is the light strength obtained by mapping voxel indices into the coordinate system of light source $i$ and tri-linearly interpolating from the nearest eight light strength values [S1, unofficial].

**Pitfalls:**
Surfaces in volumetric data (which often have a thickness of several voxels) tend to partially shadow themselves, causing distracting aliasing artifacts [S1, unofficial].
*   **Solution:** Translate the 3-D light strength volume a few voxels away from the light source before computing $c_t$ [S1, unofficial]. This reduces accuracy within small objects but avoids the artifacts [S1, unofficial].

## Subtopics

### [Half-Angle Slicing for Incremental Lighting](3446-half-angle-slicing-for-incremental-lighting.md)
A technique that uses a shared slicing axis aligned between the viewer and light source to avoid the visual artifacts of volumetric shadow maps. It sets itself apart from this method by utilizing a two-pass loop per slice to accumulate light attenuation, whereas this method uses a full 3D buffer.

## Operating characteristics
*   **Memory:** The amount of memory required for a 3D buffer is obviously much greater than a 2D shadow depth buffer [S1, unofficial].
*   **Advantages:** Avoids shadow aliasing problems encountered by Williams' 2D shadow depth buffer [S1, unofficial]. Correctly handles semi-transparent shadows and penumbras [S1, unofficial]. Correctly renders shadows cast by volumetrically defined objects on themselves, shadows cast by polygons on volumetric objects, and vice versa [S1, unofficial].
*   **Limitations:** High memory consumption due to the 3D nature of the buffer [S1, unofficial]. The solution to self-shadowing aliasing (translating the buffer) reduces accuracy within small objects [S1, unofficial].

## Background
The method employs a two-pass approach similar to Williams [1978], but stores shadow information in a 3D light strength buffer instead of a 2D shadow depth buffer [S1, unofficial].

## Sources
[S1] DTIC ADA208085: Rendering Mixtures of Geometric and Volumetric Data, https://archive.org/details/DTIC_ADA208085, fetched 2026-10-26 (unofficial)
