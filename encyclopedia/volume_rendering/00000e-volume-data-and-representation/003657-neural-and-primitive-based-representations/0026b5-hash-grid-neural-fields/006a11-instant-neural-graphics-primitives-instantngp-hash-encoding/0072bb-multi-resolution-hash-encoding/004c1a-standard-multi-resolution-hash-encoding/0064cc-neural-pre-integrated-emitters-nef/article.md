# Neural Pre-integrated Emitters (NEF)

## Overview
Neural Pre-integrated Emitters (NEF) learn a neural field that represents the full pre-integrated direct illumination around an emissive object, allowing runtime shading via a single network evaluation instead of geometric integration or sampling [S1, unofficial]. Unlike analytical methods or Linearly Transformed Cosines (LTC) which scale linearly with emitter vertex count, NEF runtime cost is independent of emitter complexity [S1, unofficial]. It builds on [Standard Multi-resolution Hash Encoding](13152-standard-multi-resolution-hash-encoding.md) to encode spatial and directional inputs into a compact feature vector processed by shallow MLPs [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** World-space shading point $\mathbf{x}$, surface normal $\mathbf{n}$, outgoing view direction $\boldsymbol{\omega}_o$, and optionally roughness $\alpha \in [0.1, 1]$ and time $t$ for animated emitters [S1, unofficial].
*   **Outputs:** Outgoing radiance $\mathbf{L}_o$ pre-integrated over the emitter's surface [S1, unofficial].

**Data Structures:**
The method uses two shallow Multi-Layer Perceptrons (MLPs), each with 4 layers and width 128, using ReLU activations [S1, unofficial].
1.  **Diffuse Head ($f_{d\theta}$):** Takes encoded position $\tilde{\mathbf{x}}$ and normal $\tilde{\mathbf{n}}$ to output the diffuse radiance contribution [S1, unofficial].
2.  **Glossy Head ($f_{g\theta}$):** Takes encoded position $\tilde{\mathbf{x}}$, normal $\tilde{\mathbf{n}}$, reflected direction $\mathbf{r}(\tilde{\boldsymbol{\omega}}_o, \tilde{\mathbf{n}})$, and roughness $\alpha$ to output the glossy radiance contribution [S1, unofficial].

All network inputs are normalized to $[0, 1]$ before encoding [S1, unofficial]. The encoding uses the learned multiresolution hash encoding described in the parent article [S1, unofficial].

**Steps:**
1.  **Transform to Emitter-Local Space:** Given the emitter's world transform $\mathbf{M}$, compute the inverse transform $\mathbf{M}^{-1}$ [S1, unofficial]. Transform the world-space position $\mathbf{x}$ by $\mathbf{M}^{-1}$ to get $\tilde{\mathbf{x}}$ [S1, unofficial]. Transform the normal $\mathbf{n}$ and outgoing direction $\boldsymbol{\omega}_o$ by the rotational part of $\mathbf{M}^{-1}$ to get $\tilde{\mathbf{n}}$ and $\tilde{\boldsymbol{\omega}}_o$ [S1, unofficial].
2.  **Compute Reflected Direction:** Calculate the reflection vector $\mathbf{r}(\tilde{\boldsymbol{\omega}}_o, \tilde{\mathbf{n}})$ [S1, unofficial].
3.  **Encode Inputs:** Apply the multiresolution hash encoding to $\tilde{\mathbf{x}}$ and $\mathbf{r}(\tilde{\boldsymbol{\omega}}_o, \tilde{\mathbf{n}})$ [S1, unofficial].
4.  **Evaluate Heads:** Compute diffuse contribution $\mathbf{L}_d = f_{d\theta}(\text{encode}(\tilde{\mathbf{x}}, \tilde{\mathbf{n}}))$ [S1, unofficial]. Compute glossy contribution $\mathbf{L}_g = f_{g\theta}(\text{encode}(\tilde{\mathbf{x}}, \tilde{\mathbf{n}}, \mathbf{r}(\tilde{\boldsymbol{\omega}}_o, \tilde{\mathbf{n}}), \alpha))$ [S1, unofficial]. Combine with albedo textures $\rho_d(\mathbf{x})$ and $\rho_g(\mathbf{x})$ factored out to retain high-frequency details: $\mathbf{L}_o = \rho_d(\mathbf{x}) \mathbf{L}_d + \rho_g(\mathbf{x}) \mathbf{L}_g$ [S1, unofficial].
5.  **Handle Boundary/Fall-off:** If $\tilde{\mathbf{x}}$ is outside the emitter-space bounding box, clamp the query to the nearest boundary point $\tilde{\mathbf{x}}_{boundary}$ [S1, unofficial]. Scale the result by $(d_{boundary} / d_{actual})^2$, where $d_{boundary}$ is the distance from the emitter center to the clamped point and $d_{actual}$ is the true distance from the emitter center to $\mathbf{x}$ [S1, unofficial].

**Emitter-Space Definition:**
The emitter-space is an axis-aligned bounding region in the local frame, sized to extend $20\times$ the emitter’s largest extent on each axis [S1, unofficial]. This region captures the significant direct illumination; beyond this range, inverse-square fall-off is applied [S1, unofficial].

**Training:**
*   **Sample Generation:** Queries are drawn uniformly from the emitter-space [S1, unofficial]. For the glossy head, sample $\tilde{\mathbf{x}}$ uniformly in the local bounding box, $\tilde{\mathbf{n}}$ such that the emitter is visible, $\tilde{\boldsymbol{\omega}}_o$ uniformly on the upper hemisphere of $\tilde{\mathbf{n}}$, and $\alpha$ uniformly in $[0.1, 1]$ [S1, unofficial]. For the diffuse head, use the same procedure restricted to $(\tilde{\mathbf{x}}, \tilde{\mathbf{n}})$ [S1, unofficial].
*   **Target Generation:** Target radiance is generated via offline path tracing with 64 samples per estimate [S1, unofficial]. For self-occluding emitters, full path tracing inside the emitter assembly is used to capture self-occlusion and interreflections [S1, unofficial].
*   **Loss:** Minimize relative L2 loss between network prediction and path-traced target [S1, unofficial].
*   **Optimizer:** Use Adam optimizer with learning rate $10^{-3}$, batch size $2^{16}$, and cosine annealing decay [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def nef_shade(x_world, n_world, w_o_world, M_inv, R_inv, alpha, rho_d, rho_g,
              hash_tables, num_levels, bbox_min, bbox_max):
    # 1. Transform to emitter-local space
    x_local = M_inv @ np.array([x_world[0], x_world[1], x_world[2], 1.0])[:3]
    n_local = R_inv @ n_world
    n_local = n_local / np.linalg.norm(n_local)
    w_o_local = R_inv @ w_o_world
    w_o_local = w_o_local / np.linalg.norm(w_o_local)

    # 2. Compute reflected direction
    r_local = w_o_local - 2.0 * np.dot(w_o_local, n_local) * n_local

    # 3. Check boundary and apply fall-off if necessary
    x_clamped = np.clip(x_local, bbox_min, bbox_max)
    d_boundary = np.linalg.norm(x_clamped)
    d_actual = np.linalg.norm(x_local)
    scale_factor = (d_boundary / d_actual) ** 2 if d_actual > 0 else 1.0

    # 4. Encode inputs using Multi-resolution Hash Encoding
    x_norm = (x_clamped - bbox_min) / (bbox_max - bbox_min)
    r_norm = (r_local + 1.0) / 2.0
    n_norm = (n_local + 1.0) / 2.0

    enc_pos = hash_encode(x_norm[0], x_norm[1], x_norm[2], hash_tables, num_levels)
    enc_dir = hash_encode(r_norm[0], r_norm[1], r_norm[2], hash_tables, num_levels)
    enc_norm = hash_encode(n_norm[0], n_norm[1], n_norm[2], hash_tables, num_levels)

    # 5. Evaluate Diffuse Head
    diffuse_input = np.concatenate([enc_pos, enc_norm])
    L_diffuse_raw = mlp_forward(diffuse_input, diffuse_weights)

    # 6. Evaluate Glossy Head
    alpha_vec = np.array([alpha])
    glossy_input = np.concatenate([enc_pos, enc_norm, enc_dir, alpha_vec])
    L_glossy_raw = mlp_forward(glossy_input, glossy_weights)

    # 7. Combine with albedo textures
    L_diffuse = rho_d * L_diffuse_raw
    L_glossy = rho_g * L_glossy_raw

    # 8. Final Radiance
    L_out = scale_factor * (L_diffuse + L_glossy)
    return L_out

def mlp_forward(x, weights):
    h = x
    for W, b in weights:
        h = np.maximum(0.0, h @ W + b)
    return h

def hash_encode(x, y, z, hash_tables, num_levels):
    # Calls [Standard Multi-resolution Hash Encoding](13152-standard-multi-resolution-hash-encoding.md)
    # Returns concatenated feature vector from all levels
    pass
```

## Operating characteristics
*   **Runtime Cost:** Independent of emitter vertex count [S1, unofficial]. At 1920x1080 on an RTX 4090, runtime is 1.2ms for a Quad (4 verts) and 1.2ms for a Dragon mesh (436k verts) [S1, unofficial]. Analytical methods take 0.03ms for Quad but 1951.4ms for Dragon [S1, unofficial].
*   **Training Time:** Ranges from 1 minute (diffuse receiver, planar emitters) to 3 hours (complex emissive effects) on an RTX 4090 [S1, unofficial].
*   **Strengths:** Handles complex emitter geometries, self-occlusion, and interreflections baked into the field [S1, unofficial]. Supports roughness variation via input $\alpha$ [S1, unofficial]. Portable across instances [S1, unofficial].
*   **Limitations:** Does not inherently capture occlusions by surrounding scene geometry; shadow rays must be handled separately [S1, unofficial]. Requires offline path tracing for training data generation [S1, unofficial].
*   **When to use:** When emitter geometry is complex or dynamic, and real-time shading is required without per-pixel integration cost [S1, unofficial].
*   **When not to use:** When scene occlusion by non-emitter geometry is the primary concern and cannot be handled by simple shadow rays, or when training time is prohibitive [S1, unofficial].

## Background
NEF was introduced in "Real-time Rendering of Pre-integrated Neural Emitters" [S1, unofficial]. It extends the idea of neural fields in rendering, which amortize training cost over subsequent viewings [S1, unofficial]. It addresses the gap left by proxy methods that reduce emitter transport cost but still require receiver-side integration [S1, unofficial].

## Sources
[S1] Real-time Rendering of Pre-integrated Neural Emitters, https://arxiv.org/abs/2610.06762, fetched 2026-10-30 (unofficial)
