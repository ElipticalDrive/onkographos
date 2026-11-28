# TileSkipper

Also called: Ours-Global, Ours-Regional, Region-Adaptive Tile Pruning for 3D Gaussian Splatting

## Overview
TileSkipper is a post-training systems adapter for 3D Gaussian Splatting (3DGS) that replaces a single scene-wide contribution cutoff with a static per-Gaussian cutoff policy to reduce tile-pair enumeration work without retraining the checkpoint [S1, unofficial]. It differs from its siblings by using a region-adaptive search based on isolated-deletion distortion proxies and held-out validation, whereas [SnugBox](3378-snugbox.md) focuses on tight bounding boxes and [Warp-Level Culling for Efficient Blending in 3D Gaussian Splatting](12278-warp-level-culling-for-efficient-blending-in-3d-gaussian-spl.md) operates at the warp level for blending efficiency.

## How it works
**Inputs and Outputs:**
*   **Input:** A converged 3DGS checkpoint (frozen parameters), a tile rasterizer using a baseline scene-wide cutoff $\tau_0$, and a set of calibration training views.
*   **Output:** A compiled policy consisting of one `uint8` byte per Gaussian (storing a multiplier index) and a threshold lookup table (LUT) of at most 256 thresholds.

**Data Structures:**
*   **Gaussian Policy:** An array of size $N$ (number of Gaussians), where each element is a `uint8` index into the LUT.
*   **Lookup Table (LUT):** An array of at most 256 `float` values representing the actual cutoff thresholds $\tau_g$. Total size < 1 KB.
*   **Discrete Ladder ($M$):** A fixed set of multipliers: $\{1, 1.05, 1.1, 1.15, 1.25, 1.5, 2, 3, 4, 6, 8\}$ [S1, unofficial].

**Method Steps:**

1.  **Geometric Support Definition:**
    For a Gaussian $g$ with peak opacity $\alpha_g$, the elliptical support boundary at threshold $\tau$ is defined by the squared Mahalanobis radius [S1, unofficial]:
    $$ r_g^2(\tau) = 2 \ln(\alpha_g / \tau) $$
    The support set is $S_g(\tau) = \{x : \alpha_g(x) \ge \tau\}$.

2.  **Runtime Rasterization (Inference):**
    The rasterizer modifies only Stage 2 (Tile-Intersection) of the standard 3DGS pipeline [S1, unofficial]:
    1.  **Project:** Project each visible Gaussian to screen space.
    2.  **Enumerate Pairs:** For each Gaussian $g$, read its policy byte to get index $i$, retrieve multiplier $m_g$ from the LUT, and compute $\tau_g = m_g \tau_0$. Create a (Gaussian, tile) pair if and only if the tile intersects the support $S_g(\tau_g)$.
    3.  **Sort:** Sort each tile’s assigned Gaussians front-to-back.
    4.  **Blend:** Alpha-blend the sorted Gaussians per pixel.

