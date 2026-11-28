# Postoperative Surface Alignment

Also called: Height-field Gaussian Primitive Optimization

## Overview
Postoperative Surface Alignment is an optimization strategy that deforms an initial height-field Gaussian model to match postoperative MRI depth data while retaining the initial cortical texture [S1, unofficial]. It solves the problem of adapting a preoperative 3D Gaussian Splatting reconstruction to postoperative anatomical changes without requiring sparse cross-modal point correspondences [S1, unofficial]. It sets itself apart from naive depth fitting by combining a depth loss with a transition objective to prevent surface tearing and a tracked self-supervision term to preserve appearance, using alternating optimization windows to stabilize geometry and appearance updates [S1, unofficial].

## How it works
**Inputs:**
*   Initial Gaussian set $G_0$ (height-field primitives) with fixed basis centers and bandwidths [S1, unofficial].
*   Postoperative MRI depth data (target depth $D_v(\mathbf{p})$) [S1, unofficial].
*   Reference renderings from the initial reconstruction [S1, unofficial].
*   Camera parameters for the reference views [S1, unofficial].

**Outputs:**
*   Deformed Gaussian set $G$ with updated centers, rotations, tangent scales, height coefficients, and appearance coefficients [S1, unofficial].

**Data Structures:**
*   **Gaussian Primitive:** Defined by trainable parameters: centers $\mathbf{c}$, rotations $\mathbf{q}$, tangent scales $\mathbf{s}_t$, appearance coefficients $\mathbf{a}$, opacity $\alpha$, and height coefficients $h$ [S1, unofficial]. Fixed parameters: basis centers and bandwidths [S1, unofficial].
*   **Tracked Samples:** A set $\mathcal{Q} = \{(i_m, u_m, v_m, h_m, \mathbf{c}_m^0)\}$, where $i_m$ is the primitive index, $(u_m, v_m, h_m)$ are local surface coordinates, and $\mathbf{c}_m^0 \in \mathbb{R}^3$ is the initial rendered RGB color [S1, unofficial].

**Steps:**

1.  **Initialization:** Construct the initial height-field model $G_0$ from preoperative data [S1, unofficial]. Back-project valid pixels from reference renderings to local coordinates on primitives to populate $\mathcal{Q}$ with initial colors $\mathbf{c}_m^0$ [S1, unofficial].
2.  **Geometry Window:** Optimize geometry parameters to fit MRI depth [S1, unofficial].
    *   Override opacity $\alpha$ to 0.999 to prevent the model from reducing loss by making mismatched foreground patches transparent [S1, unofficial].
    *   Compute rendered camera-linear depth $\hat{D}_v(\mathbf{p})$ by reusing the explicit ray-surface intersection [S1, unofficial].
    *   Calculate Depth Loss $\mathcal{L}_{\text{depth}}$ using rendered validity mask $M_v$ and target validity mask $M_v^*$:
        $$ \mathcal{L}_{\text{depth}} = \frac{\sum_{\mathbf{p}} M_v M_v^* |\hat{D}_v(\mathbf{p}) - D_v(\mathbf{p})|}{\sum_{\mathbf{p}} M_v M_v^* + \epsilon} \quad (13) $$
    *   Calculate Transition Loss $\mathcal{L}_{\text{trans}}$ to couple adjacent charts and prevent tearing [S1, unofficial].
    *   Calculate Tracked Loss $\mathcal{L}_{\text{track}}$ by reprojecting samples from $\mathcal{Q}$ and comparing rendered colors to $\mathbf{c}_m^0$ [S1, unofficial].
    *   Minimize total geometry loss:
        $$ \mathcal{L}_{\text{depth-win}} = \mathcal{L}_{\text{depth}} + \lambda_{\text{trans}} \mathcal{L}_{\text{trans}} + \lambda_{\text{track}} \mathcal{L}_{\text{track}} \quad (16) $$
    *   Update centers, rotations, tangent scales, height coefficients, and appearance coefficients [S1, unofficial].
    *   Apply learning rate hierarchy: Centers decay from $1.6 \times 10^{-3}$ to $1.6 \times 10^{-5}$; tangent scale uses $10^{-3}$; rotation and height coefficients use $10^{-4}$; opacity uses $10^{-8}$ [S1, unofficial].
    *   Set $\lambda_{\text{trans}} = 6.0$ and $\lambda_{\text{track}} = 10.0$ [S1, unofficial].
