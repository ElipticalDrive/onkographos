# Early Termination for Gaussian Splatting

Also called: Early Termination (ET)

## Overview
Early Termination (ET) is an acceleration mechanism for Gaussian Splatting that selectively omits computations for Gaussians with negligible impact on the final rendered output [S1]. It leverages physically grounded transmittance to achieve a decay pattern similar to alpha-blending, allowing the rendering process to halt when the remaining contribution is below a threshold [S1]. Unlike sort-free Gaussian Splatting methods (e.g., LC-WSR) which lack early termination due to their design, ET enables significant speedups comparable to vanilla 3DGS without requiring an explicit Gaussian order [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray, a set of valid Gaussians intersected by the ray ($N^{\text{val}}$), and a termination threshold $\epsilon$.
*   **Output:** The final pixel color and the number of Gaussians actually rendered ($N^{\text{rendered}}$).

**Data Structures:**
*   **Valid Gaussians:** A list of Gaussians intersected by the ray. In cell-based variants, these are grouped by cell.
*   **Transmittance:** A scalar value representing the accumulated opacity along the ray.

**Algorithm:**
1.  Initialize accumulated color $C = 0$ and transmittance $T = 1$.
2.  Iterate through the valid Gaussians (or cells) in front-to-back order.
3.  For each Gaussian $i$ (or cell $n$):
    a. Compute the contribution of the Gaussian to the color.
    b. Update the accumulated color: $C \leftarrow C + \text{contribution}_i \times T$.
    c. Update the transmittance: $T \leftarrow T \times (1 - \alpha_i)$, where $\alpha_i$ is the opacity.
    d. Check termination condition:
        i. In standard ET, if $T < \epsilon$, terminate.
        ii. In Cell-Based ET, compute cell-level transmittance $T_{cell}^n$. If $T_{cell}^n < \epsilon$, halt blending for that cell and proceed to the next cell or terminate if no more cells are relevant.
4.  Return $C$.

**Formulas:**
*   **Termination Ratio:** The efficiency of early termination is measured by the ratio of skipped computations:
    $$r^{\text{ET}} = 1 - \frac{N^{\text{rendered}}}{N^{\text{val}}}$$
    where $N^{\text{rendered}}$ is the number of Gaussians contributed to the final color, and $N^{\text{val}}$ is the total number of interacted valid Gaussians along the ray [S1].

*   **Cell-Level Transmittance:** In cell-based variants, the transmittance $T_{cell}^n$ decays towards zero during front-to-back rendering [S1].

**Parameters:**
*   **Threshold ($\epsilon$):** The predefined threshold for early termination. For standard ET, this is maintained at the same value as used in vanilla 3DGS [S1]. For Cell-Based ET, it is an empirical threshold constrained by the opacity parameter $\sigma_{n,k} \in [0, 1]$ [S1]. The exact numerical value is not given in the sources [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** The transmittance must decay sufficiently to trigger termination. If the opacity parameters are not constrained to physically plausible ranges (e.g., [0, 1]), the transmittance may not decay correctly, leading to artifacts or failure to terminate [S1].
*   **Order Independence:** In sort-free methods, the lack of strict front-to-back ordering can affect the accuracy of transmittance decay. However, physically grounded transmittance helps mitigate popping artifacts [S1].

## Subtopics

### Early Termination for Cell Based 3DGS
A rendering optimization that halts the front-to-back blending process at the cell level once the cell-level transmittance decays below a predefined threshold $\epsilon$, eliminating transparency artifacts and accelerating rendering [S1]. Unlike prior Weighted Splatting/Rendering (WSR) methods, this formulation constrains the opacity parameter $\sigma_{n,k}$ to the physically plausible range of [0, 1], allowing the empirical threshold to be directly transferred and facilitating more effective pruning and regularization [S1]. The cell-level transmittance $T_{cell}^n$ is the key innovation that eliminates transparency artifacts and enables early termination [S1].

## Operating characteristics
*   **Performance:** On the Mip-NeRF360 dataset, enabling Early Termination results in 155 FPS, compared to 212 FPS when disabled (w/o ET) [S1].
*   **Quality:** The PSNR with ET enabled is 27.66, compared to 26.60 when disabled [S1].
*   **Comparison:** Duplex-GS achieves termination results comparable to vanilla 3DGS without requiring an explicit Gaussian order, whereas sort-free methods like LC-WSR do not support early termination [S1].
*   **Limitations:** The efficiency of cell search rasterization is contingent upon the degree of coherence between the visible regions of a cell and its associated Gaussians. Discrepancies between spatial intervals of cells and emitted neural Gaussians can introduce redundant computations [S1].

## Background
Early Termination for Gaussian Splatting was introduced in the context of Duplex-GS, a proxy-guided weighted blending method for real-time order-independent Gaussian Splatting [S1]. The method addresses the limitations of prior sort-free Gaussian Splatting methods, which lacked early termination capabilities [S1]. By restoring explicit physical meaning to the opacity parameter and ensuring physically grounded transmittance, the method enables efficient acceleration while maintaining competitive visual quality [S1].

## Sources
[S1] Duplex-GS: Proxy-Guided Weighted Blending for Real-Time Order-Independent Gaussian Splatting, https://doi.org/10.1109/tcsvt.2026.3666188, fetched 2026-10-30
