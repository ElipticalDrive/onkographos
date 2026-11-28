# Standard-Codec-in-the-Loop (SCL)

Also called: Codec-in-the-Loop with STE

## Overview
Standard-Codec-in-the-Loop (SCL) is a training framework that inserts standard, non-differentiable codecs (such as JPEG, HEVC, or AV1) directly into the neural radiance field training loop [S1]. It solves the problem of optimizing scene representations under realistic compression constraints by using a straight-through estimator (STE) to bypass the codec during backpropagation [S1]. Unlike prior codec-in-the-loop methods that rely on differentiable proxies or learned entropy models, SCL allows the use of arbitrary, off-the-shelf codecs [S1]. It builds on [Straight-Through Estimator (STE) for Gradient Extension](11563-straight-through-estimator-ste-for-gradient-extension.md) by applying the STE mechanism specifically to the quantization and packing steps of feature planes and density grids.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Pre-codec feature planes $P^{ax}$ for each axis $ax \in \{xy, xz, yz\}$; pre-codec density grid $D$; camera parameters $\pi$; network weights $\phi$ [S1].
*   **Outputs:** Rendered image $I$; updated parameters $(P, D, \phi)$ [S1].

**Data Structures:**
*   $P^{ax}$: Feature planes for each spatial axis [S1].
*   $D$: Density grid [S1].
*   $Q(\cdot)$: Quantization function mapping continuous values to discrete indices [S1].
*   $\text{pack}(\cdot)$: Bit-packing function converting quantized data into a bitstream [S1].
*   $C_q(\cdot)$: The standard codec encoder, taking packed bits and returning reconstructed bits $\hat{Y}$ and bit count [S1].
*   $\text{unpack}(\cdot)$: Bit-unpacking function converting reconstructed bits back to quantized indices [S1].
*   $Q^{-1}(\cdot)$: Dequantization function mapping indices back to continuous values [S1].
*   $\text{detach}(\cdot)$: A stop-gradient operator that returns the input in the forward pass but blocks gradients in the backward pass [S1].

**Formulas:**
The STE overrides for the reconstructed feature planes $\widetilde{P}^{ax}$ and density grid $\widetilde{D}$ are defined as [S1]:

$$
\widetilde{P}^{ax} = \widehat{P}^{ax} + \left( P^{ax} - \texttt{detach}(P^{ax}) \right)
$$

$$
\widetilde{D} = \widehat{D} + \left( D - \texttt{detach}(D) \right)
$$

where $\widehat{P}^{ax}$ and $\widehat{D}$ are the values obtained after the full codec round-trip (quantize, pack, encode, decode, unpack, dequantize) [S1].

The gradient flow through these overrides is [S1]:

$$
\frac{\partial \widetilde{P}^{ax}}{\partial P^{ax}} = \mathbf{I}, \quad \frac{\partial \widetilde{D}}{\partial D} = \mathbf{I}
$$

This ensures gradients bypass the non-differentiable codec operations and flow directly to the pre-codec features $P^{ax}$ and $D$ [S1].

**Parameters:**
*   **Gradient-surrogate choice:** The method supports vanilla STE, modified STE, and hybrid STE+SPSA [S1]. Vanilla STE is reported to provide the best robustness [S1].
*   **Loss weights:** $\lambda_{\text{rec}}$ for image reconstruction loss and $\lambda_{\text{tv}}$ for total variation regularization on $P$ [S1].

**Pseudocode:**
The training step for SCL is implemented as follows [S1]:

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def scl_training_step(P, D, pi, phi, ground_truth):
    # Helper operations matching source notation
    def quantize(x): return torch.round(x * 255).int() # [G] Q(·)
    def pack(x): return x.tobytes() # [G] pack(·)
    def codec_encode(bits): return bits, len(bits) # [G] C_q(·)
    def unpack(bits): return torch.frombuffer(bits) # [G] unpack(·)
    def dequantize(x): return x.float() / 255.0 # [G] Q^{-1}(·)
    def detach(x): return x.detach() # [G] stop-gradient
    def render(P_t, D_t, pi, phi): return torch.zeros_like(ground_truth) # [G] R(·)
    def compute_loss(I, gt, P): return torch.mean(torch.abs(I - gt)) + 0.1 * torch.sum(torch.abs(P[1:] - P[:-1])) # [G] L1 + TV
    def update_parameters(P, D, phi, L): pass # [G] optimizer step

    P_tilde = {}
    for ax in ['xy', 'xz', 'yz']:
        X01 = quantize(P[ax])
        Y = pack(X01)
        Y_hat, bits = codec_encode(Y)
        P_hat_ax = dequantize(unpack(Y_hat))
        P_tilde[ax] = P_hat_ax + P[ax] - detach(P[ax])

    D_quant = quantize(D)
    D_packed = pack(D_quant)
    D_hat_packed, _ = codec_encode(D_packed)
    D_hat = dequantize(unpack(D_hat_packed))
    D_tilde = D_hat + D - detach(D)

    I = render(P_tilde, D_tilde, pi, phi)
    L = compute_loss(I, ground_truth, P)
    update_parameters(P, D, phi, L)
    return I, L
```

**Edge Cases and Pitfalls:**
*   **Precision:** The forward pass uses values numerically equal to the decoded features and density grids, but the backward pass relies on the differentiability of the pre-codec tensors [S1]. If quantization or packing introduces numerical instability, gradients may be affected, though the STE bypasses the codec's internal non-differentiability [S1].

## Subtopics

### [Caching for SCL Training](17907-caching-for-scl-training.md)
Reduces computational cost by caching decoded feature planes and density volumes, refreshing them only when necessary based on time or parameter drift.

### [Hybrid STE+SPSA Gradient Estimation](17911-hybrid-ste-spsa-gradient-estimation.md)
Combines vanilla STE with SPSA to reduce gradient bias during cache updates, at the cost of higher variance and training time.

## Operating characteristics
*   **Time Complexity:** The forward pass involves a full codec round-trip for each feature plane and the density grid [S1]. The backward pass bypasses the codec via STE [S1].
*   **Memory Complexity:** Requires storing the pre-codec features, density grid, and the codec's internal state during encoding and decoding [S1].
*   **Performance:** SCL enables training with standard codecs, which are widely available and optimized [S1]. It provides robust gradient flow compared to other surrogate methods [S1].
*   **Strengths:** No need for differentiable codec proxies; compatible with any standard codec; robust gradient estimation [S1].
*   **When to use:** When the goal is to optimize NeRFs for delivery over networks using standard compression codecs [S1].

## Background
SCL was introduced in the CATRF (Codec-Adaptive TriPlane Radiance Fields) paper [S1]. It addresses the limitations of prior codec-in-the-loop frameworks that relied on differentiable proxies or learned codecs [S1]. By using STE, SCL allows the direct insertion of standard, non-differentiable codecs into the training loop, enabling more realistic optimization for volumetric content delivery [S1].

## Sources
[S1] CATRF: Codec-Adaptive TriPlane Radiance Fields for Volumetric Content Delivery, https://doi.org/10.1109/cvprf71909.2026.00048, fetched 2026-10-30
