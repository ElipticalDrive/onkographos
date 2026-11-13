# DP-NeRF

Also called: DP-NeRF (Rigid Blur Kernel / RBK & SE(3) Fields)

## Overview
DP-NeRF is a method for reconstructing radiance fields from blurry images by modeling the camera blur kernel as a 3D rigid transformation of rays, thereby imposing physical consistency across views [S1]. It differs from pixel-wise blur models (e.g., DeblurNeRF) by using a scene-wise shared rigid transformation (RBK) defined by SE(3) fields, ensuring that the blurring process respects the 3D geometry of the scene rather than treating pixels independently [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of blurry images from a scene. The method uses latent embedded scene information $l_s$ and ray parameters $r$ (origin and direction).
*   **Output:** A deblurred radiance field representation and rendered sharp images.

**Data Structures:**
*   $l_s$: Latent embedded scene information vector.
*   $S_q \in \mathbb{R}^6$: Scene-wise screw axis for the $q$-th proposal.
*   $T_\Psi$: An MLP with parameters $\Psi$.
*   $r$: Input ray vector.
*   $\hat{B}$: Composite blurred color.
*   $\tilde{m}_q$: Adaptive composition weights.

**Numbered Steps:**
1.  **Predict Screw Axis:** Pass the latent scene information $l_s$ through the MLP $T_\Psi$ to predict a set of screw axes $S_q$ and associated weights $m_q$ for $q \in \{1, \dots, n\}$ [S1].
    $$ (S_{s;q}, m_s) = (r_s, v_s, m_s)_q = T_\Psi(l_s) $$
    Here, $S_q$ is decomposed into a rotation axis $r_q$ and angle $v_q$ [S1].
2.  **Compute Rigid Transformation:** Convert the predicted rotation axis $r_q$ and angle $v_q$ into a rotation matrix $e_{r_q}$ and a translation matrix $p_q$ using Rodrigues' formula and standard SE(3) conversion [S1, S58, S59].
3.  **Transform Rays:** Apply the rigid transformation to the input ray $r$ to obtain the transformed ray $r_q$:
    $$ r_q = e_{r_q} r + p_q $$
    This transformation is shared across all pixels in the image, enforcing 3D consistency [S1].
4.  **Render Transformed Colors:** Render the color $\hat{C}_q$ for each transformed ray $r_q$ using the underlying radiance field [S1].
5.  **Adaptive Weight Proposal:** Compute adaptive composition weights $\tilde{m}_q$ for each pixel. This is done using an Adaptive Weight Proposal (AWP) network that calculates the internal correlation between the transformed rays $r_q$ and the motion axis to account for depth differences [S1].
6.  **Composite Blurred Color:** Compute the final blurred color $\hat{B}$ for the target pixel by weighted summation of the rendered colors:
    $$ \hat{B} = \sum_{q=1}^{n} \tilde{m}_q \hat{C}_q $$
    [S1]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dp_nerf_forward(rays, latent_scene_info, mlp_screw, mlp_awp, radiance_field):
    """
    rays: Tensor of shape (H, W, 4) representing ray origins and directions.
    latent_scene_info: Tensor of shape (D) representing scene embedding.
    mlp_screw: MLP predicting screw axis and initial weights.
    mlp_awp: MLP predicting adaptive weights based on ray correlation.
    radiance_field: Function to render color from ray.
    """
    # Step 1: Predict screw axis and initial weights
    # T_Psi outputs (r_s, v_s, m_s) for q proposals
    screw_predictions = mlp_screw(latent_scene_info)
    # Extract rotation axis r_s and angle v_s for each proposal q
    r_s = screw_predictions['rotation_axis'] # Shape (n, 3)
    v_s = screw_predictions['angle']         # Shape (n,)
    m_s_initial = screw_predictions['weights'] # Shape (n,)

    # Step 2 & 3: Compute rigid transformation and transform rays
    # Convert r_s and v_s to rotation matrices e_rq and translation vectors p_q
    rotation_matrices = []
    translation_vectors = []
    for i in range(len(r_s)):
        rq = r_s[i]
        vs = v_s[i]
        # Rodrigues' formula to get rotation matrix
        R = rodrigues(rq, vs)
        # Translation vector p_q derived from SE(3) screw axis parameters
        # Note: Exact decomposition of screw axis to p_q depends on [58]
        p = compute_translation_from_screw(rq, vs) 
        rotation_matrices.append(R)
        translation_vectors.append(p)

    transformed_rays = []
    for i in range(len(rotation_matrices)):
        R = rotation_matrices[i]
        p = translation_vectors[i]
        # r_q = e_rq * r + p_q
        # Apply to all rays (H, W, 4)
        # Assuming rays are homogeneous or handled appropriately
        r_origins = rays[..., :3]
        r_dirs = rays[..., 3:]
        
        # Transform origin and direction
        new_origins = torch.matmul(r_origins, R.T) + p
        new_dirs = torch.matmul(r_dirs, R.T)
        
        # Reconstruct ray tensor
        new_ray = torch.cat([new_origins, new_dirs], dim=-1)
        transformed_rays.append(new_ray)

    # Step 4: Render colors for transformed rays
    rendered_colors = []
    for tr in transformed_rays:
        # Render using the radiance field
        c = radiance_field(tr)
        rendered_colors.append(c)

    # Step 5: Adaptive Weight Proposal
    # Compute correlation between transformed rays and motion axis
    # This generates adaptive weights m_tilde_q for each pixel
    # AWP takes transformed rays and motion axis info
    adaptive_weights = mlp_awp(transformed_rays, r_s) # Shape (n, H, W)

    # Step 6: Composite
    # B_hat = sum(m_tilde_q * C_hat_q)
    composite_blur = torch.zeros_like(rendered_colors[0])
    for i in range(len(rendered_colors)):
        composite_blur += adaptive_weights[i] * rendered_colors[i]

    return composite_blur
```

**Parameters:**
*   $n$: Number of proposals in the screw axis set. Not given in the sources.
*   $\Psi$: Parameters of the MLP $T_\Psi$. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Depth Inaccuracy:** Blur in images makes predicted depth inaccurate and inconsistent across multiple views, which hinders the usability of depth priors from SfM or pretrained networks [S1].
*   **3D Consistency:** Unlike DeblurNeRF, which uses pixel-wise independent transformations, DP-NeRF's shared rigid transformation ensures 3D consistency. Using pixel-wise kernels on scenes requiring global rigidity may fail [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Imposes physical consistency across images by modeling the blur kernel as a 3D rigid transformation, approximating the actual blurring process in the camera more precisely than pixel-wise methods [S1].
*   **Limitations:** Relies on the assumption that the blur can be modeled as a rigid transformation of rays. May struggle if the blur is not consistent with a single rigid camera motion per view [S1].
*   **When to use:** When reconstructing radiance fields from sparse, blurry images where 3D consistency of the blur is critical [S1].
*   **When not to use:** When the blur is caused by non-rigid motion or when pixel-wise independent blur models are more appropriate (e.g., DeblurNeRF) [S1].

## Background
DP-NeRF was introduced as a key prior work in the field of Deblurred Neural Radiance Fields (DeRF) [S1]. It serves as a main baseline in experiments involving sparse blurry images [S1]. The method was motivated by the need to impose physical consistency across images, differing from earlier approaches like DeblurNeRF which used deformable sparse kernels (DSK) [S1]. It draws inspiration from SE(3) fields and Rodrigues' formula [S1, S29, S58, S59].

## Sources
[S1] Sparse-DeRF: Deblurred Neural Radiance Fields From Sparse View, https://doi.org/10.1109/tpami.2025.3563398, fetched 2026-10-27
