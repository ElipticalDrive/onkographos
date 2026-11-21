# Standard-Codec-in-the-Loop (SCL)

Also called: Codec-in-the-Loop with STE

## Overview
Standard-Codec-in-the-Loop (SCL) is a training framework that inserts standard, non-differentiable codecs (such as JPEG, HEVC, or AV1) directly into the neural radiance field (NeRF) training loop [S1]. It solves the problem of optimizing scene representations under realistic compression constraints by using a straight-through estimator (STE) to bypass the codec during backpropagation [S1]. Unlike prior codec-in-the-loop methods that rely on differentiable proxies or learned entropy models, SCL allows the use of arbitrary, off-the-shelf codecs [S1]. It builds on [Straight-Through Estimator (STE) for Gradient Extension](11563-straight-through-estimator-ste-for-gradient-extension.md) by applying the STE mechanism specifically to the quantization and packing steps of feature planes and density grids.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Pre-codec feature planes $P^{ax}$ for each axis $ax \in \{xy, xz, yz\}$; pre-codec density grid $D$; camera parameters $\pi$; network weights $\phi$.
*   **Outputs:** Rendered image $I$; updated parameters $(P, D, \phi)$.

**Data Structures:**
*   $P^{ax}$: Feature planes for each spatial axis.
*   $D$: Density grid.
*   $Q(\cdot)$: Quantization function mapping continuous values to discrete indices.
*   $\text{pack}(\cdot)$: Bit-packing function converting quantized data into a bitstream.
*   $C_q(\cdot)$: The standard codec encoder, taking packed bits and returning reconstructed bits $\hat{Y}$ and bit count.
*   $\text{unpack}(\cdot)$: Bit-unpacking function converting reconstructed bits back to quantized indices.
*   $Q^{-1}(\cdot)$: Dequantization function mapping indices back to continuous values.
*   $\text{detach}(\cdot)$: A stop-gradient operator that returns the input in the forward pass but blocks gradients in the backward pass.

**Formulas:**
The STE overrides for the reconstructed feature planes $\widetilde{P}^{ax}$ and density grid $\widetilde{D}$ are defined as [S1]:

$$
\widetilde{P}^{ax} = \widehat{P}^{ax} + \left( P^{ax} - \texttt{detach}(P^{ax}) \right)
$$

$$
\widetilde{D} = \widehat{D} + \left( D - \texttt{detach}(D) \right)
$$

where $\widehat{P}^{ax}$ and $\widehat{D}$ are the values obtained after the full codec round-trip (quantize, pack, encode, decode, unpack, dequantize).

The gradient flow through these overrides is [S1]:

$$
\frac{\partial \widetilde{P}^{ax}}{\partial P^{ax}} = \mathbf{I}, \quad \frac{\partial \widetilde{D}}{\partial D} = \mathbf{I}
$$

This ensures gradients bypass the non-differentiable codec operations and flow directly to the pre-codec features $P^{ax}$ and $D$.

**Parameters:**
*   **Gradient-surrogate choice:** The method supports vanilla STE, modified STE, and hybrid STE+SPSA. Vanilla STE is reported to provide the best robustness [S1].
*   **Loss weights:** $\lambda_{\text{rec}}$ for image reconstruction loss and $\lambda_{\text{tv}}$ for total variation regularization on $P$ [S1].

**Pseudocode:**
The training step for SCL is implemented as follows [S1]:

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def scl_training_step(P, D, pi, phi):
    # 1. Process feature planes for each axis
    P_tilde = {}
    for ax in ['xy', 'xz', 'yz']:
        # (a) Quantize and pack
        X01 = quantize(P[ax])
        Y = pack(X01)
        
        # (b) Encode and decode using standard codec
        Y_hat, bits = codec_encode(Y)
        P_hat_ax = dequantize(unpack(Y_hat))
        
        # (c) Apply STE override
        # P_tilde[ax] = P_hat_ax + (P[ax] - detach(P[ax]))
        P_tilde[ax] = P_hat_ax + P[ax] - detach(P[ax])
        
    # 2. Process density
    # Quantize and mono tiling are applied to D before encoding
    # Assuming similar codec round-trip for density
    D_hat = codec_roundtrip_density(D)
    
    # Apply STE override for density
    D_tilde = D_hat + D - detach(D)
    
    # 3. Render and Update
    # Render using the STE-override values
    I = render(P_tilde, D_tilde, pi, phi)
    
    # Compute loss (e.g., L1 reconstruction + TV regularization)
    L = compute_loss(I, ground_truth, P)
    
    # Update parameters (P, D, phi) using gradients from L
    update_parameters(P, D, phi, L)
    
    return I, L
```

**Edge Cases and Pitfalls:**
*   **Precision:** The method relies on the assumption that the pre-codec values $P^{ax}$ and $D$ are differentiable. If the quantization or packing introduces numerical instability, gradients may be affected, though the STE bypasses the codec's internal non-differentiability [S1].
*   **Codec Compatibility:** The method is designed to work with any standard codec that accepts packed bitstreams and returns reconstructed bitstreams. The specific implementation of `codec_encode` and `codec_roundtrip_density` must match the codec's API [S1].

## Subtopics

### [Caching for SCL Training](17907-caching-for-scl-training.md)
Reduces computational cost by caching decoded feature planes and density volumes, refreshing them only when necessary.

### [Hybrid STE+SPSA Gradient Estimation](17911-hybrid-ste-spsa-gradient-estimation.md)
Combines vanilla STE with SPSA to reduce gradient bias during cache updates, at the cost of higher variance and training time.

## Operating characteristics
*   **Time Complexity:** The forward pass involves a full codec round-trip for each feature plane and the density grid, which is computationally expensive. The backward pass is efficient as it bypasses the codec via STE [S1].
*   **Memory Complexity:** Requires storing the pre-codec features, density grid, and the codec's internal state during encoding/decoding [S1].
*   **Performance:** SCL enables training with standard codecs, which are widely available and optimized. It provides robust gradient flow compared to other surrogate methods [S1].
*   **Strengths:** No need for differentiable codec proxies; compatible with any standard codec; robust gradient estimation [S1].
*   **Limitations:** High computational cost during training due to codec invocation; potential for gradient bias if the codec's distortion is significant [S1].
*   **When to use:** When the goal is to optimize NeRFs for delivery over networks using standard compression codecs [S1].
*   **When not to use:** When real-time training is required or when differentiable compression models are available and sufficient [S1].

## Background
SCL was introduced in the CATRF (Codec-Adaptive TriPlane Radiance Fields) paper [S1]. It addresses the limitations of prior codec-in-the-loop frameworks that relied on differentiable proxies or learned codecs [S1]. By using STE, SCL allows the direct insertion of standard, non-differentiable codecs into the training loop, enabling more realistic optimization for volumetric content delivery [S1].

## Sources
[S1] CATRF: Codec-Adaptive TriPlane Radiance Fields for Volumetric Content Delivery, https://doi.org/10.1109/cvprf71909.2026.00048, fetched 2026-10-30
