# Neural Pre-integrated Emitters (NEF)
Also called: Neural Pre-integrated Emitter Field, GlossyNEF

## Overview
Neural Pre-integrated Emitters (NEF) learn a neural field that represents the full pre-integrated direct illumination around an emissive object, allowing runtime shading via a single network evaluation instead of geometric integration or sampling [S1, unofficial]. Unlike analytical methods or Linearly Transformed Cosines (LTC) which scale linearly with emitter vertex count, NEF runtime cost is independent of emitter complexity [S1, unofficial]. It builds on [Standard Multi-resolution Hash Encoding](13152-standard-multi-resolution-hash-encoding.md) to encode spatial and directional inputs into a compact feature vector processed by shallow MLPs [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** World-space shading point $\mathbf{x}$, surface normal $\mathbf{n}$, outgoing view direction $\boldsymbol{\omega}_o$, and optionally roughness $\alpha \in [0.1, 1]$ and time $t$ for animated emitters [S1, unofficial].
*   **Outputs:** Outgoing radiance $\mathbf{L}_o$ pre-integrated over the emitter's surface [S1, unofficial].

**Data Structures:**
The method uses two shallow Multi-Layer Perceptrons (MLPs), each with 4 layers and width 128, using ReLU activations [S1, unofficial]:
1.  **Diffuse Head ($f_{d\theta}$):** Takes encoded position $\tilde{\mathbf{x}}$ and normal $\tilde{\mathbf{n}}$. Outputs diffuse radiance contribution [S1, unofficial].
2.  **Glossy Head ($f_{g\theta}$):** Takes encoded position $\tilde{\mathbf{x}}$, normal $\tilde{\mathbf{n}}$, reflected direction $\mathbf{r}(\tilde{\boldsymbol{\omega}}_o, \tilde{\mathbf{n}})$, and roughness $\alpha$. Outputs glossy radiance contribution [S1, unofficial].

Inputs are normalized to $[0, 1]$ before encoding [S1, unofficial]. The encoding uses the learned multiresolution hash encoding described in the parent article [S1, unofficial].

**Steps:**
1.  **Transform to Emitter-Local Space:** Given the emitter's world transform $\mathbf{M}$, compute the inverse transform $\mathbf{M}^{-1}$. Transform the world-space position $\mathbf{x}$ by $\mathbf{M}^{-1}$ to get $\tilde{\mathbf{x}}$. Transform the normal $\mathbf{n}$ and outgoing direction $\boldsymbol{\omega}_o$ by the rotational part of $\mathbf{M}^{-1}$ to get $\tilde{\mathbf{n}}$ and $\tilde{\boldsymbol{\omega}}_o$ [S1, unofficial].
2.  **Compute Reflected Direction:** Calculate the reflection vector $\mathbf{r}(\tilde{\boldsymbol{\omega}}_o, \tilde{\mathbf{n}})$ [S1, unofficial].
3.  **Encode Inputs:** Apply the multiresolution hash encoding to $\tilde{\mathbf{x}}$ and $\mathbf{r}(\tilde{\boldsymbol{\omega}}_o, \tilde{\mathbf{n}})$ [S1, unofficial].
4.  **Evaluate Heads:**
    *   Compute diffuse contribution: $\mathbf{L}_d = f_{d\theta}(\text{encode}(\tilde{\mathbf{x}}, \tilde{\mathbf{n}}))$ [S1, unofficial].
    *   Compute glossy contribution: $\mathbf{L}_g = f_{g\theta}(\text{encode}(\tilde{\mathbf{x}}, \tilde{\mathbf{n}}, \mathbf{r}(\tilde{\boldsymbol{\omega}}_o, \tilde{\mathbf{n}}), \alpha))$ [S1, unofficial].
    *   Combine with albedo textures $\rho_d(\mathbf{x})$ and $\rho_g(\mathbf{x})$ factored out to retain high-frequency details: $\mathbf{L}_o = \rho_d(\mathbf{x}) \mathbf{L}_d + \rho_g(\mathbf{x}) \mathbf{L}_g$ [S1, unofficial].
5.  **Handle Boundary/Fall-off:** If $\tilde{\mathbf{x}}$ is outside the emitter-space bounding box, clamp the query to the nearest boundary point $\tilde{\mathbf{x}}_{boundary}$. Scale the result by $(d_{boundary} / d_{actual})^2$, where $d_{boundary}$ is the distance from the emitter center to the clamped point and $d_{actual}$ is the true distance from the emitter center to $\mathbf{x}$ [S1, unofficial].

**Emitter-Space Definition:**
The emitter-space is an axis-aligned bounding region in the local frame, sized to extend $20\times$ the emitter’s largest extent on each axis [S1, unofficial]. This region captures the significant direct illumination; beyond this range, inverse-square fall-off is applied [S1, unofficial].

**Training:**
*   **Sample Generation:** Queries are drawn uniformly from the emitter-space [S1, unofficial].
    *   For the glossy head: sample $\tilde{\mathbf{x}}$ uniformly in the local bbox, $\tilde{\mathbf{n}}$ such that the emitter is visible, $\tilde{\boldsymbol{\omega}}_o$ uniformly on the upper hemisphere of $\tilde{\mathbf{n}}$, and $\alpha$ uniformly in $[0.1, 1]$ [S1, unofficial].
    *   For the diffuse head: same procedure restricted to $(\tilde{\mathbf{x}}, \tilde{\mathbf{n}})$ [S1, unofficial].
*   **Target Generation:** Target radiance is generated via offline path tracing with 64 samples per estimate [S1, unofficial]. For self-occluding emitters, full path tracing inside the emitter assembly is used to capture self-occlusion and interreflections [S1, unofficial].
*   **Loss:** Minimize relative L2 loss between network prediction and path-traced target [S1, unofficial].
*   **Optimizer:** Adam with learning rate $10^{-3}$, batch size $2^{16}$, and cosine annealing decay [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def nef_shade(x_world, n_world, w_o_world, M, alpha, rho_d, rho_g):
    # 1. Transform to emitter-local space
    M_inv = inverse(M)
    R_inv = rotation_part(M_inv)
    
    x_local = M_inv @ x_world
    n_local = normalize(R_inv @ n_world)
    w_o_local = normalize(R_inv @ w_o_world)
    
    # 2. Compute reflected direction
    r_local = reflect(w_o_local, n_local)
    
    # 3. Check boundary and apply fall-off if necessary
    bbox_min, bbox_max = get_emitter_bbox() # Defined in emitter-space
    if not is_inside(x_local, bbox_min, bbox_max):
        x_clamped = clamp(x_local, bbox_min, bbox_max)
        d_boundary = length(x_clamped)
        d_actual = length(x_local)
        scale_factor = (d_boundary / d_actual) ** 2
    else:
        x_clamped = x_local
        scale_factor = 1.0
        
    # 4. Encode inputs using Multi-resolution Hash Encoding
    # Note: Inputs normalized to [0, 1] before encoding
    enc_pos = hash_encode(normalize(x_clamped))
    enc_dir = hash_encode(normalize(r_local))
    
    # 5. Evaluate Diffuse Head
    # Input: encoded position and normal
    # Architecture: 4-layer MLP, width 128, ReLU
    diffuse_input = concatenate(enc_pos, hash_encode(normalize(n_local)))
    L_diffuse_raw = diffuse_mlp(diffuse_input)
    
    # 6. Evaluate Glossy Head
    # Input: encoded position, normal, reflected direction, roughness alpha
    # Architecture: 4-layer MLP, width 128, ReLU
    glossy_input = concatenate(enc_pos, hash_encode(normalize(n_local)), enc_dir, [alpha])
    L_glossy_raw = glossy_mlp(glossy_input)
    
    # 7. Combine with albedo textures
    L_diffuse = rho_d * L_diffuse_raw
    L_glossy = rho_g * L_glossy_raw
    
    # 8. Final Radiance
    L_out = scale_factor * (L_diffuse + L_glossy)
    return L_out
```

## Operating characteristics
*   **Runtime Cost:** Independent of emitter vertex count. At 1920x1080 on an RTX 4090, runtime is 1.2ms for a Quad (4 verts) and 1.2ms for a Dragon mesh (436k verts) [S1, unofficial]. Analytical methods take 0.03ms for Quad but 1951.4ms for Dragon [S1, unofficial].
*   **Training Time:** Ranges from 1 minute (diffuse receiver, planar emitters) to 3 hours (complex emissive effects) on an RTX 4090 [S1, unofficial].
*   **Strengths:** Handles complex emitter geometries, self-occlusion, and interreflections baked into the field. Supports roughness variation via input $\alpha$. Portable across instances [S1, unofficial].
*   **Limitations:** Does not inherently capture occlusions by surrounding scene geometry; shadow rays must be handled separately [S1, unofficial]. Requires offline path tracing for training data generation [S1, unofficial].
*   **When to use:** When emitter geometry is complex or dynamic, and real-time shading is required without per-pixel integration cost.
*   **When not to use:** When scene occlusion by non-emitter geometry is the primary concern and cannot be handled by simple shadow rays, or when training time is prohibitive.

## Background
NEF was introduced in "Real-time Rendering of Pre-integrated Neural Emitters" [S1, unofficial]. It extends the idea of neural fields in rendering, which amortize training cost over subsequent viewings [S1, unofficial]. It addresses the gap left by proxy methods that reduce emitter transport cost but still require receiver-side integration [S1, unofficial].

## Sources
[S1] Real-time Rendering of Pre-integrated Neural Emitters, https://arxiv.org/abs/2610.06762, fetched 2026-10-30 (unofficial)
