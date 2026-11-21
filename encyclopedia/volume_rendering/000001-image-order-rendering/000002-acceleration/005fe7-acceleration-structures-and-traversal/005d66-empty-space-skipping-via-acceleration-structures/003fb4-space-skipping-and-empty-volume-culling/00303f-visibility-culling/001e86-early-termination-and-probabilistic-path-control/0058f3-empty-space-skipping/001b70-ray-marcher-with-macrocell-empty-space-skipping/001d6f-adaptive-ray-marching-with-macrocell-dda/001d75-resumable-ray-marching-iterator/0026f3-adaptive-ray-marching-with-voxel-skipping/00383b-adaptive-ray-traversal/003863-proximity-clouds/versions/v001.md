# Proximity Clouds


## Overview
Proximity clouds are an acceleration technique for 3D grid traversal that allows a ray to perform long skips through empty space using pre-computed distance values [S1]. It extends adaptive ray traversal by storing the distance to the nearest occupied voxel for each empty voxel, enabling the ray marcher to jump directly to the next potential data interaction [S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D voxel grid where each voxel stores a scalar value (the "proximity value") and potentially occupancy data.
- Output: A ray traversal path that skips empty regions, reducing the number of samples taken in empty space.

**Data Structures:**
- **Proximity Grid:** A 3D array of integers (or fixed-point numbers). Each cell $(i, j, k)$ stores an integer $n$, representing the number of voxels to skip along the ray direction.
- **Occupancy Grid:** A binary mask or similar structure indicating which voxels contain data [S2, unofficial].

**Pre-processing:**
1. For every empty voxel in the grid, compute the distance to the nearest occupied voxel.
2. Store this distance (or a quantized version thereof) as the proximity value for that voxel [S2, unofficial].

**Ray Marching Algorithm:**
1. Initialize ray origin $O$ and direction $D$.
2. Determine the starting voxel index $V_{start}$ for the ray.
3. While the ray has not exited the volume:
    a. Retrieve the proximity value $n$ from the current voxel $V_{current}$ [S2, unofficial].
    b. If $V_{current}$ is occupied:
        i. Perform standard volume rendering steps (sample, composite, etc.) [S2, unofficial].
        ii. Advance the ray by one voxel step (or a small fixed step) to handle potential transitions or detailed sampling [G].
    c. If $V_{current}$ is empty:
        i. Skip $n$ voxels along the ray direction. This is implemented by advancing the voxel index by $n$ steps in the direction of $D$ [S2, unofficial].
        ii. Update the ray position to the new voxel location.
    d. Check for exit conditions (end of volume, alpha threshold) [G].

**Formulas:**
- Let $n$ be the proximity value of the current voxel.
- The next voxel index $V_{next}$ is calculated as:
  $$ V_{next} = V_{current} + n \cdot \text{step}(D) $$
  where $\text{step}(D)$ is the voxel traversal step vector in the direction of $D$ [S2, unofficial].

**Parameters:**
- Proximity value $n$: Integer representing the skip length. Typical values depend on the density of the data [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** The proximity value must be quantized to fit in the data structure. If the distance is large, the integer representation may lose precision, leading to overshooting or undershooting the nearest occupied voxel [S2, unofficial].
- **Boundaries:** Ensure that skipping $n$ voxels does not exceed the volume bounds.
- **Degenerate Input:** If a voxel is marked as empty but has a proximity value of 0, it may cause an infinite loop if not handled [G].

## Subtopics

### Adaptive Ray Traversal
The parent method that Proximity Clouds extends. It uses hierarchical grids to skip empty space but does not pre-compute distance values for empty voxels. [Adaptive Ray Traversal](8185-adaptive-ray-traversal.md)

## Operating characteristics
- **Time Complexity:** Pre-processing is $O(N)$ where $N$ is the number of voxels. Ray traversal time is reduced in sparse volumes due to skipping, but overhead exists for computing proximity values.
- **Memory Complexity:** Requires additional storage for the proximity grid, typically one integer per voxel [S2, unofficial].
- **Strengths:** Effective for sparse volumes with large empty regions. Reduces the number of samples taken in empty space [S1].
- **Limitations:** Pre-processing cost can be high for large volumes. Quantization errors may affect accuracy [S2, unofficial].
- **When to Use:** Suitable for sparse volumetric data where empty space skipping is beneficial. Not ideal for dense data where most voxels are occupied [S2, unofficial].

## Background
Proximity clouds were proposed by Cohen and Sheffer in 1994 [S1]. The method is an extension of adaptive ray traversal techniques, aiming to improve performance by leveraging distance information to skip empty space [S2, unofficial]. It is used in direct volume rendering to optimize ray traversal in sparse datasets [S1].

## Sources
[S1] A Survey on Exploiting Grids for Ray Tracing, https://doi.org/10.2312/localchapterevents/italchap/italianchapconf2008/089-096, fetched 2026-10-24
[S2] Direct Volume Rendering Techniques, https://www.scribd.com/document/853597780/VIS-Modules-06-Direct-Volume-Rendering, fetched 2026-10-25 (unofficial)
