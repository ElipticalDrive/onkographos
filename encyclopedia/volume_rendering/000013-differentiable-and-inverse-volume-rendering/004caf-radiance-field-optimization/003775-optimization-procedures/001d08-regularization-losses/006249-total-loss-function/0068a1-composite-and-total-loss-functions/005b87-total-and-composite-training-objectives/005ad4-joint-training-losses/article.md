# Joint Training Losses

Also called: Composite Loss with Pose, Skin, and INN Constraints

## Overview
This method defines a composite objective function for the joint optimization of a Template-Free NeRF (TFS-NeRF), an Implicit Neural Network (INN) for spatial transformation, and a skinning weight prediction network. It solves the ambiguity in dynamic scene reconstruction where incorrect combinations of canonical points and skinning weights can satisfy forward Linear Blend Skinning (LBS) equations [S1]. It distinguishes itself by explicitly constraining the INN and skinning networks through pose, cycle consistency, and skinning weight losses, rather than relying solely on photometric reconstruction [S1].

## How it works
The method optimizes three coupled components simultaneously: the NeRF for appearance, the INN for mapping between view and canonical spaces, and a skinning weight network for deformation [S1].

**Inputs:**
*   Input video frames with known camera poses.
*   Canonical skeleton joints $J_{p0}$ and deformed skeleton joints $J_p$.
*   Point clouds initialized around the canonical skeleton of individual entities [S1].

**Outputs:**
*   Rendered RGB images $\hat{C}(r)$.
*   Transformed points in canonical and deformed spaces.
*   Predicted skinning weights $\hat{w}$.

**Data Structures:**
*   $R$: Set of rays.
*   $P$: Total number of points sampled around bones from each skeleton [S1].
*   $H$: INN transformation function from view to canonical space.
*   $H^{-1}$: Inverse transformation from canonical to view space (or vice versa depending on conditioning, see formula) [S1].
*   $G_{x_c \to w}$: Skinning weight prediction network [S1].
*   $G_{v \to c}$: Transformation from view to canonical space used in Pose Loss (distinct from INN $H$ in notation, but conceptually similar mapping) [S1].

**Loss Components:**

1.  **Reconstruction Loss ($L_{rgb}$):** Minimizes the difference between rendered RGB $\hat{C}(r)$ and input pixel RGB $C(r)$ along ray $r$ [S1].
    $$ L_{rgb} = \sum_{r \in R} ||\hat{C}(r) - C(r)|| $$

2.  **Pose Loss ($L_{pose}$):** Ensures the INN correctly transforms points $X_{J_p}$ sampled around deformed bones to $X_{J_{p0}}$ around canonical bones [S1].
    $$ L_{pose} = \sum_{p \in P} ||X_{J_{p0}} - G_{v \to c}(X_{J_p})|| $$
    *Note: The source formula uses a norm over the set $P$. The specific norm type (L1/L2) is not explicitly defined in the snippet, but L2 is standard for such geometric terms.* [S1]

3.  **Skinning Weight Loss ($L_W$):** Constrains the predicted skinning weight $\hat{w}$ for canonical joints to be a one-hot vector (1 for the respective joint, 0 for others) [S1].
    $$ L_W = ||G_{x_c \to w}(J_{p0}) - \hat{w}||_2^2 $$

4.  **Cycle Consistency Loss ($L_{INN}$):** Ensures the INN transformation is invertible and consistent [S1].
    $$ L_{INN} = ||H(H^{-1}(x_v, J_{p0}), J_p) - x_v||_2^2 $$
    Where $H$ transforms view to canonical, and $H^{-1}$ transforms canonical to view (or vice versa as defined by the specific INN architecture) [S1].

5.  **Consistency Loss ($L_{Consis}$):** Minimizes the L2 distance between the INN-transformed point $x'_c$ and the skinning-weight-conditioned canonical point $x_c$ [S1].
    $$ L_{Consis} = ||x'_c - x_c||_2^2 $$

6.  **Shape Loss ($L_{shape}$):** Accelerates learning by constraining transformed points $x_c$ to fall within a point cloud initialization around the canonical skeleton, with NeRF densities $\alpha=1$ [S1].
    *Formula not explicitly provided in the source text snippet, but described as following Guo et al. [39]* [S1].

**Total Loss:**
The total loss is the weighted sum of these components:
$$ L_{tot} = \lambda_{rgb} L_{rgb} + \lambda_{pose} L_{pose} + \lambda_{W} L_W + \lambda_{INN} L_{INN} + \lambda_{Consis} L_{Consis} + \lambda_{shape} L_{shape} $$
*Note: Specific weights $\lambda$ are not provided in the source notes.* [S1]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_joint_losses(rendered_rgb, input_rgb, deformed_points, canonical_points, 
                         canonical_joints, predicted_weights, inn_transform, 
                         skinning_net, point_cloud_init, nerf_densities):
    # 1. Reconstruction Loss
    # Sum of L2 norms over all rays R
    l_rgb = sum(norm(rendered_rgb[r] - input_rgb[r]) for r in rays)
    
    # 2. Pose Loss
    # Transform deformed points to canonical space using view-to-canonical mapping
    transformed_deformed = inn_transform_view_to_canonical(deformed_points)
    l_pose = sum(norm(canonical_points[p] - transformed_deformed[p]) for p in points_around_bones)
    
    # 3. Skinning Weight Loss
    # Predict weights for canonical joints and compare to one-hot encoding
    predicted_w = skinning_net(canonical_joints)
    target_w = get_one_hot_vector(canonical_joints)
    l_w = norm(predicted_w - target_w)**2
    
    # 4. Cycle Consistency Loss
    # Transform view point to canonical, then back to view
    x_v = get_view_point()
    x_c = inn_transform_view_to_canonical(x_v)
    x_v_recon = inn_transform_canonical_to_view(x_c)
    l_inn = norm(x_v_recon - x_v)**2
    
    # 5. Consistency Loss
    # Compare INN-transformed point with skinning-weight-conditioned point
    x_prime_c = inn_transformed_point
    x_c_skinning = skinning_weight_conditioned_canonical_point
    l_cons = norm(x_prime_c - x_c_skinning)**2
    
    # 6. Shape Loss
    # Constraint based on point cloud initialization and NeRF density
    # Specific formula depends on Guo et al. [39], assumed to be a distance/penalty term
    l_shape = compute_shape_loss(x_c_skinning, point_cloud_init, nerf_densities)
    
    return l_rgb, l_pose, l_w, l_inn, l_cons, l_shape
```

## Subtopics

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** The method enables template-free reconstruction of dynamic scenes [S1].
*   **Strengths:** Constrains ambiguous solutions in LBS by supervising skeleton space [S1].
*   **Limitations:** Relies on accurate point sampling around bones [S1].

## Background
Originated in TFS-NeRF (Template-Free NeRF for Semantic 3D Reconstruction of Dynamic Scene) [S1]. The shape loss component follows Guo et al. [39] [S1].

## Sources
[S1] TFS-NeRF: Template-Free NeRF for Semantic 3D Reconstruction of Dynamic Scene, https://arxiv.org/abs/2409.17459v4, fetched 2026-10-26