3.  **Appearance Window:** Freeze geometry parameters [S1, unofficial]. Minimize $\mathcal{L}_{\text{track}}$ using multi-view augmentation to restore texture fields around deformed locations [S1, unofficial].
4.  **Alternating Optimization:** Repeat steps 2 and 3 until convergence [S1, unofficial]. This separation prevents instability caused by competing geometry and appearance updates [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def postoperative_surface_alignment(G0, mri_depth_data, reference_views, lambda_trans=6.0, lambda_track=10.0):
    """
    Deforms initial Gaussian set G0 to match MRI depth while preserving appearance.
    """
    G = copy(G0)  # Standard deep copy of primitive attributes
    Q = []
    
    # Back-project valid pixels from reference renderings to local coordinates on primitives to store initial colors [S1, unofficial]
    for view in reference_views:
        for pixel in view.valid_pixels:
            prim_idx, u, v, h = back_project_to_primitive(G, view, pixel)
            Q.append((prim_idx, u, v, h, pixel.color))
    
    lr_centers = 1.6e-3
    lr_center_min = 1.6e-5
    max_steps = 1000  # Iteration count not given in sources [G]
    
    for step in range(max_steps):
        # --- Geometry Window ---
        G.opacity = 0.999  # Override to prevent model from reducing loss by making mismatched foreground patches transparent [S1, unofficial]
        
        # Obtain camera-linear depth by reusing explicit ray-surface intersection [S1, unofficial]
        D_hat = render_depth(G, mri_depth_data.camera_params)
        L_depth = compute_depth_loss(D_hat, mri_depth_data.target_depth, mri_depth_data.valid_mask)
        L_trans = compute_transition_loss(G)  # Coupling adjacent charts [S1, unofficial]
        L_track = compute_tracked_loss(G, Q)  # Reproject samples and compare to initial colors [S1, unofficial]
        
        L_total = L_depth + lambda_trans * L_trans + lambda_track * L_track
        grads = compute_gradients(L_total, G)
        
        G.centers -= lr_centers * grads.centers
        G.tangent_scales -= 1e-3 * grads.tangent_scales
        G.rotations -= 1e-4 * grads.rotations
        G.height_coefficients -= 1e-4 * grads.height_coefficients
        G.appearance_coefficients -= 1e-4 * grads.appearance_coefficients
        G.opacity -= 1e-8 * grads.opacity
        
        lr_centers = max(lr_center_min, lr_centers * 0.99)
        
        # --- Appearance Window ---
        # Freeze geometry parameters
        frozen_params = {'centers', 'rotations', 'tangent_scales', 'height_coefficients'}
        
        # Minimize L_track using multi-view augmentation to restore texture fields [S1, unofficial]
        L_track_appearance = compute_tracked_loss(G, Q)
        grads_appearance = compute_gradients(L_track_appearance, G, frozen_params=frozen_params)
        G.appearance_coefficients -= 1e-4 * grads_appearance.appearance_coefficients
        
        if step % 10 == 0:  # Convergence check not specified [G]
            pass
            
    return G
```

**Edge Cases and Pitfalls:**
*   **Opacity Manipulation:** If opacity is not overridden to 0.999 during depth fitting, the model may reduce loss by making mismatched foreground patches transparent rather than moving geometry [S1, unofficial].
*   **Instability:** Simultaneous geometry and appearance updates can yield an unstable compromise [S1, unofficial].
*   **Surface Tearing:** Without the transition objective, neighboring patches can move independently and tear the surface [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1, unofficial].
*   **Memory Complexity:** Not given in the sources [S1, unofficial].
*   **Performance:** Evaluated on five MRI cases using real neurosurgical videos (60–80 images per case) [S1, unofficial].
*   **Strengths:** Retains initial cortical texture, prevents surface tearing, stabilizes visibility through learning rate hierarchy [S1, unofficial].
*   **Limitations:** Requires reference renderings for appearance tracking; alternating optimization increases iteration count [S1, unofficial].
*   **When to use:** When adapting preoperative Gaussian splatting models to postoperative MRI data without cross-modal point correspondences [S1, unofficial].
*   **When not to use:** When real-time performance is critical and alternating optimization is too slow, or when appearance preservation is not required [S1, unofficial].

## Background
Originated in "H-2DGS: Height-Field Gaussian Splatting for Sharp and Deformable Neurosurgical Surface Reconstruction" [S1, unofficial]. It is used in neurosurgical surface reconstruction to align preoperative 3D Gaussian Splatting models with postoperative MRI morphology [S1, unofficial].

## Sources
[S1] H-2DGS: Height-Field Gaussian Splatting for Sharp and Deformable Neurosurgical Surface Reconstruction, https://arxiv.org/abs/2610.30910, fetched 2026-10-26 (unofficial)
