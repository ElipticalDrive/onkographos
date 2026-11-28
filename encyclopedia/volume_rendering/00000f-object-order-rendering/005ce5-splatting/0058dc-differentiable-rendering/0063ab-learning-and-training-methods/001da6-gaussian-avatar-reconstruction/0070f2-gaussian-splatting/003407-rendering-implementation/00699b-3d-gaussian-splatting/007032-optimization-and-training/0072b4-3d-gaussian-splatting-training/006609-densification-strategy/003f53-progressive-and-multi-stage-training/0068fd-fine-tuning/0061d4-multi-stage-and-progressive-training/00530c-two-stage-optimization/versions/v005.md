# Two-stage Optimization

Also called: GS-IR Training Process

## Overview
Two-stage optimization is a training strategy that decouples the learning of stable geometric/appearance parameters from the optimization of complex attributes (such as material properties or compressed latent codes) [S1][S2, unofficial]. It solves the instability caused by simultaneously optimizing high-dimensional generative pipelines and compact, quantized, or physically-based representations [S2, unofficial]. It sets itself apart from single-stage training by preventing the gradient conflicts between generation stability and compression/physical accuracy, and differs from progressive training by using distinct, non-overlapping parameter sets for each stage rather than temporal data splitting [S1][S2, unofficial].

## How it works
The method consists of two sequential phases. Stage 1 stabilizes the base representation (geometry and appearance) without the constraints of the final target domain (e.g., codec or PBR). Stage 2 fixes these base parameters and optimizes the specific attributes (materials or latent codes) using a rate-distortion or decomposition loss [S1][S2, unofficial].

### Inputs and Outputs
*   **Inputs:** Context views $C$ (images with poses), target view $Y_t$ (for supervision), and hyperparameters ($\lambda, \gamma, \beta, \lambda_{n-TV}, \lambda_M, \lambda_E$).
*   **Outputs:** Optimized Gaussian primitives $\hat{G}$ (or their latent representation) and associated attributes (material $M$, lighting $E$, or codec weights).

### Data Structures
The specific data structures depend on the variant (GS-IR vs. CodecSplat), but generally include:
1.  **Gaussian Set $G$:** Defined by position $p$, scale $s$, rotation $q$, opacity $\alpha$, and color/SH coefficients $A$. In GS-IR, this also includes normal $n$ [S1]. In CodecSplat, this includes features $F_{mv}$ and depth $D_{enc}$ [S2, unofficial].
2.  **Material/Lighting Attributes (GS-IR):** Albedo $a$, roughness $\rho$, metallicity $m$, and lighting coefficients $E$ [S1].
3.  **Codec Modules (CodecSplat):** Feature encoder, entropy bottleneck, and post-codec refiner [S2, unofficial].

### Numbered Steps

**Stage 1: Base Optimization**
1.  **Initialize:** Load context views $C$. Initialize Gaussian parameters $T$ (geometry: $p, s, q$) and appearance $A$ (color/SH). For GS-IR, also initialize normals $n$ [S1]. For CodecSplat, initialize the feed-forward backbone $MVEnc$ and Gaussian heads [S2, unofficial].
2.  **Forward Pass (GS-IR):** Render the scene to obtain predicted color $\hat{I}$ [S1].
3.  **Forward Pass (CodecSplat):**
    *   Extract features: $(F_{mv}, D_{enc}, P_{depth}) = MVEnc(C)$ [S2, unofficial].
    *   Generate refined feature: $F_g = FeatHead(I_{1:N}, F_{mv}, P_{depth})$ [S2, unofficial].
    *   Predict codec-free Gaussian set $G_{cf}$ from $F_g$ and $D_{enc}$ [S2, unofficial].
    *   Render target view: $\hat{Y}_{tcf} = Render(G_{cf}, K_t, T_t)$ [S2, unofficial].
4.  **Compute Loss:**
    *   **GS-IR:** $L_{init} = L_c + L_n$ [S1].
        *   $L_c$: L1 color reconstruction loss [S1].
        *   $L_n = L_{n-p} + \lambda_{n-TV} TV_{normal}$ [S1].
        *   $TV_{normal}$: Total variation of normals [S1].
    *   **CodecSplat:** $L_{stage1} = D_{render}(\hat{Y}_{tcf}, Y_t)$ [S2, unofficial].
5.  **Update:** Minimize $L_{init}$ or $L_{stage1}$ using Adam optimizer to update $T$ and $A$ (and $n$ for GS-IR) for 30,000 iterations [S1]. For CodecSplat, update backbone modules [S2, unofficial].

**Stage 2: Attribute/Codec Optimization**
1.  **Freeze:** Fix $T$ and $A$ (and $n$ for GS-IR) [S1]. For CodecSplat, fix the backbone modules trained in Stage 1 [S2, unofficial].
2.  **Initialize Attributes:**
    *   **GS-IR:** Initialize material $M=\{a, \rho, m\}$ and lighting $E$ [S1].
    *   **CodecSplat:** Enable the feature codec and post-codec refiner [S2, unofficial].
3.  **Forward Pass (GS-IR):**
    *   Render with PBR model using $M$ and $E$ [S1].
    *   Compute predicted shading $\hat{I}_{shade}$ [S1].
