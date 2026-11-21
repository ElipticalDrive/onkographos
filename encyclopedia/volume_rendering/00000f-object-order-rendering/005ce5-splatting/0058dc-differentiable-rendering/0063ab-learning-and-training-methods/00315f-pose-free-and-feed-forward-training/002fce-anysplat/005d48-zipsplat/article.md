# ZipSplat

Also called: ZipSplat: Fewer Gaussians, Better Splats

## Overview
ZipSplat is a feed-forward method that reconstructs scenes from unposed, uncalibrated images as compact, observation-grounded scene tokens, each derived from a source patch and decoded into freely placed Gaussians [S1, unofficial]. It solves the problem of redundant, misaligned Gaussian predictions across views by aggregating multi-view observations into a fixed set of learned tokens, while allowing the reconstruction budget to be adjusted at inference without retraining [S1, unofficial]. It sets itself apart from its parent [AnySplat](6137-anysplat.md) and sibling [FreeSplat](15163-freesplat.md) by deriving tokens directly from image patches rather than learned queries, and by using a cone loss to ground decoded Gaussians to their source observations, preventing spatial drift [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** $N$ unposed and uncalibrated images $\mathcal{I} = \{I_n \in \mathbb{R}^{H \times W \times 3}\}_{n=1}^N$.
*   **Output:** A set of 3D Gaussians and estimated camera parameters.

**Data Structures:**
*   **Geometry Tokens ($\mathbf{g}$):** One token per image patch, extracted from a multi-view backbone.
*   **Scene Tokens ($\mathbf{z}$):** A compact set of $K$ tokens derived by clustering geometry tokens.
*   **Gaussians ($\mathcal{G}$):** Each scene token $k$ decodes into $G$ Gaussians. Each Gaussian $j$ is parameterized by:
    *   Center $\boldsymbol{\mu}_{kj} \in \mathbb{R}^3$
    *   Scales $\mathbf{s}_{kj} \in \mathbb{R}^3$
    *   Rotation quaternion $\mathbf{q}_{kj} \in \mathbb{R}^4$
    *   Opacity $\alpha_{kj} \in [0, 1]$
    *   Color coefficients $\mathbf{c}_{kj} \in \mathbb{R}^C$ (using degree-1 spherical harmonics) [S1, unofficial].

**Algorithm Steps:**

1.  **Token Extraction:** A multi-view backbone extracts per-patch visual tokens at multiple layers. These are merged into geometry tokens $\mathbf{g}$, one per image patch [S1, unofficial].
2.  **Clustering:** Geometry tokens are clustered into $K$ scene tokens using k-means. The number of tokens $K$ is determined by a compression ratio $r$ at inference: $K = r \cdot T$ (where $T$ is the total number of geometry tokens) [S1, unofficial]. Clustering is guided by the GeoCE loss to ensure patches observing the same surface are merged [S1, unofficial].
3.  **Token Fusion (Refinement):** Scene tokens are refined through alternating cross- and self-attention to recover detail lost during cluster averaging [S1, unofficial].
    *   For $l=1, \ldots, L$:
        $$ \mathbf{z}_k^l = \operatorname{SA}\left( \operatorname{CA} (\mathbf{z}_k^{l-1}, \mathbf{g}, \mathbf{v}_k^l) \right) $$
        where geometry tokens $\mathbf{g}$ serve as keys and backbone features $\mathbf{f}^l$ provide values [S1, unofficial].
    *   Final residual cross-attention incorporates color features:
        $$ \mathbf{z}_k = \mathbf{z}_k^L + \operatorname{CA} (\mathbf{z}_k^L, \mathbf{g}, \mathbf{v}_k^{\mathrm{col}}) $$
        [S1, unofficial].
4.  **Gaussian Decoding:** An MLP decoder $\mathcal{D}$ maps each scene token $\mathbf{z}_k$ into $G$ freely placed Gaussians:
    $$ \mathcal{G}_k = \left\{ \left( \boldsymbol{\mu}_{kj}, \mathbf{s}_{kj}, \mathbf{q}_{kj}, \alpha_{kj}, \mathbf{c}_{kj} \right) \right\}_{j=1}^{G} = \mathcal{D}(\mathbf{z}_k) $$
    The full scene is assembled from all $K$ tokens [S1, unofficial].
5.  **Geometric Grounding (Cone Loss):** To prevent decoded Gaussians from drifting from their source patch, a cone loss penalizes the reprojection distance between Gaussian centers and the source patch center $\mathbf{u}_k$:
    $$ \mathcal{L}_{\mathrm{cone}} = \sum_{k=1}^{K} \sum_{j=1}^{G} \rho \left( \left\| \pi_k(\boldsymbol{\mu}_{kj}) - \mathbf{u}_k \right\| \right) $$
    where $\pi_k$ projects into the source view of token $k$, distances are measured in patch units, and $\rho$ is a robust penalty [S1, unofficial].
6.  **Token Shaping (GeoCE Loss):** To ensure clustering merges patches observing the same surface, a geometric consistency loss aligns token similarity with 3D proximity. Each token's patch is unprojected to a 3D anchor $\mathbf{p}_i$ using median depth. A soft target distribution is constructed:
    $$ w_{ij} \propto \exp \left( -\frac{\|\mathbf{p}_i - \mathbf{p}_j\|^2}{2\sigma_i\sigma_j} \right), \quad j \neq i $$
    where $\sigma_i$ scales with local patch spacing [S1, unofficial].
7.  **TokenBA Refinement:** Cameras and tokens are refined using robust PnP on patch-Gaussian correspondences and updating scene tokens through the frozen decoder using photometric error on input views [S1, unofficial].

**Parameters:**
*   $G=32$: Number of Gaussians per scene token [S1, unofficial].
*   $L=3$: Number of cross- and self-attention blocks for fusion [S1, unofficial].
*   $r$: Compression ratio, sampled uniformly from $[0.2/N, 1]$ during training to enable adjustable budget at inference [S1, unofficial].
*   Loss weights: $\lambda_{geo}=0.05$, $\lambda_{cone}=0.01$, $\lambda_{pose}=0.05$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Gaussian Drift:** Without geometric constraints (cone loss), the decoder can place Gaussians anywhere in 3D, drifting from the source patch [S1, unofficial].
*   **Surface Mixing:** Clustering can merge tokens from different surfaces if not guided by geometric consistency (GeoCE) [S1, unofficial].

## Subtopics

### Token Fusion
Token Fusion is the refinement stage where scene tokens are updated through alternating cross- and self-attention to recover detail lost during the cluster averaging step [S1, unofficial]. It uses geometry tokens as keys to bias retrieval toward relevant observations across views and backbone features as values [S1, unofficial]. It is chosen when high-frequency detail recovery is needed after aggressive compression [S1, unofficial].

## Operating characteristics
*   **Time and Memory Complexity:** The method produces a compact representation where the number of Gaussians scales with the compression ratio $r$ rather than the number of input views, allowing for a continuous quality-efficiency trade-off [S1, unofficial].
*   **Measured Performance:** Trained on RealEstate10K and DL3DV at $256 \times 256$ resolution for 450K steps on 16 GH200 GPUs [S1, unofficial].
*   **Strengths:** Adjustable reconstruction budget at inference; observation-grounded tokens prevent redundancy; pose-free operation [S1, unofficial].
*   **Limitations:** Requires a multi-view backbone and attention mechanisms, which may increase inference latency compared to simpler splatting methods [S1, unofficial].
*   **When to use:** When reconstructing scenes from uncalibrated, unposed image collections with a need for adjustable memory/compute budgets [S1, unofficial].

## Background
ZipSplat was introduced in a preprint in 2026 by the authors of the arXiv paper "ZipSplat: Fewer Gaussians, Better Splats" [S1, unofficial]. It builds on the lineage of pose-free feed-forward training methods like [AnySplat](6137-anysplat.md), addressing the limitation of pixel-to-Gaussian mappings that scale with input size rather than scene complexity [S1, unofficial].

## Sources
[S1] ZipSplat: Fewer Gaussians, Better Splats, https://doi.org/10.48550/arXiv.2606.05102, fetched 2026-10-29 (unofficial)
