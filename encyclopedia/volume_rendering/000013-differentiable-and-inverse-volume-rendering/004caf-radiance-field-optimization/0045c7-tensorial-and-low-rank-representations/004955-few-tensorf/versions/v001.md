# Few TensoRF

## Overview
Few TensoRF is a 3D reconstruction framework that combines the efficient tensor-based representation of [TensoRF](3026-tensorf.md) with frequency-driven regularization from [FreeNeRF](3027-freenerf.md) to improve stability and reconstruction quality under sparse input views [S1, unofficial]. It addresses the problem of high-frequency artifacts and bias that occur when TensoRF converges too rapidly on high-frequency components during few-shot training [S1, unofficial]. By applying a dynamic frequency mask to tensor components, Few TensoRF forces the model to prioritize low-frequency information in early training stages, enhancing data efficiency without sacrificing the fast training times characteristic of the parent method [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Sparse set of input images (e.g., 8–10 views), camera poses, and intrinsics.
*   **Outputs:** A factorized radiance field representation (density and appearance grids) and rendered novel views.

**Data Structures:**
The radiance field is modeled using two distinct grids, both employing Vector-Matrix (VM) decomposition as defined in [TensoRF](3026-tensorf.md):
1.  **Geometry Grid ($G_\sigma$):** Stores volume density $\sigma$.
2.  **Appearance Grid ($G_c$):** Stores appearance features used to compute color $c$.

The density $\sigma$ and appearance features are derived via trilinear interpolation from these grids at location $x$:
$$ \sigma(x) = G_\sigma(x) $$
$$ \mathbf{c}_{feat}(x) = G_c(x) $$
The final color $c$ is computed by a function $S$:
$$ c = S(\mathbf{c}_{feat}(x), \mathbf{d}) $$
where $\mathbf{d}$ is the viewing direction and $S$ is a pre-selected function, typically a small Multi-Layer Perceptron (MLP) or Spherical Harmonics (SH) [S1, unofficial].

**Tensor Decomposition:**
The grids are factorized into rank-1 components. For the appearance grid $G_c$, the decomposition is:
$$ G_c(x) = \sum_{r=1}^{R_o} \left( A^X_{c,r} \circ b^{3r-2} + A^Y_{c,r} \circ b^{3r-2} + A^Z_{c,r} \circ b^{3r} \right) $$
where $R_o$ is the rank, $r$ is the rank index, $A$ are tensor components (mode vectors/matrices), $b$ are basis vectors, and $\circ$ denotes the combination operation defined in the source [S1, unofficial]. The density grid $G_\sigma$ follows a similar VM structure [S1, unofficial].

**Frequency Masking:**
To mitigate over-fast convergence in high-frequency components, a dynamic frequency mask $\alpha(t, T, L)$ is applied to the tensor components $A$ (density) and $A_c$ (appearance). The masked component $A'_L$ is:
$$ A'_L(t, T; x) = A_L(x) \odot \alpha(t, T, L) $$
where $\odot$ is element-wise multiplication [S1, unofficial].

The mask value $\alpha_i$ for the $i$-th bit is defined as:
$$ \alpha_i(t, T, L) = \begin{cases} 1 & \text{if } i \le \frac{t \cdot L}{T} \\ \frac{t \cdot L}{T} - \lfloor \frac{t \cdot L}{T} \rfloor & \text{if } \frac{t \cdot L}{T} < i \le \frac{t \cdot L}{T} + 6 \\ 0 & \text{if } i > \frac{t \cdot L}{T} + 6 \end{cases} $$
where:
*   $t$: current training iteration.
*   $T$: total number of training iterations.
*   $L$: component length (density or appearance feature length) [S1, unofficial].

If $t \ge T$, the mask returns a tensor of ones [S1, unofficial].

**Algorithm Steps:**
1.  Initialize geometry grid $G_\sigma$ and appearance grid $G_c$ using VM decomposition factors.
2.  For each training iteration $t$ from 1 to $T$:
    a. Sample rays and compute density $\sigma$ and appearance features $\mathbf{c}_{feat}$ via trilinear interpolation from $G_\sigma$ and $G_c$.
    b. Compute the dynamic frequency mask $\alpha(t, T, L)$ for both density and appearance components based on current iteration $t$ and total iterations $T$.
    c. Apply the mask to the tensor factors $A$ and $A_c$ using element-wise multiplication.
    d. Render the image using volume rendering integration.
    e. Compute loss against ground truth images.
    f. Backpropagate and update the tensor factors.
3.  After training, render novel views using the final unmasked (or fully active) factors.

**Parameters:**
*   $T$: Total training iterations. Typical values align with TensoRF's fast training, approximately 10–15 minutes of optimization [S1, unofficial].
*   $L$: Length of the tensor component being masked.
*   $S$: The color prediction function (MLP or SH) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Rapid Convergence:** Without masking, TensoRF converges too quickly on high-frequency components, hindering the exploration of low-frequency information and causing bias and high-frequency artifacts in sparse scenarios [S1, unofficial].
*   **Sparse Views:** The method is specifically designed for few-shot scenarios (e.g., 8–10 images). Performance degrades if the input is extremely sparse beyond the tested range [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Maintains the fast training time of TensoRF, approximately 10–15 minutes [S1, unofficial].
*   **Memory Complexity:** Inherits the low memory footprint of the VM decomposition from [TensoRF](3026-tensorf.md).
*   **Performance:** On the Synthesis NeRF benchmark, Few TensoRF improves average PSNR from 21.45 dB (standard TensoRF) to 23.70 dB, with a fine-tuned version reaching 24.52 dB [S1, unofficial]. On the THuman 2.0 dataset, it achieves 27.37–34.00 dB PSNR with only eight input images [S1, unofficial].
*   **Strengths:** High data efficiency for few-shot inputs; prevents high-frequency artifacts; retains real-time rendering speed.
*   **Limitations:** Relies on the effectiveness of the frequency mask schedule; performance is tied to the specific few-shot regime.

## Background
Few TensoRF was introduced by authors from FPT University, Ho Chi Minh City, Vietnam, in a paper published in March 2026 [S1, unofficial]. It builds directly on [TensoRF](3026-tensorf.md) and incorporates regularization techniques from [FreeNeRF](3027-freenerf.md) to address the limitations of tensor-based radiance fields in sparse view settings.

## Sources
[S1] Few TensoRF: Enhance the Few-shot on Tensorial Radiance Fields, https://doi.org/10.48550/arxiv.2603.25008, fetched 2026-10-26 (unofficial)