3.  **Compilation (Offline Policy Search):**
    The policy is compiled in five steps using calibration views [S1, unofficial]:

    *   **Step 1: Factorized Probes.**
        For every Gaussian $g$ and every multiplier $m \in M$ (corresponding to ladder rung $\ell$):
        *   Perform a global render at threshold $\tau = m \tau_0$.
        *   Measure pair savings $s_{g,\ell}$: the exact change in enumerated pairs attributed to Gaussian $g$.
        *   Measure risk proxy $\rho_{g,\ell}$: an isolated-deletion RGB-energy statistic. This is computed via a custom backward pass that accumulates the squared contribution of removed pairs, holding others fixed [S1, unofficial]:
            $$ \rho_{g,\ell} = \sum_{v,x,c} \left( \alpha_{g,v}(x) \frac{\partial C_{v,c}(x)}{\partial \alpha_{g,v}(x)} \right)^2 $$
            where the sum is over pixels $x$ and channels $c$ in pairs removed at rung $\ell$, and $v$ denotes calibration views.

    *   **Step 2: Risk Stratification.**
        Bucket Gaussians into $S=8$ quantile strata based on the ratio of savings to risk at the most aggressive ladder rung. The sorting key is [S1, unofficial]:
        $$ \log\left(\frac{s_{g,\ell} + \epsilon}{\rho_{g,\ell} + \epsilon}\right) $$
        This groups Gaussians with similar "cheap-to-remove" characteristics.

    *   **Step 3: Workload-Balanced Spatial Partition.**
        Within each stratum, sort Gaussians by their Morton code (spatial locality). Cut the sorted list into contiguous blocks such that each block has equal measured tile-pair work. This produces $K_{reg}$ regions (default $K_{reg}=64$) [S1, unofficial].

    *   **Step 4: Constrained Search.**
        Aggregate savings and risk per region $c$: $S_{c,\ell} = \sum_{g \in c} s_{g,\ell}$ and $R_{c,\ell} = \sum_{g \in c} \rho_{g,\ell}$.
        For 48 geometrically spaced penalty values $\lambda$, compute the optimal level $\ell^*$ for each region by maximizing efficiency [S1, unofficial]:
        $$ \ell^*_c(\lambda) = \arg \max_{\ell \in M} (S_{c,\ell} - \lambda R_{c,\ell}) $$
        Combine these regional proposals with uniform-level policies (where all Gaussians share one multiplier, $K_{reg}=1$) to form the candidate set.

    *   **Step 5: Held-out Selection.**
        Render each candidate policy on a disjoint set of held-out training views. A candidate is feasible if:
        1.  Minimum per-view Teacher PSNR $\ge P_{min}$.
        2.  Maximum per-view p99 absolute residual $\le E_{max}$.
        
        The Teacher PSNR is calculated against the unmodified renderer $I_0$ [S1, unofficial]:
        $$ P_{teacher} = -10 \log_{10} \text{mean}_{x,c} (I(x,c) - I_0(x,c))^2 $$
        The p99 residual is computed on a fixed 4x spatial subsample. Among feasible candidates, select the one with the greatest pair savings.

**Parameters:**
*   $K_{reg}$: Number of regions. Default: 64 [S1, unofficial].
*   $M$: Discrete ladder of multipliers. Values: $\{1, 1.05, 1.1, 1.15, 1.25, 1.5, 2, 3, 4, 6, 8\}$ [S1, unofficial].
*   $P_{min}$: Floor for teacher PSNR.
*   $E_{max}$: Cap for worst-view p99 absolute residual.
*   $\tau_0$: Baseline scene-wide cutoff (e.g., $1/255$ in Speedy-Splat) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   If the threshold $\tau_g$ reaches the Gaussian's opacity $\alpha_g$, the Gaussian produces no tile pairs but its checkpoint parameters are retained [S1, unofficial].
*   The risk proxy $\rho_{g,\ell}$ omits cross terms when many contributions are removed jointly; it is a ranking proxy, not an exact prediction of joint image error [S1, unofficial].
*   Compilation time averages 9.0 s per scene (max 14.0 s), requiring $O(|M|V)$ rasterizer passes for probes [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Compilation is $O(|M|V)$ rasterizer passes, where $|M|=11$ and $V$ is the number of calibration views. Inference adds negligible overhead (one byte read and LUT lookup per Gaussian).
*   **Memory Complexity:** Policy storage is 1 byte per Gaussian + <1 KB LUT. No additional memory for runtime inference.
*   **Performance:** On 13 scenes from Mip-NeRF 360, Tanks & Temples, and Deep Blending, TileSkipper provides dataset-macro speedups of $1.088\times$ at standard resolution and $1.238\times$ at 3840 pixels wide, with mean PSNR changes of $-0.007$ dB and $-0.023$ dB respectively [S1, unofficial].
*   **Strengths:** Training-free, static policy, no per-frame inference, compatible with existing opacity-aware bounds.
*   **Limitations:** Requires offline compilation; performance gain is modest compared to architectural changes; relies on calibration views for policy selection.

## Background
TileSkipper was introduced by Yadav, Peng, and Chella in 2026 [S1, unofficial]. It addresses the limitation of fixed scene-wide cutoffs in tiled 3DGS rasterizers, which do not account for content sensitivity to support truncation. It builds on [Tile-Based Culling](2819-tile-based-culling.md) by replacing the static cutoff with a learned (compiled) static policy. It is distinct from [AdaGScale](link), which uses runtime view adaptation, by trading runtime adaptation for a scene-level, static one-byte policy [S1, unofficial].

## Sources
[S1] TileSkipper: Region-Adaptive Tile Pruning for 3D Gaussian Splatting, https://arxiv.org/abs/2610.09343, fetched 2026-10-29 (unofficial)
