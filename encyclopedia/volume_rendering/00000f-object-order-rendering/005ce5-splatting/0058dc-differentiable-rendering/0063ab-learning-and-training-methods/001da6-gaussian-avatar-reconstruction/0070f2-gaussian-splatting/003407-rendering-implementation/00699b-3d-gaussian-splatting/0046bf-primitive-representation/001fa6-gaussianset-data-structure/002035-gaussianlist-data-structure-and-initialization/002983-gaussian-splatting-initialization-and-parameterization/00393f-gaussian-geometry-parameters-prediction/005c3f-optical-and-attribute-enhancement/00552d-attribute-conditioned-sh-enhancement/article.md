# Attribute-Conditioned SH Enhancement

Also called: Gaussian-aware SH offset prediction, Geometry-Conditioned Residual Module, SH Enhancement module, Static SH offset baking

## Overview
The Attribute-Conditioned SH Enhancement module recovers high-frequency visual details lost when compressing radiance into low-order Spherical Harmonics (SH), specifically first-order SH [S1][S2, unofficial]. Unlike methods that introduce view-dependent neural networks at render time (e.g., Mobile-GS), this method predicts view-independent offsets based on intrinsic Gaussian attributes and statically bakes them into the Gaussian parameters before inference [S1]. This approach eliminates additional computational overhead during rendering while maintaining high reconstruction fidelity [S1][S2, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** The set of $N$ 3D Gaussians, each defined by position $\mu$, scale $s$, rotation $r$, opacity $\alpha$, and base first-order SH coefficients $c'$.
*   **Output:** Enhanced SH coefficients $c_{out}$ for each Gaussian, where the high-frequency residuals are encoded into the static parameters.

**Data Structures**
The module $\Phi$ is a Multi-Layer Perceptron (MLP).
*   **Architecture:** 4 hidden layers with ReLU activation.
*   **Hidden Neurons:** The layer sizes are $(128, 64, 32, 12)$ respectively [S1].
*   **Input Vector:** For each Gaussian $i$, the input to $\Phi$ is the concatenation of:
    1.  Normalized first-order SH coefficients $\tilde{c}'_i$ [S2, unofficial].
    2.  Opacity $\alpha_i$ (denoted as $o_i$ in the formula) [S2, unofficial].
    3.  Normalized scales $\hat{s}_i$ [S2, unofficial].
    4.  Spatial position $\mu_i$ [S2, unofficial].
    5.  Rotation quaternion $r_i$ [S2, unofficial].
*   **Output Vector:** The residual offset $\Delta c_i$ for the SH coefficients [S2, unofficial].

**Steps**
1.  **Training Phase (Offset Prediction):**
    1.  Initialize the final linear layer of $\Phi$ with strict zero initialization [S2, unofficial]. This ensures the model starts as standard 3D Gaussian Splatting (3DGS) and learns residuals gradually [S2, unofficial].
    2.  During optimization, compute the residual offset $\Delta c_i$ for each Gaussian $i$ using the MLP $\Phi$:
        $$ \Delta c_i = \Phi([\mu_i, \hat{s}_i, r_i, o_i, \tilde{c}'_i]) $$
        where $[\cdot]$ denotes concatenation [S2, unofficial].
    3.  The enhanced coefficients are calculated as:
        $$ c_{out, i} = c'_i + \Delta c_i $$
        [S2, unofficial].
    4.  *Note on Training Curriculum:* In some implementations (e.g., Flux-GS), training begins with full third-order SH for the first 3k iterations to establish high-frequency representation, followed by aggregation to first-order SH via a Monte Carlo Specular Energy Aggregator, after which the enhancement module is active [S2, unofficial].

2.  **Inference Phase (Static Baking):**
    1.  Because $\Phi$ depends only on intrinsic Gaussian attributes and not on the camera viewing direction, the offset $\Delta c_i$ is view-independent [S2, unofficial].
    2.  Pre-compute $\Delta c_i$ for all Gaussians once before rendering [S2, unofficial].
    3.  Add $\Delta c_i$ to the base coefficients $c'_i$ and store the result $c_{out, i}$ directly into the Gaussian's SH attribute storage [S1][S2, unofficial].
    4.  Render using the standard 3DGS rasterization pipeline with the baked coefficients, incurring zero additional computational overhead [S1][S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def attribute_conditioned_sh_enhancement(gaussians, mlp_phi):
    """
    Predicts and bakes SH offsets for a set of Gaussians.
    
    Args:
        gaussians: List of Gaussian objects. Each has attributes:
                   .mu (position), .s (scale), .r (rotation quaternion),
                   .alpha (opacity), .c_prime (first-order SH coefficients).
        mlp_phi: The trained MLP model with layers [128, 64, 32, 12] and ReLU.
                 The final layer is zero-initialized.
    
    Returns:
        Updated Gaussians with baked SH coefficients.
    """
    for g in gaussians:
        # 1. Normalize inputs as required by the training setup
        # Note: Exact normalization formulas for s and c' are not specified in sources.
        # [G] Assuming standard normalization to [0,1] or similar based on context.
        norm_scale = normalize_scale(g.s)
        norm_sh = normalize_sh(g.c_prime)
        
        # 2. Construct input vector for MLP
        # Concatenation order: [mu, s_hat, r, o, c_tilde_prime]
        input_vector = concatenate(
            g.mu, 
            norm_scale, 
            g.r, 
            g.alpha, 
            norm_sh
        )
        
        # 3. Predict residual offset
        delta_c = mlp_phi.forward(input_vector)
        
        # 4. Compute enhanced coefficients
        c_out = add(g.c_prime, delta_c)
        
        # 5. Bake into Gaussian parameters (replace c_prime with c_out)
        g.c_prime = c_out
        
    return gaussians
```

**Parameters**
*   **MLP Layers:** 4 layers [S1].
*   **Hidden Neurons:** 128, 64, 32, 12 [S1].
*   **Activation:** ReLU [S1].
*   **Initialization:** Final linear layer strictly zero-initialized [S2, unofficial].

**Edge Cases and Pitfalls**
*   **Optimization Stability:** If the final layer of $\Phi$ is not zero-initialized, the model may diverge at the start of training because it attempts to predict large residuals before the base representation is stable [S2, unofficial].
*   **Inference Overhead:** Failing to bake the offsets statically and instead computing them per-view during rendering will reintroduce the computational overhead that this method aims to eliminate [S1][S2, unofficial].
*   **Degenerate Input:** The method relies on intrinsic attributes. If Gaussian attributes (e.g., scale) are extreme or undefined, normalization steps must be robust to prevent NaNs in the MLP [G].

## Subtopics

### Monte Carlo Specular Energy Aggregator
Compresses high-order radiance residuals into first-order SH before enhancement. It handles the initial compression that necessitates the SH Enhancement module. [Link](17107-optical-and-attribute-enhancement.md#monte-carlo-specular-energy-aggregator)

### Multi-view Alpha-based Densification and Pruning
Addresses the inefficiency of single-view densification by using multi-view consistency to prune redundant Gaussians, ensuring the compact representation required for mobile efficiency. [Link](17107-optical-and-attribute-enhancement.md#multi-view-alpha-based-densification-and-pruning)

## Operating characteristics
*   **Time Complexity:** Zero additional time complexity during rendering compared to standard 3DGS with first-order SH. The MLP is executed only once during the baking phase [S1][S2, unofficial].
*   **Memory Complexity:** Adds no memory overhead during rendering, as offsets are baked into existing SH coefficient storage. The MLP model itself is small (4 layers, ~12k parameters) and not stored per-Gaussian [S1].
*   **Performance:** On mobile hardware (e.g., RTX 4090 for training, mobile inference), Flux-GS achieves competitive PSNR (e.g., 23.27 on Mip-NeRF 360 subset) with significantly lower storage (3.5 MB) and higher FPS (139) compared to baselines like Mobile-GS or third-order SH 3DGS [S1].
*   **Strengths:** Recovers high-frequency details (specular highlights, fine textures) lost in SH compression without sacrificing rendering speed. Maintains compatibility with standard rasterization pipelines.
*   **Limitations:** The enhancement is view-independent. It cannot capture dynamic view-dependent effects that change with camera position (e.g., complex anisotropic reflections that vary significantly with angle) as effectively as a real-time view-dependent MLP, though it captures the average high-frequency structure well [S1].
*   **When to use:** Resource-constrained environments (mobile, edge devices) where high-order SH is too expensive, but visual fidelity is critical.
*   **When not to use:** Scenes requiring precise, dynamic view-dependent specularities that vary strongly with viewing angle, where a real-time neural renderer is acceptable.

## Background
The method was introduced in **Flux-GS** (Du et al., 2024) and **Mobile-4DGS** (2026) to address the trade-off between rendering efficiency and visual fidelity in 3D Gaussian Splatting on mobile devices [S1][S2, unofficial]. Traditional 3DGS uses third-order SH (81 coefficients per Gaussian), which is memory-intensive. Mobile-GS used first-order SH (3 coefficients) but added a neural view-dependent network at render time, which hindered FPS [S1]. Flux-GS proposed the Attribute-Conditioned SH Enhancement module to recover the lost detail from compression by baking view-independent residuals into the Gaussians, achieving "zero-cost" enhancement [S1]. The approach was later extended to dynamic scenes in Mobile-4DGS [S2, unofficial].

## Sources
[S1] Monte Carlo Energy Aggregation for Mobile 3D Gaussian Splatting, https://doi.org/10.1007/978-3-032-37167-6_17, fetched 2026-10-29
[S2] Mobile-4DGS: Unified Static-Dynamic Real-time Mobile Gaussian Splatting, https://arxiv.org/abs/2610.05289, fetched 2026-10-29 (unofficial)
