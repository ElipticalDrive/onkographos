# Mixture-of-Experts and Modular Architectures

Also called: MoE-NeRF, Switch-NeRF, MIMO-NeRF, MoE-INR

## Overview
Mixture-of-Experts (MoE) and Modular Architectures address the computational bottleneck and memory constraints of monolithic Neural Radiance Fields (NeRFs) by decomposing the scene representation into specialized sub-networks (experts) that are dynamically activated based on input characteristics. Unlike standard NeRFs that use a single global function for all spatial locations and viewing directions, these methods partition the input space—whether spatial, temporal, or feature-based—and route queries to a subset of experts, thereby reducing per-ray computation while maintaining high fidelity [S1][S2]. This approach is distinct from simple parallelization (like MIMO-NeRF's batch processing) or explicit grid decompositions (like Voxel Grids) by using learned gating mechanisms to handle irregular data distributions and boundary continuity issues automatically [S1][S2].

## Shared principles
The family shares a core architectural pattern that replaces or augments the monolithic MLP $F_\Theta$ with a routing mechanism and a set of expert networks.

1.  **Gating/Policy Network:** A lightweight network (often an MLP) that takes the input coordinate $\mathbf{x}$ (and potentially viewing direction $\mathbf{d}$ or encoded features $\mathbf{h}$) and outputs a probability distribution $\mathbf{p}$ over $k$ experts. The gating function is typically:
    $$
    \mathbf{p} = \text{Softmax}(G(\mathbf{x}))
    $$
    where $G$ is the policy network [S1][S2].

2.  **Expert Networks:** A set of $k$ specialized networks $\{E_1, \dots, E_k\}$. Each expert $E_j$ is responsible for modeling a specific subset of the input space (e.g., a spatial region, a time interval, or a frequency band). Experts may share weights or have distinct parameters [S1][S2].

3.  **Routing/Dispatch:**
    *   **Hard Routing (Top-1):** During inference, the expert with the highest probability is selected: $j^* = \arg\max_j p_j$. The output is $v = E_{j^*}(\mathbf{x})$ [S1].
    *   **Soft Routing (Weighted):** During training, outputs from all experts (or top-$k$ experts) are combined using the gating probabilities to allow gradient flow to non-selected experts:
        $$
        v_{pred} = \sum_{j=1}^k p_j E_j(\mathbf{x})
        $$
        This acts as a soft-MoE, preventing experts from becoming "dead" [S1].

4.  **Shared Encoder (Optional but common in INR variants):** To maintain continuity across expert boundaries and reduce redundancy, a shared encoder processes the input coordinates into a high-dimensional feature vector $\mathbf{h}$ before routing. This prevents the "boundary discontinuity" artifact common in grid-based decompositions [S1].

5.  **Load Balancing:** To prevent expert collapse (where one expert handles most data), auxiliary loss terms or capacity constraints are often used. For example, a load balancing loss $\mathcal{L}_{lb}$ encourages uniform distribution of coordinates:
    $$
    \mathcal{L}_{lb} = \frac{1}{N^2} \sum_{j=1}^k c_j s_j
    $$
    where $c_j$ is the count of coordinates assigned to expert $j$, and $s_j$ is the sum of assignment probabilities for expert $j$ [S1].

## Subtopics

### [MIMO-NeRF-self-R2](11400-mimo-nerf-self-r2.md)
A variant of the MIMO-NeRF framework that employs two reformulated Multi-Input Multi-Output (MIMO) Multi-Layer Perceptrons (MLPs) with reduction factors $R^1=1$ and $R^2=1$. It reduces training time by processing multiple rays in parallel while maintaining rendering quality comparable to the original NeRF. It is distinguished from other variants (R1, R3–R6) by its specific reduction factors and number of MLPs.

### [Mixture-of-Experts Implicit Neural Representation](13765-mixture-of-experts-implicit-neural-representation.md)
Mixture-of-Experts Implicit Neural Representation (MoE-INR) is an architecture for time-varying volumetric data compression that uses a mixture-of-experts framework to automatically subdivide spatiotemporal fields and dynamically assign input coordinates to specialized expert networks. Unlike conventional INRs that use a single network for the entire field or grid-based INRs that manually decompose the field into spatial blocks (causing boundary discontinuity), MoE-INR automates irregular subdivisions and uses a shared encoder to maintain cross-boundary continuity. It serves as a unified framework that can function as a conventional INR (with $k=1$), a grid-based INR (with spatial pre-training), or an ensemble INR (by averaging multiple experts), while typically operating as a single-expert selection model for efficiency.

### [Mixture-of-Experts Implicit Neural Representation](13765-mixture-of-experts-implicit-neural-representation.md)
*Note: Switch-NeRF is the primary implementation of MLP-based MoE in. Its specific dispatch mechanism is detailed below.*

#### Uniform Dispatch with Capacity Factor
*   **Definition:** A dispatching strategy for MLP-based gating that limits the number of input samples per NeRF expert to ensure uniform tensor dimensions, balanced computation, and predictable communication overheads, dropping overflow points and zero-padding unused capacity [S2].
*   **Mechanism:** Dispatch points to experts based on gating indexes. If an expert's capacity is exceeded, drop overflow points. If capacity is not fully used, zero-pad the tensor to maintain uniform shape [S2].
*   **Formula:** Maximum samples per expert $kBC_e = \lceil \frac{B}{n} C_f \rceil$, where $B$ is total batch size, $n$ is number of NeRF experts, and $k$ is number of chosen experts per sample [S2].
*   **Parameters:** Capacity factor $C_f$: limits input sample number per expert. Empirically set to 1.0 in training to avoid consuming extra memory [S2].
*   **Pitfalls:** Using uniform dispatch during inference causes accuracy degradation and rendering artifacts due to dropped sample points, because the single gate network is less robust to point dropping than stacked MoE layers [S2].

#### MIMO-NeRF-self-R3
*   **Definition:** A variant employing $N_p-1$ reformulated MIMO MLPs with reduction factors $R^1=1, \dots, R^{N_p-1}=1$ [S1].
*   **Formula:** The total number of MLPs running is calculated as $\sum_{m=1}^{N_p-1} \frac{N}{N_p} R^m = N \frac{N_p-1}{N_p} < N$ [S1].
*   **Performance:**
    *   Blender ($N_p=4$): PSNR 30.89, SSIM 0.949, LPIPS 0.060, T-time 5.05h [S1].
    *   LLFF ($N_p=4$): PSNR 27.27, SSIM 0.860, LPIPS 0.171, T-time 3.78h [S1].
    *   Blender ($N_p=8$): PSNR 29.88, SSIM 0.937, LPIPS 0.080, T-time 7.75h [S1].
    *   LLFF ($N_p=8$): PSNR 25.66, SSIM 0.797, LPIPS 0.243, T-time 5.97h [S1].

## Background
The concept of Mixture-of-Experts originates from classical machine learning and has been adapted for NeRFs to handle large-scale scenes and complex data distributions.

*   **MIMO-NeRF:** Introduced in 2023, MIMO-NeRF focuses on parallelizing ray processing using Multi-Input Multi-Output MLPs to accelerate training without sacrificing quality [S1].
*   **MoE-INR:** Developed for time-varying volumetric data, MoE-INR addresses the limitations of static INRs by using a policy network to route spatiotemporal coordinates to specialized experts, improving compression and reconstruction accuracy [S1].
*   **Switch-NeRF:** Proposed to handle large-scale scenes by partitioning the scene into heterogeneous experts. It introduces capacity-constrained dispatching to manage memory and computation efficiently [S2].

These methods are used in applications requiring high-fidelity rendering of large-scale environments, dynamic scenes, and efficient training of neural radiance fields.

## Sources
[S1] MIMO-NeRF: Fast Neural Rendering with Multi-input Multi-output Neural Radiance Fields, https://arxiv.org/abs/2310.01821v1, fetched 2026-10-26
[S2] Learning Heterogeneous Mixture of Scene Experts for Large-Scale Neural Radiance Fields., https://doi.org/10.1109/tpami.2025.3603305, fetched 2026-10-26
