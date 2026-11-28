# L-System Detail Scalar LOD

Also called: Detail scalar, Ramp LOD

## Overview
L-System Detail Scalar LOD replaces hand-built, discrete LOD stages with a continuous "detail scalar" parameter that maps a value in $[0, 1]$ to raster resolution, grammar depth, leaf cluster count, and porosity [S1, unofficial]. This approach ensures that the visual transition between levels of detail is smooth via crossfading, eliminating visible popping artifacts that occur when switching between discrete mesh variants [S1, unofficial]. It is distinct from previous tree LOD methods by unifying the LOD selection logic into a single scalar function of camera distance, ensuring consistency between window trees and horizon instances [S1, unofficial].

## How it works
### Inputs and Outputs
*   **Inputs:**
    *   Camera distance to the tree or chunk center ($d$).
    *   L-system definition (skeleton, grammar rules, material properties).
    *   Preset variant seed (for horizon instances).
    *   Configuration parameters: `DWARF_EYE_LEAF_PIXELS` (tested at 2.0 and 1.5) [S1, unofficial].
*   **Outputs:**
    *   A cached mesh for the current LOD band.
    *   Triangle count and geometry data for rendering.

### Data Structures
The system maintains a set of cached meshes per tree (for window trees) or per preset variant (for horizon instances). Each mesh corresponds to a specific sample on the detail scalar ramp.

*   **Band Samples:** Discrete points along the detail scalar ramp.
    *   The bottom sample corresponds to 1 voxel/tile resolution, generated via greedy-merging (the "box crown").
    *   Higher resolution samples correspond to 2, 3, and 4 voxels/tile.
*   **Crossfade Bands:** Defined by the gap between adjacent resolution samples.
    *   Bands are labeled: Near (4 voxels/tile), Close (3 voxels/tile), Mid (2 voxels/tile), Far (1 voxel/tile) [S1, unofficial].
    *   Crossfades are log-spaced across 45% of the gap between bands [S1, unofficial].
    *   Specific transition widths: 28 tiles at the first hand-off (Near to Close), 80 tiles at the last hand-off (Mid to Far) [S1, unofficial].

### Steps
1.  **Calculate Detail Scalar:** Compute the detail scalar $s \in [0, 1]$ based solely on camera distance $d$. The mapping ensures that $s=1$ corresponds to the highest detail (4 voxels/tile) and $s=0$ to the lowest (1 voxel/tile) [S1, unofficial].
2.  **Select Bands:** Identify the two adjacent band samples $b_i$ and $b_{i+1}$ that bracket the current detail scalar $s$ [S1, unofficial].
3.  **Compute Blend Factor:** Determine the crossfade weight $\alpha$ based on the log-spaced position of $s$ within the gap between $b_i$ and $b_{i+1}$ [S1, unofficial].
4.  **Generate/Retrieve Meshes:**
    *   If the mesh for band $b_i$ or $b_{i+1}$ is not resident in VRAM, build it lazily from the cached skeleton when the camera enters the band's range plus a margin [S1, unofficial].
    *   Evict meshes when the camera leaves the range (hysteresis) [S1, unofficial].
    *   Regenerate from the cached skeleton upon return [S1, unofficial].
5.  **Apply Porosity Compensation:** Adjust leaf cluster count and porosity during mesh generation to ensure mean color and see-through fraction remain consistent across the ramp. Specifically, maintain wood width from the near band's width in coarser cuts to prevent darkening (bark share ranges from 1.4% to 13%) [S1, unofficial].
6.  **Render:** Draw the blended result using instancing for horizon instances and standard drawing for window trees [S1, unofficial].

