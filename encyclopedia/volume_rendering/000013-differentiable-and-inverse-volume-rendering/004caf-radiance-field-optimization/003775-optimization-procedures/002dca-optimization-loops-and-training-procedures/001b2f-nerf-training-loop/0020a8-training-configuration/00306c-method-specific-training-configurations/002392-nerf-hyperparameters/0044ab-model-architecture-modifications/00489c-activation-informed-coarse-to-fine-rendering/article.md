# Activation-Informed Coarse-to-Fine Rendering

Also called: Activation-based early exit, Masked fine pass

## Overview
This method accelerates NeRF inference by skipping the expensive fine-pass MLP evaluation for rays determined to be empty or background, based on intermediate activation values from the coarse pass [S1]. It differs from standard coarse-to-fine rendering by introducing a conditional mask derived from network internals rather than relying solely on density thresholds or fixed sampling strategies [S1].

## How it works
**Inputs:** Ray $r$ generated from pixel $(x, y)$; Coarse MLP $\Theta_{\text{coarse}}$; Fine MLP $\Theta_{\text{fine}}$; Number of coarse samples $N_c$; Number of fine samples $N_f$; Layer index $\ell$ for activation analysis.
**Outputs:** Final radiance $\hat{C}$.

**Data Structures:**
*   $A^{(\ell)} \in \mathbb{R}^{N_s \times N_h}$: Activation tensor at layer $\ell$, where $N_s$ is the number of samples and $N_h$ is the number of hidden units. This tensor is recorded after the ReLU activation function has been applied [S1].
*   $V_\ell$: The "activation image," a 2D map where each pixel $(x, y)$ contains the scalar value $v_\ell$ for the corresponding ray [S1].

**Steps:**
1.  For each pixel $(x, y)$, generate ray $r$ and apply positional encoding $\gamma(\cdot)$ [S1].
2.  Pass samples along ray $r$ through $\Theta_{\text{coarse}}$ to obtain predicted densities $\sigma$ and intermediate activations $A^{(\ell)}$ [S1].
3.  Compute the scalar activation $v_\ell$ for the ray by summing all elements of $A^{(\ell)}$ and dividing by the number of samples $N_s$ and hidden units $N_h$:
    $$
    v_\ell = \frac{1}{N_s} \sum_{i=1}^{N_s} \sum_{j=1}^{N_h} A^{(\ell)}_{i,j}
    $$
    [S1]
4.  Calculate the threshold $\tau$ as the mean activation $\mu(V_\ell)$ of the activation image $V_\ell$ across all rays/pixels in the current batch or frame [S1].
5.  Identify the set of rays $P$ where $v_\ell < \tau$. These rays are treated as having no significant scene content (e.g., transparent background) [S1].
6.  For rays $r \in P$, evaluate $\Theta_{\text{fine}}$ to compute final radiance. For rays $r \notin P$, use the coarse radiance $\hat{C}_c$ as the final output [S1].

**Parameters:**
*   $\tau$: Threshold for skipping fine pass. Value: $\mu(V_\ell)$, the mean of the activation image $V_\ell$ [S1].
*   $\ell$: Layer index for activation analysis. Typical value: 1, 2, or 3. Applying to earlier layers accelerates inference more substantially [S1].

