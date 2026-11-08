# CasCPRF

Also called: Cascaded Canonical Polyadic Tensorial Radiance Fields, Cascaded Tensorial Radiance Fields, CasCPRF Cascaded Frequency Disentanglement, Cascaded CPD, Cascaded Canonical Polyadic decomposition, IRCasTRF

## Overview
CasCPRF is a multi-scale tensor decomposition radiance field representation that achieves superior compactness compared to TensoRF by employing a cascaded scheme to separate frequencies across multiple hierarchical levels [S1]. It approximates a high-order radiance field feature volume using a tree structure inspired by Hierarchical Tucker decomposition, where vectors in higher levels are estimated by multiplying vectors from lower levels [S1]. This approach implicitly controls the frequency band of learned contents, separating low spatial/high angular frequency components (coarse scale) from high spatial/low angular frequency components (fine scale) to enable robust and compact scene reconstruction [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** $N$ multi-view images $I_j \in \mathbb{R}^{H \times W \times 3}$ captured under the same illumination [S1].
*   **Outputs:** A compact radiance field representation for novel view synthesis, and subsequently, an explicit mesh with learned materials and lighting [S1].

**Data Structures:**
The method approximates a high-order radiance field feature volume $\mathcal{V} \in \mathbb{R}^{I_x \times I_y \times I_z \times (I_\sigma + I_c)}$ [S1].
*   **Cascaded Levels:** The structure consists of $D$ cascaded levels, denoted $d=1 \dots D$. Default $D=3$ [S1].
*   **Vector Subspace:** For each mode $n \in \{x, y, z\}$ and level $d$, rank-one vectors $v_{n,d}$ reside in a subspace $\mathbb{R}^{I_{n,d} \times R}$ [S1].
*   **Resolution Scaling:** The grid dimensions for level $d$ are initialized as $I_{n,d} = I_n / 2^{d-1}$ [S1]. The base grid $I_{n,1}$ is typically $125^3$, upsampled adaptively not exceeding $250^3$ [S1].
*   **Ranks:** The number of components in mode $n$ is $R_n = R_{\sigma n} + R_{c n}$, where $R_{\sigma n}$ and $R_{c n}$ are the ranks for density and color respectively [S1]. $R_n \ll I_x, I_y, I_z$ ensures compactness [S1].

**Formulas:**
1.  **Tensor Decomposition:** The density feature volume $\mathcal{V}_\sigma$ and color feature volume $\mathcal{V}_c$ are factorized as sums over cascaded levels [S1]:
    $$
    \mathcal{V}_\sigma = \sum_{d=1}^{D} \sum_{r=1}^{R_\sigma} \left( v_{rx,d} \times v_{ry,d} \times v_{rz,d} \right)
    $$
    $$
    \mathcal{V}_c = \sum_{d=1}^{D} \sum_{r=1}^{R_c} \left( v_{rx,d} \times v_{ry,d} \times v_{rz,d} \right)
    $$
    where $\times$ denotes the outer product of vectors [S1].

2.  **Positional Encoding:** Sinusoidal positional encoding $PE(x)$ is applied to the viewing direction $\omega_o$ [S1]:
    $$
    PE(x) = [\sin(2^0 \cdot x), \cos(2^0 \cdot x), \dots, \sin(2^{N_q-1} \cdot x), \cos(2^{N_q-1} \cdot x)]
    $$
    where $N_q$ is the number of frequency bands [S1].

3.  **Radiance Calculation:**
    *   Density: $\sigma = \text{ReLU}(F_\sigma)$ [S1].
    *   Color: $c = \text{MLP}_c(F_c, PE(\omega_o))$ [S1].
    *   $F_\sigma$ and $F_c$ are obtained by grid sampling $\mathcal{V}_\sigma(x)$ and $\mathcal{V}_c(x)$ [S1].

4.  **Volume Rendering:**
    *   Sampling: $N_s$ points are sampled per ray along the viewing direction [S1].
    *   Transmittance: $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \Delta_j\right)$, where $\Delta_j$ is the distance between adjacent points [S1].
    *   Color Prediction: $C = \sum_{i=1}^{N_s} T_i c_i \Delta_i$ [S1].

**Parameters:**
*   $N$: Number of input images [S1].
*   $D$: Total cascaded levels. Default: 3 [S1].
*   $I_n$: Base grid dimension for mode $n$. Default: 125 [S1].
*   $R_\sigma, R_c$: Ranks for density and color. Varying budgets tested (e.g., sum of density components $R_{\sigma x} + R_{\sigma y} + R_{\sigma z}$ from 12 to 4) [S1].
*   $\text{MLP}_c$: Two-layer MLP, 128 nodes per layer, ReLU activation [S1].
*   Optimizer: Adam [S1].
*   Learning Rates: CasCPRF parameters: 0.02; $\text{MLP}_c$: 0.0001 [S1].
*   Training Iterations: 30,000 [S1].
*   Batch Size: 4096 rays [S1].

**Algorithm Steps:**

