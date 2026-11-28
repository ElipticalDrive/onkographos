# DeltaSplat

Also called: DeltaSplat refinement module, Iterative Gaussian Refinement

## Overview
DeltaSplat is a lightweight, iterative refinement module for pose-free feed-forward 3D Gaussian Splatting (3DGS) that corrects geometric and photometric errors introduced by single-pass prediction and camera estimation uncertainty [S1, unofficial]. It distinguishes itself from prior residual refinement methods like [DeSplat](12925-desplat.md) by conditioning per-Gaussian updates on per-pixel Plücker rays and rendered depth, providing a soft geometric prior that disambiguates 2D rendering residuals into reliable 3D corrections under uncalibrated or predicted camera poses [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of context images $\{I_i\}_{i \in C}$, where $C$ is the set of context view indices.
*   **Output:** A refined set of 3D Gaussians $G^{(K)}$ after $K$ refinement iterations.

**Data Structures:**
The Gaussian set $G$ consists of $G = N_{ctx}HW$ primitives, where $N_{ctx}$ is the number of context views, and $H, W$ are image height and width [S1, unofficial]. Each Gaussian $g$ is defined by:
*   $\mu_g \in \mathbb{R}^3$: Spatial mean (position).
*   $\alpha_g \in (0, 1)$: Opacity.
*   $q_g \in S^3$: Rotation (quaternion).
*   $s_g \in \mathbb{R}^3_+$: Scale.
*   $c^{(0)}_g \in \mathbb{R}^3$: Degree-0 spherical harmonic coefficient (view-independent color) [S1, unofficial].

**Parameters:**
*   $K$: Number of refinement iterations. Typical value: $K=3$ [S1, unofficial].
*   $\lambda_{lp}$: Perceptual loss weight. Typical value: $0.1$ [S1, unofficial].
*   The refinement module $h_\phi$ weights are shared across all $K$ iterations [S1, unofficial].
*   Parameter overhead: $\sim 2.2\%$ of the backbone network parameters [S1, unofficial].

**Pitfalls and Edge Cases:**
*   **Fixed Shape:** Scales $s_g$ and rotations $q_g$ remain fixed during refinement to preserve the local structure established by the backbone; only position, opacity, and color are updated [S1, unofficial].
*   **Geometric Conditioning:** Omitting Plücker rays and depth causes positional updates $|\Delta \mu|$ to become larger and more dispersed, degrading PSNR by $\sim 1.37$ dB [S1, unofficial].
*   **Iteration Count:** Iterating beyond $K=3$ yields no further quality gains and increases latency linearly [S1, unofficial].

**Algorithm:**

1.  **Initialization:**
    Pass context images $\{I_i\}_{i \in C}$ through the feed-forward backbone $f_\theta$ to obtain initial Gaussians $G_{ff}$ and predicted camera poses $\{p_i\}_{i \in C}$ [S1, unofficial]:
    $$ G_{ff}, \{p_i\}_{i \in C} = f_\theta(\{I_i\}_{i \in C}) $$
    Set the initial state for refinement: $G^{(0)} = G_{ff}$ [S1, unofficial].

2.  **Iterative Refinement (for $k = 0$ to $K-1$):**
    a. **Render:** Render the current Gaussian set $G^{(k)}$ at the predicted context camera poses $\{p_i\}_{i \in C}$ using a differentiable rasterizer $\mathcal{R}$ to produce rendered images $\hat{I}^{(k)}_i$ and rendered depths $d^{(k)}_i$ [S1, unofficial]:
    $$ \hat{I}^{(k)}_i = \mathcal{R}(G^{(k)}; p_i), \quad d^{(k)}_i = \text{Depth}(\mathcal{R}(G^{(k)}; p_i)) $$
    
    b. **Compute Residual:** Calculate the pixel-wise rendering residual $r^{(k)}_i$ between the input context image $I_i$ and the rendered image $\hat{I}^{(k)}_i$ [S1, unofficial]:
    $$ r^{(k)}_i = I_i - \hat{I}^{(k)}_i $$
    
    c. **Encode Geometry and Residual:** Construct per-pixel Plücker ray maps $\ell_i \in \mathbb{R}^{H \times W \times 6}$ from the camera poses $p_i$. Encode the residual $r^{(k)}_i$, rendered image $\hat{I}^{(k)}_i$, rendered depth $d^{(k)}_i$, and Plücker rays $\ell_i$ using a dual-branch convolutional mixer $E$ to produce fused features $F$ [S1, unofficial]. The dual-branch design gathers broad structural context at coarse resolutions before full-resolution processing [S1, unofficial].
    
    d. **Decode Updates:** Decode the fused features $F$ using per-attribute MLP heads to predict the update $\Delta^{(k)}_g \in \mathbb{R}^7$ for each Gaussian $g \in G^{(k)}$ [S1, unofficial]:
    $$ \Delta^{(k)}_g = (\Delta \mu_g, \Delta \alpha_g, \Delta c^{(0)}_g) = \text{Decode}(F, g) $$
    where $\Delta \mu_g \in \mathbb{R}^3$, $\Delta \alpha_g \in \mathbb{R}$, and $\Delta c^{(0)}_g \in \mathbb{R}^3$ [S1, unofficial].
    
    e. **Update Gaussians:** Apply the predicted updates to the current Gaussian state [S1, unofficial]:
    $$ G^{(k+1)} = G^{(k)} \oplus \Delta^{(k)} $$
    Specifically:
    $$ \mu^{(k+1)}_g = \mu^{(k)}_g + \Delta \mu_g $$
    $$ \alpha^{(k+1)}_g = \alpha^{(k)}_g + \Delta \alpha_g $$
    $$ c^{(0)(k+1)}_g = c^{(0)(k)}_g + \Delta c^{(0)}_g $$
    $$ q^{(k+1)}_g = q^{(k)}_g, \quad s^{(k+1)}_g = s^{(k)}_g $$

3.  **Final Output:**
    Return the refined Gaussian set $G^{(K)}$ [S1, unofficial].

**Training Objective:**
The model is trained end-to-end by unrolling $K$ iterations. The loss combines pixel-wise mean squared error and a perceptual loss (VGG backbone) over all rendered views (context and target) [S1, unofficial]:
$$ \mathcal{L} = \sum_{i \in C \cup T} \|\hat{I}^{(K)}_i - I_i\|_2^2 + \lambda_{lp} \mathcal{L}_{\text{LPIPS}}(\hat{I}^{(K)}_i, I_i) $$
Training runs for 50k steps with $K=3$ [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Refinement time grows linearly with $K$. On tested hardware, latency increases from 54 ms ($K=0$) to 266 ms ($K=6$) [S1, unofficial].
*   **Memory Complexity:** Adds $\sim 2.2\%$ parameters to the backbone [S1, unofficial].
*   **Performance:** On the DL3DV dataset (6 context views, pose-free, known intrinsics), DeltaSplat achieves 26.64 dB PSNR, 0.862 SSIM, and 0.115 LPIPS, improving the YoNoSplat backbone by 1.75 dB [S1, unofficial].
*   **Strengths:** Effective at correcting camera estimation errors; lightweight; fully feed-forward at inference.
*   **Limitations:** Performance degrades if geometric conditioning (Plücker rays/depth) is removed; quality peaks at $K=3$ and does not improve with more iterations [S1, unofficial].
*   **When to use:** Pose-free or uncalibrated camera regimes where single-pass prediction errors are significant.
*   **When not to use:** When ground-truth camera poses are available and high accuracy is required without the overhead of iterative refinement (though DeltaSplat still provides gains).

## Background
DeltaSplat was introduced to address the limitations of pose-free feed-forward 3DGS methods, where camera estimation errors propagate into the Gaussian representation [S1, unofficial]. It builds on the concept of iterative residual refinement seen in methods like [DeSplat](12925-desplat.md) but adapts it for the pose-free setting by introducing geometric conditioning via Plücker rays and rendered depth [S1, unofficial]. The method was evaluated on DL3DV, ScanNet++, and RE10K datasets [S1, unofficial].

## Sources
[S1] DeltaSplat: Iterative Gaussian Refinement for Pose-Free Feed-Forward 3D Gaussian Splatting, https://arxiv.org/abs/2610.09853, fetched 2026-10-29 (unofficial)
