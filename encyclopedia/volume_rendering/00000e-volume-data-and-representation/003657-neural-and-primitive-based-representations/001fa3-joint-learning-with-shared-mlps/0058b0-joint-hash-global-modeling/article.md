# Joint-Hash Global Modeling

Also called: Pipeline II: Joint-Hash Global Modeling, QuantumGS GLOBAL MODEL, QuantumGS Joint-Hash

## Overview
Joint-Hash Global Modeling is a scalable neural rendering pipeline designed for large-scale real-world scenes, serving as the global counterpart to per-Gaussian hyper-quantum modeling [S1]. It addresses the memory and computational bottlenecks of scene-scale optimization by replacing individual Gaussian parameters with a shared, scene-wide approximation [S1]. Unlike sibling methods that encode unique properties per primitive, this approach conditions a single Variational Quantum Circuit (VQC) on concatenated hash encodings of spatial positions and viewing directions, enabling superior memory scaling while preserving quantum-enhanced view-dependent modeling [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of 3D Gaussian primitives $\mathcal{G}$, each defined by mean position $\mu$, covariance, base spherical harmonic (SH) coefficients, and base opacity. A viewing direction $d$ for each pixel.
*   **Outputs:** Refined color $c_{final}(d)$ and opacity $\alpha_{final}(d)$ for rasterization [S1].

**Data Structures:**
1.  **Spatial Hash Grid:** Encodes Gaussian mean positions $\mu$.
2.  **Directional Hash Grid:** Encodes viewing directions $d$.
3.  **Shared VQC Parameters ($\theta$):** A global set of trainable rotation angles for the quantum ansatz [S1].
4.  **Shared MLP Weights ($W_{MLP}$):** Global weights for the classical decoding network [S1].

**Steps:**
1.  **Hash Encoding:** Compute the hash encoding for the scene's Gaussian positions, denoted as $hash_{mean}(\mu)$, and for the viewing directions, denoted as $hash_{vdir}(d)$ [S1].
2.  **Feature Concatenation:** Concatenate the spatial and directional hash features to form the conditioning vector for the quantum circuit [S1].
3.  **VQC Processing:** Condition the shared VQC on the concatenated hash features. The VQC acts as a scene-wide light field approximator. The circuit applies parameterized rotations to a 3-qubit register. For the $j$-th qubit ($j=0,1,2$), the rotation is defined as:
    $$ R_j(\theta_{j,\ell}, \phi_{j,\ell}) = R_z^{(j)}(\phi_{j,\ell}) R_y^{(j)}(\theta_{j,\ell}) $$
    where $\theta_{j,\ell}$ and $\phi_{j,\ell}$ are drawn from the global parameters $\theta$ [S1].
4.  **MLP Decoding:** Process the output of the VQC through a shared Multi-Layer Perceptron (MLP) with global weights $W_{MLP}$ [S1].
5.  **Attribute Refinement:** The MLP output generates multiplicative updates ($\Delta c(d)$ and $\Delta \alpha(d)$) applied to the base SH color and base opacity [S1]. The final attributes are computed as:
    $$ c_{final}(d) = c(d) \cdot \Delta c(d) $$
    $$ \alpha_{final}(d) = \alpha(d) \cdot \Delta \alpha(d) $$
    These refined attributes are fed into a differentiable Gaussian rasterizer [S1].

**Parameters:**
*   Learning rate for spatial XYZ hash grid and shared directional encoder: $1 \cdot 10^{-3}$ [S1].
*   Learning rate for hybrid quantum network (VQC and decoding MLP): $7.5 \cdot 10^{-3}$ [S1].
*   Densification gradient threshold: $5 \cdot 10^{-4}$ (to prevent excessive primitive growth) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def joint_hash_global_modeling(gaussians, view_directions, hash_grid_pos, hash_grid_dir, theta, w_mlp):
    """
    Renders a large-scale scene using shared quantum and classical parameters.
    
    Args:
        gaussians: List of Gaussian primitives with base attributes (mu, base_c, base_alpha).
        view_directions: Array of viewing directions for the current frame.
        hash_grid_pos: Spatial hash grid function.
        hash_grid_dir: Directional hash grid function.
        theta: Global VQC parameters.
        w_mlp: Global MLP weights.
        
    Returns:
        final_colors: Rendered color buffer.
        final_opacities: Rendered opacity buffer.
    """
    # Step 1: Compute hash encodings
    # hash_mean(mu) encodes the spatial position of the Gaussian
    # hash_vdir(d) encodes the viewing direction
    hash_pos = hash_grid_pos(gaussians.mu)
    hash_dir = hash_grid_dir(view_directions)
    
    # Step 2: Concatenate features
    # The VQC is conditioned on the combined spatial and directional information
    combined_features = concatenate(hash_pos, hash_dir)
    
    # Step 3: Process through Shared VQC
    # The VQC applies rotations based on global theta
    # For a 3-qubit register, the circuit applies R_j(theta_j, phi_j)
    # This step simulates the quantum ansatz U(theta; L)
    vqc_output = apply_vqc(combined_features, theta)
    
    # Step 4: Process through Shared MLP
    # Decode the quantum output using global MLP weights
    modulation_factors = apply_mlp(vqc_output, w_mlp)
    
    # Step 5: Refine attributes
    # Apply multiplicative updates to base SH color and opacity
    # c_final(d) = c(d) * Delta_c(d)
    # alpha_final(d) = alpha(d) * Delta_alpha(d)
    delta_c, delta_alpha = split_modulation(modulation_factors)
    
    final_colors = gaussians.base_c * delta_c
    final_opacities = gaussians.base_alpha * delta_alpha
    
    # Step 6: Rasterize
    # Feed refined attributes into differentiable Gaussian rasterizer
    rendered_image = rasterize(gaussians, final_colors, final_opacities)
    
    return rendered_image
```

## Subtopics

### Per-Gaussian Hyper-Quantum Modeling
A pipeline where a hypernetwork $H$ generates unique VQC parameters and MLP weights for each Gaussian based on its hashed position, maximizing local precision [S1]. It differs from Joint-Hash Global Modeling by using per-primitive parameters rather than a shared scene-wide model, making it suitable for synthetic scenes prioritizing high PSNR over scalability [S1]. On the NeRF Synthetic dataset, it achieves PSNR 33.98 and SSIM 0.970, outperforming standard 3DGS (PSNR 33.30) and VDGS (PSNR 33.37) [S1]. The learning rate for the spatial XYZ hash grid and hypernetwork is $5 \cdot 10^{-5}$ [S1].

## Operating characteristics
*   **Time and Memory Complexity:** The method offers superior memory scaling compared to per-Gaussian approaches because it uses a single shared VQC and MLP for the entire scene, rather than storing unique parameters for each primitive [S1].
*   **Measured Performance:** On the Deep Blending dataset, Pipeline II achieves PSNR 30.15, SSIM 0.916, LPIPS 0.163, and FPS 10.76 [S1]. On Mip-NeRF 360, it achieves PSNR 27.27, SSIM 0.793, LPIPS 0.244, and FPS 10.78 [S1].
*   **Strengths and Limitations:** It excels in large-scale real-world scenes (e.g., Tanks and Temples, Deep Blending) where memory efficiency and generalization are critical [S1]. The efficient 3-qubit VQC design mitigates the computational costs typically associated with quantum mechanics simulation, maintaining real-time rendering speeds despite the overhead [S1].
*   **When to use:** Use for large-scale real-world scenes requiring scalable memory usage and robust view-dependent modeling [S1]. Avoid for object-centric synthetic scenes where maximum local precision and highest PSNR are required, as the per-Gaussian variant is superior in that context [S1].

## Background
Joint-Hash Global Modeling is part of the QuantumGS framework, introduced in the paper "QuantumGS: Quantum Encoding Framework for Gaussian Splatting" [S1]. It was developed to balance the expressivity of per-Gaussian modeling with the scalability required for large-scale environments [S1]. The method leverages hybrid quantum-classical neural networks, specifically Variational Quantum Circuits (VQCs), to approximate light fields efficiently [S1].

## Sources
[S1] QuantumGS: Quantum Encoding Framework for Gaussian Splatting, https://arxiv.org/abs/2602.05047v1, fetched 2026-10-26
