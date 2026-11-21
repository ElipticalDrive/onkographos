# Patch-based Depth Correlation Loss

Also called: Depth Pearson Correlation, Pearson depth loss

## Overview
A regularization loss that encourages rendered depth maps to align with monocular depth priors by maximizing the Pearson Correlation Coefficient (PCC) between corresponding image patches, thereby enforcing local structural consistency without requiring global scale/shift alignment. It differs from [Patch-level Scale-Shift Alignment](12336-patch-level-scale-shift-alignment.md) by avoiding explicit parameter estimation for scale and shift, which can introduce unwanted distortion if the transformation is not constant across pixels, and instead relies on rank-order similarity within local neighborhoods.

## How it works
**Inputs and Outputs:**
- **Input:** Rendered depth map $d_{softmax}$ (computed via softmax-scaling), monocular depth prior $d_{pt}$ (e.g., from a depth estimation network), patch size $S$, number of patches $N$.
- **Output:** Scalar loss value $L_{depth}$.

**Data Structures:**
- $d_{softmax}$: 2D array of shape $(H, W)$, containing depth values computed using softmax-weighted accumulation.
- $d_{pt}$: 2D array of shape $(H, W)$, containing depth values from the monocular prior.
- Patches: Sub-regions of size $S \times S$ extracted from $d_{softmax}$ and $d_{pt}$.

**Steps:**
1. **Render Depth:** Compute the rendered depth map $d_{softmax}$ using softmax-scaling. This technique biases depth accumulation toward Gaussians with higher weights, approximating the mode depth while propagating gradients to off-mode Gaussians [S1]. The softmax temperature parameter $\beta$ modulates this amplification [S1].
2. **Sample Patches:** Randomly sample $N$ non-overlapping patches from the image. The patch size $S$ is a hyper-parameter, typically set to $1/100$ of the image resolution (e.g., $64 \times 64$ for MipNeRF360, $32 \times 32$ for other resolutions) [S1].
3. **Compute Local Loss:** For each sampled patch $i$, calculate the Pearson Correlation Coefficient (PCC) between the patch from the rendered depth $p^i_{softmax}$ and the corresponding patch from the depth prior $p^i_{pt}$.
4. **Aggregate Loss:** Average the normalized correlation differences across all $N$ patches.
5. **Global Constraint:** Apply a global Pearson depth loss in addition to the patch-based loss to prevent depth discontinuities at patch edges [S1].

**Formula:**
The local patch-based loss is defined as [S1]:

$$
L_{depth} = \frac{1}{N} \sum_{i=1}^{N} \frac{1 - PCC(p^i_{softmax}, p^i_{pt})}{2}
$$

Where:
- $N$ is the number of sampled patches.
- $PCC(a, b)$ is the Pearson Correlation Coefficient between vectors $a$ and $b$.
- $p^i_{softmax} \in \mathbb{R}^{S^2}$ is the $i$-th patch of $d_{softmax}$.
- $p^i_{pt} \in \mathbb{R}^{S^2}$ is the $i$-th patch of $d_{pt}$.

The PCC between two vectors $u$ and $v$ of length $M$ is computed as [S1]:

$$
PCC(u, v) = \frac{\sum_{j=1}^{M} (u_j - \bar{u})(v_j - \bar{v})}{\sqrt{\sum_{j=1}^{M} (u_j - \bar{u})^2} \sqrt{\sum_{j=1}^{M} (v_j - \bar{v})^2}}
$$

Where $\bar{u}$ and $\bar{v}$ are the means of $u$ and $v$ respectively.

**Parameters:**
- **Patch Size ($S$):** Hyper-parameter for patch dimensions. Typical value: $1/100$ of image resolution (e.g., $64 \times 64$ for MipNeRF360, $32 \times 32$ for others) [S1].
- **Loss Weight:** Weight for the local Pearson loss. Typical value: $0.15$ [S1]. The global Pearson loss weight is also typically $0.15$ [S1].

**Pitfalls:**
- **Depth Distortion:** Using standard alpha-blending for depth supervision can distort actual depth by manipulating the opacity of floaters. Softmax-scaling mitigates this by ensuring depth variations align correctly, as the model consolidates low-weight Gaussians into a single Gaussian point at the correct depth by the end of training [S1].
- **Patch Edges:** Without a global constraint, depth discontinuities may appear at patch boundaries. A global Pearson depth loss must be applied alongside the local loss [S1].

## Operating characteristics
- **Time Complexity:** $O(N \cdot S^2)$ per iteration for patch extraction and correlation computation, where $N$ is the number of patches and $S$ is the patch size.
- **Memory Complexity:** $O(S^2)$ for storing patch data during correlation computation.
- **Strengths:** Avoids the need for explicit scale/shift alignment parameters, which can be inconsistent across pixels and introduce distortion. Encourages local structural consistency and relative depth ordering.
- **Limitations:** Does not enforce absolute metric depth accuracy directly, relying on the prior's relative structure. Requires careful tuning of patch size and loss weights.
- **When to use:** When monocular depth priors are available and the goal is to improve structural consistency in novel view synthesis, particularly in few-shot settings where global alignment is unreliable.
- **When not to use:** When absolute metric depth is required without a reliable prior, or when computational resources are extremely limited and patch-based sampling adds overhead.

## Background
The Patch-based Depth Correlation Loss was introduced in the context of SparseGS, a method for sparse view synthesis using 3D Gaussian Splatting [S1]. It addresses the issue of floaters and depth inconsistencies in few-shot novel view synthesis by leveraging monocular depth priors. The method builds on the intuition that relative depth ordering is more robust than absolute values, similar to depth ranking losses, but extends this to entire patches for stronger local structural learning [S1].

## Sources
[S1] Sparsegs: Sparse View Synthesis Using 3D Gaussian Splatting, https://doi.org/10.1109/3dv66043.2025.00100, fetched 2026-10-26
