# Per-sample LOD selection

Also called: Adaptive LOD Sampling, Per-sample Level of Detail

## Overview
Per-sample LOD selection is a volume ray marching optimization where the required data resolution is determined on a per-sample basis [S1]. It allows rays to automatically select the locally appropriate data resolution, balancing rendering quality against memory bandwidth and computational cost [S1]. It differs from per-brick LOD selection [FSK13] by allowing resolution changes within a single brick traversal, and from uniform sampling by adapting to local scene complexity [S1].

## How it works
**Inputs:** Ray origin, ray direction, ray length, volume data hierarchy (e.g., SVO or brick-based structure), transfer function.
**Outputs:** Accumulated color and opacity for the pixel.

**Data Structures:**
The method relies on a hierarchical data structure (such as an Octree or Sparse Voxel Octree) where nodes or bricks have associated LOD levels. Each sample point $p$ along the ray is associated with a specific LOD level $L(p)$.

**Steps:**
1.  **Ray Marching Loop:** For each pixel, cast a ray $r$ through the volume.
2.  **LOD Determination:** At each sample step, determine the required LOD level $L$ for the current position. This is typically based on the projected size of the voxel at that depth relative to the pixel size, often derived from the Nyquist sampling theorem to avoid aliasing [S2, unofficial].
3.  **Data Fetch:** Retrieve the volume data at the determined LOD level $L$.
4.  **Interpolation:**
    *   **Discrete LOD:** If using discrete LOD levels, perform tri-linear interpolation using the data from the single selected LOD level [S1].
    *   **Fractional LOD:** If using fractional LOD, interpolate between data from two neighboring LOD levels (e.g., $L$ and $L+1$). This is referred to as "quad-linear" interpolation [S1].
5.  **Accumulation:** Apply the transfer function to get color and opacity, then accumulate using front-to-back or back-to-front compositing.

**Formulas:**
*   **Discrete LOD Interpolation:**
    $$ C = \sum_{i=0}^{N-1} c_i \cdot \alpha_i \cdot \prod_{j=0}^{i-1} (1 - \alpha_j) $$
    Where $c_i$ and $\alpha_i$ are the color and opacity at sample $i$, obtained via tri-linear interpolation from the selected LOD level.
*   **Fractional LOD Interpolation:**
    Let $L_{int}$ be the fractional LOD level. Let $L_{floor} = \lfloor L_{int} \rfloor$ and $L_{ceil} = \lceil L_{int} \rceil$. Let $w = L_{int} - L_{floor}$.
    The interpolated value $V$ is:
    $$ V = (1-w) \cdot V_{floor} + w \cdot V_{ceil} $$
    Where $V_{floor}$ is obtained via tri-linear interpolation from LOD $L_{floor}$ and $V_{ceil}$ from LOD $L_{ceil}$ [S1].

**Parameters:**
*   **LOD Selection Criterion:** Typically based on the Nyquist rate relative to the pixel footprint at the sample depth [S2, unofficial].
*   **Interpolation Type:** Discrete (tri-linear) or Fractional (quad-linear) [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** Fractional LOD requires careful handling of interpolation weights to avoid artifacts at LOD boundaries [S1].
*   **Boundaries:** If the ray exits the volume or reaches the maximum LOD depth without finding allocated data, the interval is treated as free space [S2, unofficial].
*   **Incomplete Data:** If the SVO is still being built and the ray reaches the global maximum depth, a coarser node is returned instead of treating it as free space to ensure continuous sampling [S2, unofficial].

## Subtopics

### LoD-aware Stratified Sampling
A scene sampling scheme that adapts sampling density to the Level of Detail (LoD) of the Sparse Voxel Octree (SVO) [S2, unofficial]. It samples scene models via multiple steps: uninformed stratified sampling, stochastic filtering, and opacity-based filtering [S2, unofficial]. For each ray $r$, it creates sampling points $s_{r,i}$ via stratified sampling, randomly distributing points within each SVO node intersected by $r$ while ensuring sampling density depends on the side length of the intersected node [S2, unofficial]. The ideal tree depth $d$ is inferred using the Nyquist sampling theorem to find the node with the highest LoD still free of aliasing [S2, unofficial]. The constant relative sampling density factor $k$ is set to 8 samples per node edge length [S2, unofficial]. If no tree node is allocated at the inferred depth, the ray depth interval is treated as free space. Special case: if the SVO is still being built up and traversal reaches the global maximum depth, a coarser higher level node is returned for sampling instead of treating it as free space [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of samples per ray and the cost of LOD determination. Fractional LOD may incur slightly higher computational cost due to dual LOD fetches and interpolation [S1].
*   **Memory Complexity:** Requires access to multiple LOD levels of the volume data. The working set size depends on the spatial distribution of high-frequency details [S1].
*   **Performance:** Reduces aliasing and improves quality in high-frequency regions while saving computation in smooth regions [S1]. Measured on electron microscopy volumes (90 GB), it allows efficient rendering by skipping empty space and adapting resolution [S1].
*   **Strengths:** High visual quality, adaptive to local detail, reduces aliasing.
*   **Limitations:** Increased complexity in LOD management, potential for artifacts if LOD transitions are not smooth [S1].
*   **When to use:** When visual quality is critical and the volume has varying frequency content.
*   **When not to use:** When computational resources are extremely limited and uniform low resolution is acceptable.

## Background
Per-sample LOD selection was introduced as an optimization for GPU-based volume rendering [S1]. It builds on earlier work in empty space skipping and brick-based rendering [GMG08, Eng11, HBJP12]. The method was highlighted in the context of large-scale volume visualization, demonstrating its effectiveness on large datasets like electron microscopy volumes [S1]. The concept of fractional LOD interpolation was also explored in this context [S1].

## Sources
[S1] State-of-the-Art in GPU-Based Large-Scale Volume Visualization, https://vcg.seas.harvard.edu/publications/20150513-stateof-the-art-in-gpu-based-large-scale-volume-visualization/paper, fetched 2026-10-24
[S2] ERF: Explicit Radiance Field Reconstruction From Scratch, https://arxiv.org/abs/2203.00051v1, fetched 2026-10-26 (unofficial)
