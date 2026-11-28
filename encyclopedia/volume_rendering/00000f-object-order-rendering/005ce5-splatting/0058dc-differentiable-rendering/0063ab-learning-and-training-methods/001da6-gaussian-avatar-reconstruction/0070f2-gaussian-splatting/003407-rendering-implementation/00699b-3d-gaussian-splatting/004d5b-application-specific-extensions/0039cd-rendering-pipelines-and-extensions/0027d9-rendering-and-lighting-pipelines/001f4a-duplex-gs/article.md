# Duplex-GS

Also called: Proxy-Guided Weighted Blending for Real-Time Order-Independent Gaussian Splatting

## Overview
Duplex-GS is a dual-hierarchy rendering framework for 3D Gaussian Splatting that eliminates popping and transparency artifacts while reducing sorting overhead [S1]. It combines alpha-blending with reformulated Weighted-Sum Rendering (WSR) to achieve real-time order-independent transparency [S1]. Unlike vanilla OIT-based Gaussian Splatting which lacks physical constraints and causes transparency artifacts, Duplex-GS reintroduces physical constraints via cell-level transmittance decay, eliminating transparency artifacts and enabling early termination [S1]. Unlike Sort-free Gaussian Splatting which uses Linear Correction WSR (LC-WSR) with linear depth decay, Duplex-GS uses cell-level sorting to compute a physically grounded transmittance that decays rapidly upon encountering opaque objects [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ ellipsoidal cells (proxies) $\mathcal{C} = \{C_n\}_{n=1}^N$, where each cell $C_n$ is initialized from Structure-from-Motion (SfM) points [S1]. Each cell is associated with a latent feature vector $\mathbf{f}_n$ and emits $K$ neural Gaussians [S1].
*   **Output:** A final rendered color $\mathcal{C}$ for each pixel [S1].

**Data Structures:**
Each cell $C_n$ is parameterized by:
*   Position $\mathbf{x}_n \in \mathbb{R}^3$ [S1].
*   Scale $S_n \in \mathbb{R}^3$ [S1].
*   Quaternion $Q_n \in \mathbb{R}^4$ [S1].
*   Calibration scalar $v_n$ [S1].
*   Latent feature vector $\mathbf{f}_n$ [S1].

Each emitted neural Gaussian $G_{n,k}$ (for $k \in \{1, \dots, K\}$) is defined by:
*   Center $\boldsymbol{\mu}_{n,k} \in \mathbb{R}^3$ [S1].
*   Opacity $\sigma_{n,k} \in [0, 1]$ [S1].
*   Color $\mathcal{C}_{n,k} \in \mathbb{R}^3$ [S1].
*   Scale $s_{n,k} \in \mathbb{R}^3$ [S1].
*   Orientation $\mathbf{q}_{n,k} \in \mathbb{R}^4$ [S1].

**Numbered Steps:**

1.  **Cell Initialization:** Initialize $N$ cells from SfM points. Each cell is centered according to the point cloud [S1]. Each cell is parameterized by $\mathbf{x}_n, S_n, Q_n, v_n$ and associated with a latent feature $\mathbf{f}_n$ [S1].

2.  **Gaussian Decoding (Layer I):** For each cell $n$, decode $K$ neural Gaussians using shared Multi-Layer Perceptrons (MLPs) activated by specific functions [S1]:
    *   **Centers:** Compute centers using learned offsets $\mathcal{O}_{n,k}^{\text{pos}}$ scaled by cell scale $S_n$:
        $$ \boldsymbol{\mu}_{n,k} = \mathbf{x}_n + \mathcal{O}_{n,k}^{\text{pos}} \cdot S_n \quad (1) $$
        where $\mathcal{O}_{n,k}^{\text{pos}} \in [-1, 1]$ [S1].
    *   **Opacities:** Decode opacities via MLP $F_\sigma$ activated by Tanh:
        $$ \{\sigma_{n,1}, \dots, \sigma_{n,K}\} = F_\sigma(\mathbf{f}_n, \delta_n, \mathbf{r}_n) \quad (2) $$
        where $\delta_n$ is viewing distance and $\mathbf{r}_n$ is viewing direction [S1].
    *   **Colors:** Decode colors via MLP $F_c$:
        $$ \{c^{n,1}, \dots, c^{n,K}\} = F_c(\mathbf{f}_n, \delta_n, d_n) \quad (8) $$
        where $d_n$ is viewing direction [S1].
    *   **Scales:** Decode scale ratios via MLP $F_s$ activated by sigmoid:
        $$ \{\mathcal{O}_{n,1}^{\text{state}}, \dots, \mathcal{O}_{n,K}^{\text{state}}\} = F_s(\mathbf{f}_n, \delta_n, d_n) \quad (9) $$
        The Gaussian scale $s_{n,k}$ is derived from cell scale $S_n$ and decoded scale ratio:
        $$ s_{n,k} = \mathcal{O}_{n,k}^{\text{scale}} * S_n \quad (11) $$
        where $\mathcal{O}_{n,k}^{\text{scale}} \in (0, 1)$ [S1].
    *   **Orientations:** Decode quaternions via MLP $F_q$:
        $$ \{\mathbf{q}_{n,1}, \dots, \mathbf{q}_{n,K}\} = F_q(\mathbf{f}_n, \delta_n, d_n) \quad (10) $$
        [S1].

3.  **Constraint Enforcement:** Ensure Gaussians reside within their cells [S1]:
    *   Update cell quaternion $Q_n$ as the normalized weighted average of associated Gaussians' orientations:
        $$ Q_n = \text{norm}\left(\sum_{k=1}^K \mathbf{q}_{n,k} * \sigma_{n,k}\right) \quad (12) $$
        [S1].
    *   Clamp scale ratio to ensure Gaussian regions stay inside the cell:
        $$ \mathcal{O}_{n,k}^{\text{scale}} = \min \left( \mathcal{O}_{n,k}^{\text{scale}}, 1 - \|\mathcal{O}_{n,k}^{\text{pos}}\| \right) \quad (13) $$
        [S1].

4.  **Cell Rasterization (Layer I):**
    *   Assign visible cells to tiles [S1].
    *   Apply radix sort to the cell sequence based on depth to establish a coarse spatial order [S1].
    *   Compute cell-level transmittance $T_n^{\text{cell}}$ for sorted cells:
        $$ T_n^{\text{cell}} = \prod_{i=1}^{n-1} \prod_{k=1}^{n-1} (1 - v_i \cdot \alpha_{i,k}) \quad (16) $$
        where $v_i$ is a learnable calibration scalar and $\alpha_{i,k}$ is the opacity of the $k$-th Gaussian in cell $i$ [S1].
    *   Compute cell weight $w_n$ for WSR:
        $$ w_n = v_n \cdot T_n^{\text{cell}} \quad (15) $$
        [S1].
    *   **Early Termination:** Halt cell-level blending when $T_n^{\text{cell}} < \epsilon$, where $\epsilon$ is a predefined threshold [S1].

5.  **Physical OIT Blending (Layer II):**
    *   Aggregate Gaussians inside each cell in an order-independent WSR manner [S1].
    *   Compute final rendered color $\mathcal{C}$:
        $$ \mathcal{C} = \frac{\sum_{n=1}^N w_n \sum_{k=1}^K \alpha_{n,k} \mathcal{C}_{n,k}}{\sum_{n=1}^N w_n \sum_{k=1}^K \alpha_{n,k}} \quad (14) $$
        where $\mathcal{C}_{n,k}$ is the color of the $k$-th Gaussian in cell $n$ [S1].

**Parameters:**
*   $K$: Predefined number of neural Gaussians emitted per anchor/cell [S1].
*   $v_n$: Learnable calibration scalar for cell $n$, parameterized using spherical harmonic coefficients to capture directional variation [S1].
*   $\epsilon$: Predefined threshold for early termination [S1].

**Pitfalls:**
*   Order confusion caused by overlap among cells is a potential issue, addressed by the calibration scalar $v_n$ and the physically grounded transmittance model [S1].
*   Vanilla OIT-based Gaussian Splatting suffers from transparency artifacts and lack of early termination. Duplex-GS handles this by ensuring transmittance decays rapidly for opaque objects and allowing early termination when $T_n^{\text{cell}} < \epsilon$ [S1].

## Subtopics

### AAA-Gaussians
A method that eliminates sorting analytically by pre-integrating the alpha blending equation to compute an equivalent order-independent transparency through a rational polynomial approximation of the visibility term, suppressing popping artifacts without per-pixel depth sorting [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Drastically reduces memory and computational overhead compared to global Gaussian sorting by performing radix sort at the cell level, as the number of cells is orders of magnitude smaller than the number of Gaussians [S1].
*   **Memory Complexity:** Reduced due to cell-level rasterization and sorting [S1].
*   **Performance:** Enables real-time order-independent Gaussian Splatting [S1].
*   **Strengths:** Eliminates popping artifacts (locally order-independent blending) and transparency artifacts (physically grounded transmittance) [S1]. Allows early termination for acceleration [S1].
*   **Limitations:** Requires maintaining a dual-hierarchy structure (cells and Gaussians) [S1].
*   **When to use:** When real-time performance is required and artifacts from sort-free OIT methods (transparency, popping) are unacceptable [S1].
*   **When not to use:** When scene complexity is low enough that standard alpha-blending with sorting is sufficient, or when strict per-Gaussian depth ordering is required for specific physical effects not covered by the cell approximation [S1].

## Background
Duplex-GS was introduced in a paper submitted to the IEEE for possible publication, titled "Duplex-GS: Proxy-Guided Weighted Blending for Real-Time Order-Independent Gaussian Splatting" [S1]. It builds upon Scaffold-GS, which introduced anchors as local encoding proxies [S1]. Duplex-GS replaces point anchors with ellipsoidal cells to provide explicit geometric structure and visible regions [S1]. It addresses limitations of vanilla OIT-based Gaussian Splatting, which lacks physical constraints and causes transparency artifacts, and Sort-free Gaussian Splatting, which uses linear depth decay that is physically inconsistent [S1].

## Sources
[S1] Duplex-GS: Proxy-Guided Weighted Blending for Real-Time Order-Independent Gaussian Splatting, https://doi.org/10.1109/tcsvt.2026.3666188, fetched 2026-10-30
[S2] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
