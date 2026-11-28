# Physically Grounded View Selection

Also called: Geometry-aware view selection, Hash-based View Selection, NBV Selection via Surface Coverage, SA-Points based view selection

## Overview
Physically Grounded View Selection is a pre-filtering strategy for Next-Best-View (NBV) selection that reduces computational cost by restricting candidate views to those offering maximal geometric novelty. It solves the bottleneck of exhaustive uncertainty evaluation (e.g., FisherRF) by first filtering candidates based on geometric dissimilarity measured via hash-encoded SA-Points, then ranking the filtered subset by uncertainty scores [S1]. Unlike sibling methods that evaluate all candidates, this approach uses a coarse-to-fine strategy: SA-Points provide an explicit estimate of observed regions to form a stable candidate subset, which is then refined through uncertainty-aware reasoning [S1].

## How it works
**Inputs:**
- A set of candidate views $\mathcal{V}_{cand} = \{v_1, \dots, v_M\}$ [S1].
- A set of previously selected/observed views $\mathcal{V}_{obs}$ [S1].
- A hash-based scene representation (Instant-NGP style) encoding the current occupancy state [S1].
- A retention percentage $N$ (e.g., $N=20$) [S1].

**Outputs:**
- A filtered candidate set $\mathcal{C}' \subset \mathcal{V}_{cand}$ containing the top $N\%$ most dissimilar views [S1].

**Data Structures:**
- **Binary Occupancy Vector ($\mathbf{b}$):** A binary vector representing voxel occupancy in a 3D grid.
- **Hash Map:** A fixed random hashing function mapping voxel coordinates to indices in the binary vector, following the spatial hashing strategy of Instant-NGP [S1].
- **Grid Size ($K$):** The total number of voxels in the grid [S1].

**Steps:**
1. **Generate SA-Points:** Reconstruct 3D point maps (SA-Points) from a training view and its rasterized extrapolated views [S1].
2. **Encode Occupancy:** Map the voxel coordinates of the currently observed volume (derived from $\mathcal{V}_{obs}$) through the fixed random hashing function to create the binary occupancy vector $\mathbf{b}_{obs}$ [S1].
3. **Evaluate Candidates:** For each candidate view $v_{cand}^{(j)} \in \mathcal{V}_{cand}$:
    a. Identify voxels within the view's frustum: $\mathcal{V}_{cand}^{(j)} = \{ v_k \in \mathcal{V} \mid v_k \in \mathcal{F}_j \}$ [S1].
    b. Map these frustum voxels through the same hashing function to create the candidate binary occupancy vector $\mathbf{b}_{cand}^{(j)}$ [S1].
    c. Compute the normalized Hamming distance $d_j$ between $\mathbf{b}_{obs}$ and $\mathbf{b}_{cand}^{(j)}$ using the formula below [S1].
4. **Rank and Filter:** Rank all candidate views in descending order of their normalized Hamming distance $d_j$. Retain the top $N\%$ of views as the physically filtered candidate set $\mathcal{C}' = \text{TopN\%}(\mathcal{C}, d)$ [S1].
5. **Final Selection:** Apply uncertainty quantification (e.g., Fisher information) only within $\mathcal{C}'$ to finalize the NBV [S1].

**Formula:**
The normalized Hamming distance $d_j$ quantifies the proportion of voxels with inconsistent occupancy status between the currently observed volume and the candidate view's frustum [S1]:

$$
d_j = \frac{1}{K} \left\| \mathbf{b}_{obs} \oplus \mathbf{b}_{cand}^{(j)} \right\|_1
$$

Where:
- $\oplus$ denotes element-wise XOR operation [S1].
- $\|\cdot\|_1$ is the $\ell_1$ norm (count of differing entries) [S1].
- $K$ is the total number of voxels in the grid [S1].
- $d_j \in [0, 1]$ measures the volume of non-overlapping occupancy, i.e., how many newly covered voxels a candidate adds [S1].

**Parameters:**
- $N$: Percentage of candidates to retain. Typical value: $N=20$ [S1].

**Edge Cases and Pitfalls:**
- **Precision:** The hashing function must be fixed and consistent between $\mathbf{b}_{obs}$ and $\mathbf{b}_{cand}^{(j)}$ to ensure valid comparison [S1].
- **Boundaries:** The grid resolution $K$ affects the granularity of the dissimilarity measure; too coarse may miss small novel regions, too fine increases memory [S1].
- **Degenerate Input:** If $\mathcal{V}_{obs}$ is empty, $\mathbf{b}_{obs}$ is all zeros, and $d_j$ simply measures the coverage of the candidate view [G].

## Subtopics

### SA-ResGS Four-Step View Selection Pipeline
This variant replaces exhaustive Fisher evaluation with a four-step pipeline: (1) dense correspondence prediction using MASt3R, (2) SA-Points triangulation, (3) physically grounded prefiltering, and (4) Fisher computation restricted to the filtered views [S1]. It is chosen when computational efficiency is critical, as it achieves 55% faster view selection and 40% reduced end-to-end runtime on a mid-range GPU (38 TFLOPS fp32) for the Bonsai scene, with only a modest increase in per-iteration GPU memory usage [S1].

## Operating characteristics
- **Time Complexity:** Significantly faster than exhaustive FisherRF. On the Bonsai scene with a mid-range GPU (38 TFLOPS fp32), view selection is 55% faster [S1].
- **Memory Complexity:** Modest increase in per-iteration GPU memory usage compared to baseline, but remains within standard limits [S1].
- **Performance:** End-to-end runtime reduced by 40% [S1].
- **Strengths:** Reduces computational bottleneck of per-Gaussian Fisher information backpropagation across all candidates; encourages exploration of under-observed regions [S1].
- **Limitations:** May inadvertently reduce multi-view overlap since dissimilar views are less likely to observe shared regions, weakening multi-view geometric constraints [S1]. This is counterbalanced by residual supervision in the full SA-ResGS framework [S1].
- **When to Use:** Active scene reconstruction where computational cost is a bottleneck and sparse views are used [S1].
- **When Not to Use:** When maximal multi-view overlap is required for geometric constraints without additional supervision mechanisms [S1].

## Background
Physically Grounded View Selection is part of the SA-ResGS (Self-augmented Residual 3D Gaussian Splatting) framework, which stabilizes uncertainty quantification and enhances uncertainty-aware supervision for next-best-view selection in active scene reconstruction [S1]. It was introduced to mitigate bias in uncertainty estimates and address artifacts caused by shape-radiance ambiguity in in-situ reconstruction [S1]. The method builds on [SA-Points (Self-Augmented Pre-filtering)](20845-sa-points-self-augmented-pre-filtering.md) by using SA-Points to generate geometrically dissimilar candidates [S1].

## Sources
[S1] SA-ResGS: Self-augmented Residual 3D Gaussian Splatting for Next Best View Selection, https://doi.org/10.1007/978-3-032-37013-6_17, fetched 2026-10-29