**Stage 1: Exploration (CasCPRF Learning)**
1.  **Initialize:** Set cascaded grid resolutions $I_{n,d} = I_n / 2^{d-1}$ for $d=1 \dots D$. Initialize tensor factors $v_{n,d}$ randomly. Initialize $\text{MLP}_c$ weights [S1].
2.  **Sample Rays:** For each training image, cast rays through pixels. Sample $N_s$ points along each ray [S1].
3.  **Grid Sample:** For each sampled point $x$, compute features $F_\sigma = \mathcal{V}_\sigma(x)$ and $F_c = \mathcal{V}_c(x)$ by evaluating the tensor decomposition sums [S1].
4.  **Compute Radiance:** Calculate $\sigma = \text{ReLU}(F_\sigma)$ and $c = \text{MLP}_c(F_c, PE(\omega_o))$ [S1].
5.  **Volume Render:** Compute accumulated transmittance $T_i$ and render color $C$ using the sampled $\sigma$ and $c$ values [S1].
6.  **Optimize:** Minimize reconstruction loss $L_{expl} = \|C - \hat{C}\|_2^2 + \lambda_{tv} L_{tv}(v)$, where $L_{tv}$ is total variation on vectors $v$ [S1]. Update CasCPRF factors and $\text{MLP}_c$ using Adam [S1].
7.  **Extract Mesh:** After training, estimate a complete density volume and apply the Marching Cubes (MC) algorithm to extract a preliminary mesh $M$ [S1].

**Stage 2: Exploitation (Physics-based Shading)**
*Note: This stage is part of the IRCasTRF pipeline utilizing CasCPRF, as described in the source.*
1.  **Global Alignment:** Initialize global movement vector $\Delta_m$ randomly in $[-0.3, 0.3]$. Learn $\Delta_m$ for 150 warm-up iterations to align the mesh [S1].
2.  **Topology Refinement:** Enable per-vertex deformation vectors $\Delta v_i$ to alter topology. Disable $\Delta_m$ after warm-up [S1].
3.  **Material & Lighting:** Learn Disney material parameters (diffuse $b_d$, roughness $r$, metalness $m$) and environment map $M \in \mathbb{R}^{6 \times 512 \times 512}$ [S1].
4.  **Optimize:** Minimize $L_1$ loss on tone-mapped colors plus regularizers. Use Adam with LR 0.03 for 5000 iterations [S1].

**Edge Cases and Pitfalls:**
*   **Spectral Bias:** Standard MLPs struggle with high frequencies due to NTK eigenvalue decay. CasCPRF mitigates this via the cascaded multi-scale structure, where coarse levels handle high angular frequencies and fine levels handle high spatial frequencies [S1].
*   **Mesh Topology:** The preliminary mesh from MC may contain high-frequency noise, but provides a sufficient topology initialization for the exploitation stage [S1].
*   **Texture Indexing:** Fixed indexes between vertices and 2D texture maps are used to ensure seamless texture variation during topology changes, avoiding slow convergence associated with volumetric texture MLPs [S1].

## Operating characteristics
*   **Time Complexity:** Training the CasCPRF model takes approximately 25 minutes on a single NVIDIA A100 GPU (40GB) with 30,000 iterations and batch size 4096 [S1]. The subsequent exploitation stage (shading pass) takes approximately 1 hour [S1].
*   **Memory Complexity:** Model size is highly compact, ranging from 1 to 3 MB [S1]. This is significantly smaller than NeRF (hours of training, large memory) and DVGO (hundreds of MBs) [S1].
*   **Performance:** CasCPRF outperforms TensoRF across various model-size budgets, with the greatest advantage at exceptionally small sizes (~1.8 MB for NeRF-synthetic, ~2.1 MB for RefNeRF-synthetic) [S1]. It maintains better compression robustness than TensoRF when reducing density components [S1].
*   **Strengths:** Superior compactness, fast training, implicit frequency disentanglement, compatibility with explicit mesh pipelines [S1].
*   **Limitations:** Requires multi-view images under consistent illumination [S1]. The two-stage pipeline adds complexity compared to single-stage NeRFs [S1].

## Background
CasCPRF was introduced by Wenpeng Xing, Jie Chen, Ka Chun Cheung, and Simon See in the paper "IRCasTRF: Inverse Rendering by Optimizing Cascaded Tensorial Radiance Fields, Lighting, and Materials From Multi-view Images" presented at ACM MM 2023 [S1]. The method draws inspiration from TensoRF [9] for tensor decomposition and Hierarchical Tucker decomposition [16, 18] for the tree structure. It also incorporates concepts from BACON [31] regarding band-limited coordinate networks for multi-scale representation. It is used as the volumetric representation component in the IRCasTRF hybrid inverse rendering pipeline [S1].

## Sources
[S1] IRCasTRF: Inverse Rendering by Optimizing Cascaded Tensorial Radiance Fields, Lighting, and Materials From Multi-view Images, https://doi.org/10.1145/3581783.3612010, fetched 2026-10-27