### Formulas
*   **Detail Scalar Mapping:** $s = f(d)$, where $f$ maps distance to $[0, 1]$. Exact function not specified, but $s$ controls resolution $R \in \{1, 2, 3, 4\}$ voxels/tile [S1, unofficial].
*   **Crossfade Weight:** $\alpha = \text{log\_interp}(s, b_i, b_{i+1})$, where the interpolation is log-spaced across 45% of the gap [S1, unofficial].
*   **Triangle Count:**
    *   Near band (4 voxels/tile): ~5k triangles per tree [S1, unofficial].
    *   Half band (3 voxels/tile): ~1k triangles per tree [S1, unofficial].
    *   Far band (1 voxel/tile): Few hundred triangles per tree [S1, unofficial].
    *   Far band reduction: 18% fewer triangles compared to previous coarse implementations [S1, unofficial].

### Parameters
*   `DWARF_EYE_LEAF_PIXELS`: Controls leaf pixel size. Tested values: 2.0, 1.5 [S1, unofficial].
*   Crossfade width: 45% of the gap between bands [S1, unofficial].
*   Transition tile counts: 28 tiles (first hand-off), 80 tiles (last hand-off) [S1, unofficial].

### Edge Cases and Pitfalls
*   **VRAM Usage:** A finer ramp increases the number of resident meshes per band. Mitigation is achieved through distance-based residency management (lazy build, hysteresis eviction) rather than reducing the number of samples [S1, unofficial].
*   **Visual Artifacts:** The window boundary must not be visible in tree detail. This is ensured by applying the same LOD chain (4/3/2/1 voxels/tile) and crossfades to horizon instances as to window trees [S1, unofficial].
*   **Color Consistency:** Coarse crowns can appear darker due to increased bark share. This is mitigated by keeping wood width from the near band in coarser cuts, ensuring mean color matches within lighting [S1, unofficial].
*   **Performance Bound:** Per-entity CPU work (visibility ranges, transform propagation, extraction) can become a bottleneck. For example, 88k horizon entities resulted in 5 fps at 4.5M triangles and 16 fps at 240k triangles [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Mesh generation is $O(1)$ per tree per LOD change if cached, but lazy building introduces variable latency. Rendering is $O(N)$ where $N$ is the number of active trees, with instancing reducing CPU overhead for horizon instances [S1, unofficial].
*   **Memory Complexity:** VRAM usage scales with the number of active LOD bands per tree. The near band (~5k triangles) dominates memory for close trees, while far chunks hold only coarse bands, saving memory [S1, unofficial].
*   **Measured Performance:**
    *   Near band: ~5k triangles/tree [S1, unofficial].
    *   Half band: ~1k triangles/tree [S1, unofficial].
    *   Far band: Few hundred triangles/tree [S1, unofficial].
    *   Held triangles increased by ~40% for the extra band, but drawn unchanged [S1, unofficial].
    *   88k horizon entities: 5 fps at 4.5M triangles, 16 fps at 240k triangles [S1, unofficial].
*   **Strengths:** Smooth transitions without popping; consistent visual quality between window and horizon trees; reduced CPU overhead via instancing for distant trees.
*   **Limitations:** Higher VRAM usage due to multiple cached meshes; potential CPU bottleneck for large numbers of horizon entities; complexity in managing residency and hysteresis.
*   **When to Use:** Suitable for scenes with many trees where smooth LOD transitions are critical and VRAM is sufficient to cache multiple LOD levels.
*   **When Not to Use:** Not suitable for memory-constrained environments where caching multiple LOD levels is prohibitive.

## Background
L-System Detail Scalar LOD was developed as part of the Dwarf Eye project, specifically addressing the limitations of hand-built LOD stages in tree rendering. The design was proposed and implemented by aaron sb in September 2026, as documented in GitHub issue #10 [S1, unofficial]. The method evolved from earlier attempts that used discrete LOD stages, aiming to provide a more seamless visual experience by mapping a single detail scalar to various geometric and material properties.

## Sources
[S1] Simplify cached fine chunks far from the camera into a mid-detail surface, https://github.com/aaronsb/dwarf-eye/issues/10, fetched 2026-10-26 (unofficial)