4.  **Forward Pass (CodecSplat):**
    *   Encode features to latent code.
    *   Decode to get Gaussian set $\hat{G}$ [S2, unofficial].
    *   Render target view: $\hat{Y}_t = Render(\hat{G}, K_t, T_t)$ [S2, unofficial].
    *   Compute codec-side depth loss:
        *   Calculate decoder-side normalized inverse-depth $S$ [S2, unofficial].
        *   Calculate encoder-side normalized inverse-depth $S_{enc}$:
            $$ S_{enc,i}(p) = \frac{D_{enc,i}(p)^{-1} - f_i^{-1}}{n_i^{-1} - f_i^{-1}} $$
            where $n_i, f_i$ are near/far bounds of context view $i$ [S2, unofficial].
        *   $D_{codec-depth} = \| S - S_{enc} \|_1$ [S2, unofficial].
5.  **Compute Loss:**
    *   **GS-IR:** $L_d = L_{shade} + \lambda_M TV_{mat} + \lambda_E TV_{light}$ [S1].
        *   $L_{shade} = I - \hat{I}_{shade}(M, E, V_{illu})$ [S1].
        *   $TV_{mat}$: Total variation of material map $\hat{M}$ [S1].
        *   $TV_{light}$: Total variation of lighting $\hat{E}$ [S1].
    *   **CodecSplat:**
        $$ L_{stage2}(\lambda) = \frac{\lambda}{\lambda_{max}} \left[ D_{render}(\hat{Y}_t, Y_t) + \gamma D_{codec-depth} \right] + \beta R $$
        where $R$ is the estimated rate, $\gamma=0.1$, $\lambda_{max}=1024$ [S2, unofficial].
6.  **Update:**
    *   **GS-IR:** Minimize $L_d$ to optimize $M$ for 10,000 iterations [S1].
    *   **CodecSplat:** Sample $\lambda \in [16, 1024]$ during training. Optimize codec-related modules [S2, unofficial].

### Parameters
*   **GS-IR:**
    *   Optimizer: Adam [S1].
    *   Iterations: 30,000 (Stage 1), 10,000 (Stage 2) [S1].
    *   $\lambda_{n-TV}$: 5.0 [S1].
    *   $\lambda_M$: 1.0 [S1].
    *   $\lambda_E$: 0.01 [S1].
    *   Hardware: Single NVIDIA V100 [S1].
*   **CodecSplat:**
    *   Optimizer: AdamW, LR $10^{-4}$, weight decay 0.01 [S2, unofficial].
    *   $\gamma$: 0.1 [S2, unofficial].
    *   $\beta$: $10^{-4}$ (DL3DV), $4 \times 10^{-4}$ (RE10K) [S2, unofficial].
    *   $\lambda$: Sampled from $[16, 1024]$ during training; fixed at test time [S2, unofficial].
    *   Hardware: Four NVIDIA RTX 5880 48GB GPUs [S2, unofficial].

### Edge Cases and Pitfalls
*   **Instability:** Directly training with a quantized bottleneck is unstable; the two-stage approach is required to prevent gradient conflicts [S2, unofficial].
*   **Rate Selection:** $\lambda$ must be sampled during training to cover the rate-distortion curve, but fixed at inference for a specific bitrate [S2, unofficial].
*   **Depth Normalization:** The codec-side depth loss requires careful normalization using near/far bounds to ensure $S$ and $S_{enc}$ are comparable [S2, unofficial].

## Subtopics

### [Two-stage training procedure for Relightable 3D Gaussians](15262-two-stage-training-procedure-for-relightable-3d-gaussians.md)
Optimizes geometry/appearance then material/lighting for inverse rendering. Uses normal gradient-based densification and PBR losses.

### [Two-stage training procedure for Relightable 3D Gaussians](15262-two-stage-training-procedure-for-relightable-3d-gaussians.md)
(implied by S2): Optimizes geometry/appearance then latent codes for compression. Uses rate-distortion loss and codec-side depth supervision.

## Operating characteristics
*   **Time Complexity:** Stage 1 typically dominates due to longer iteration counts (30k vs 10k) and full backbone training [S1][S2, unofficial].
*   **Memory Complexity:** Stage 1 requires storing full backbone gradients. Stage 2 may require additional memory for codec buffers or PBR shading computations [S1][S2, unofficial].
*   **Performance:** GS-IR achieves high-fidelity inverse rendering on a single V100 [S1]. CodecSplat achieves KiB-level compression with minimal quality loss compared to MiB-level baselines [S2, unofficial].
*   **Strengths:** Stabilizes training for complex attributes; enables compression without quality collapse [S1][S2, unofficial].
*   **Limitations:** Requires two distinct training phases; hyperparameter tuning for loss weights ($\lambda, \gamma, \beta$) is critical [S1][S2, unofficial].

## Background
The concept originates from the need to decouple stable geometric learning from complex attribute optimization in 3D Gaussian Splatting. GS-IR introduced the initial stage/decomposition stage split for inverse rendering [S1]. CodecSplat adapted this for feed-forward Gaussian compression, separating codec-free generation from codec-aware latent coding [S2, unofficial].

## Sources
[S1] GS-IR: 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.1109/cvpr52733.2024.02045, fetched 2026-10-26
[S2] CodecSplat: Ultra-Compact Latent Coding for Feed-Forward 3D Gaussian Splatting, https://doi.org/10.48550/arXiv.2605.25563, fetched 2026-10-29 (unofficial)
