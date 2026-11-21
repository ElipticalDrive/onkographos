# Slice-by-Slice Backprojection

Also called: Kajiya/Von Herzen Backprojection, Parallel Ray Backprojection

## Overview
Slice-by-Slice Backprojection reorganizes the backprojection integral into a single sweep through the volume, processing it one horizontal slice at a time [S1, unofficial]. It is designed for cloud-like volumes illuminated by parallel light rays, such as sunlight [S1, unofficial]. It builds on [Texture slicing](25-texture-slicing.md) by reordering the computation to march rays through the volume slice-by-slice [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume density field and a parallel light direction vector (e.g., sunlight) [S1, unofficial].
*   **Output:** Light intensity distribution within the volume [S1, unofficial].

**Data Structures:** Not given in the sources. [S1, unofficial]

**Steps:**
1. Reorganize the backprojection integral into a single sweep through the volume, processing it slice-by-slice [S1, unofficial].
2. Define a series of parallel horizontal slicing planes intersecting the volume bounding box [G].
3. Iterate through each slice plane from the light source direction toward the receiver [S1, unofficial].
4. For each slice, calculate the light intensity contribution by integrating along the parallel ray direction passing through the slice [S1, unofficial].
5. Propagate shadow rays simultaneously slice-by-slice in the same general direction as rendering [S1, unofficial].
6. Accumulate the computed light intensity into the output buffer for the current slice [S1, unofficial].
7. Finalize the image from the accumulated intensities of all slices [S1, unofficial].

**Formulas:** Not given in the sources. [S1, unofficial]

**Parameters:** Not given in the sources. [S1, unofficial]

**Edge Cases and Pitfalls:** Not given in the sources. [S1, unofficial]

## Subtopics

### Multi-Pass Slice-by-Slice Backprojection
A variant that scatters energy in the volume by performing a multi-pass slice-by-slice sweep at random angles [S1, unofficial]. Dachille et al. devised a custom hardware architecture for a cache-efficient implementation of this algorithm [S1, unofficial]. It is chosen when multi-directional scattering effects are required, at the cost of multiple volume passes [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [S1, unofficial]
*   **Memory Complexity:** Not given in the sources. [S1, unofficial]
*   **Performance:** Dachille et al. implemented a cache-efficient version using custom hardware [S1, unofficial].
*   **Strengths:** Efficient single-sweep processing for cloud-like volumes with parallel light [S1, unofficial].
*   **Limitations:** Limited to parallel light assumptions; multi-pass variants require multiple volume traversals [S1, unofficial].
*   **When to Use:** Cloud-like volumes with distant light sources [S1, unofficial].
*   **When Not to Use:** Scenes requiring complex local lighting or non-parallel ray approximations [S1, unofficial].

## Background
The method was introduced by Kajiya and Von Herzen [98] to calculate light intensity for cloud-like volumes efficiently by processing slices in a single sweep [S1, unofficial]. A similar technique was demonstrated in the Heidelberg ray-tracing model [157], where shadow rays were propagated simultaneously slice-by-slice [S1, unofficial]. Dachille et al. [44] extended the approach to multi-pass scattering at random angles and designed custom hardware for cache efficiency [S1, unofficial]. Kniss et al. [113][114] later proposed a single-pass approximation using recursive slice-blurring starting at the light source, exploiting 3D texture mapping hardware and a dual image buffer for interactive frame rates [S1, unofficial].

## Sources
[S1] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