**Edge Cases and Pitfalls:**
*   The method relies on transparent backgrounds found in synthetic scenes (e.g., Blender dataset) [S1]. It fails on real-world scenes with detailed backgrounds, necessitating a density approximation method instead [S1].
*   Activations are recorded after ReLU. While this causes information loss, early experiments showed it results in more consistent histograms for feature representations [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def activation_informed_render(ray, theta_coarse, theta_fine, layer_idx):
    # 1. Generate ray and encode
    pos_enc = positional_encoding(ray)
    
    # 2. Coarse pass
    # Assume theta_coarse returns (sigma, activations)
    # activations is a list of tensors for each layer
    sigma_coarse, activations = theta_coarse(pos_enc)
    
    # 3. Compute scalar activation v_l
    # A^(l) is activations[layer_idx], shape (N_samples, N_hidden)
    A_l = activations[layer_idx]
    N_s, N_h = A_l.shape
    v_l = torch.sum(A_l) / (N_s * N_h)
    
    # 4. Determine if fine pass is needed
    # In a full implementation, tau is computed over a batch or frame.
    # Here we assume tau is available or computed globally.
    if v_l < tau:
        # 5. Fine pass for empty/background rays
        # Note: The source states fine pass is evaluated for r in P (v_l < tau)
        # and coarse radiance is used for r not in P.
        # This implies the "mask" P selects rays that need refinement.
        # However, standard logic suggests skipping fine pass for empty rays.
        # Re-reading S1: "If r in P, we evaluate Theta_fine. Otherwise, we treat the coarse radiance C_c as our final output."
        # And P = {r : v_l < tau}.
        # So low activation -> fine pass? High activation -> coarse only?
        # Let's check the logic: "empty space... we do not expect any sample with significant volumetric density."
        # "If for any of the rays the result is 0, the corresponding pixel likely belongs to the transparent background."
        # "When the value of g is high, the value of v_l is low, and vice versa."
        # g is the count of densities > threshold.
        # So low v_l -> high g? No, "value of g is high, value of v_l is low".
        # Wait, Table 1 shows "Speedup" and "% fine Rays".
        # If v_l < tau, we are in P.
        # The text says: "If r in P, we evaluate Theta_fine. Otherwise, we treat the coarse radiance C_c as our final output."
        # This seems counter-intuitive for "skipping" expensive passes. Usually you skip the fine pass for empty rays.
        # Let's re-read carefully: "we can reduce the inference time... if we perform the fine pass only if the condition v_l < mu(V_l) is met."
        # This implies: IF v_l < tau THEN fine pass. ELSE coarse only.
        # Why? Maybe low activation means the coarse model is uncertain or the ray passes through empty space where coarse is inaccurate?
        # Or maybe high activation means strong signal, so coarse is sufficient?
        # Actually, looking at the results: 40% of rays processed by fine pass.
        # If v_l < tau is the condition for fine pass, then 40% of rays have low activation.
        # Let's stick to the literal instruction from S1.
        
        # Compute fine samples
        pos_enc_fine = ... # Specific sampling strategy for fine pass
        sigma_fine, _ = theta_fine(pos_enc_fine)
        radiance = render_volume(sigma_fine, ...)
    else:
        # Use coarse radiance
        radiance = render_volume(sigma_coarse, ...)
        
    return radiance
```
*[G] The pseudocode assumes standard volume rendering integration functions `render_volume` which are not defined in the source but are standard in the field. The condition `v_l < tau` triggers the fine pass according to the source text.*

## Subtopics

### Density Approximation via Activations
Uses activation features $f_\ell$ to estimate density $\hat{d}$ directly, replacing the coarse MLP evaluation. This extends the method to real-world scenes with detailed backgrounds where the binary mask approach fails [S1].

### Layer Selection
The method is applied to the first 3 layers of the NeRF MLP. Earlier layers provide greater acceleration [S1].

## Operating characteristics
*   **Time Complexity:** Reduces the number of fine-pass evaluations. On the Blender 'lego' scene (800x800), using layer 1 achieved a 34.69% speedup with 40% of rays processed by the fine pass [S1]. Using layer 3 achieved a 55% speedup with 47% of rays processed by the fine pass [S1].
*   **Memory Complexity:** Requires storing intermediate activations $A^{(\ell)}$ during the forward pass of the coarse network.
*   **Strengths:** Significant speedup on synthetic scenes with transparent backgrounds without substantial quality loss [S1].
*   **Limitations:** Fails on real-world scenes with detailed backgrounds [S1].

## Background
Originated in the paper "Analyzing the Internals of Neural Radiance Fields" (CVPRW 2024) [S1]. The work builds on the observation that intermediate activations in coordinate-based MLPs contain structural information and density cues [S1].

## Sources
[S1] Analyzing the Internals of Neural Radiance Fields, https://doi.org/10.1109/cvprw63382.2024.00288, fetched 2026-10-26
