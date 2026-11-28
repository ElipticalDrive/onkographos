# Gravity-Aligned Planarity Regularization

Also called: GAP Regularization, SatGeo-NeRF Planarity Loss

## Overview
Gravity-Aligned Planarity Regularization is a geometric regularization technique for volume rendering that aligns depth-inferred surface normals with the gravity axis to enforce local planarity [S1]. It solves the problem of geometric artifacts and non-planar noise on flat surfaces (e.g., ground planes in satellite imagery) by coupling adjacent rays through a shared surface approximation, thereby facilitating cross-ray gradient flow [S1]. Unlike standard NeRF training which treats rays independently, this method imposes area-based geometric constraints [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** A set of rays $R$, where each ray $r$ is defined by origin $o$ and direction $d$. The rays are associated with a camera model (e.g., RPC models with Bundle Adjustment and learned camera adjustments) [S1].
*   **Outputs:** A scalar planarity loss value $L_{planar}$ used for backpropagation [S1].

**Data Structures**
*   **Ray:** Defined by origin $o$ and direction $d$.
*   **Surface Point:** 3D coordinate $p_s(r)$.
*   **Normal:** Unit vector $\hat{n}$.

**Parameters**
*   **Gravity Axis ($g$):** In UTM coordinate systems, the ground lies in the XY-plane, so the Z-axis approximates the inverse gravity vector. Value: $g = (0, 0, 1)^T$ [S1].
*   **Regularization Weight ($\lambda_{planar}$):** Empirically chosen weight for the planarity loss. Value: $0.1$ [S1].
*   **Activation Epoch:** The regularizer is enabled at epoch three of training, once a coarse scene representation is learned [S1].

**Steps**
1.  **Select Neighbors:** For each ray $r$, select two random adjacent rays, $r_{n1}$ and $r_{n2}$, from the four immediate neighboring pixels. To prevent unstable normal calculation, ensure one neighbor is sampled from each of the two adjacent pixels in the X- and Y-axes [S1].
2.  **Estimate Depths:** Evaluate all three rays ($r, r_{n1}, r_{n2}$) to determine their estimated depth values $d(r)$ using the volume rendering equation (Eq. 4 in [S1]) [S1].
3.  **Compute Surface Points:** Convert the depth values into 3D surface points by following each ray along its direction:
    $$ p_s(r) = r(d(r)) = o + d(r)d $$
    This is performed for $r$, $r_{n1}$, and $r_{n2}$ [S1].
4.  **Estimate Surface Normal:** Approximate the local surface plane normal $\hat{n}(r, r_{n1}, r_{n2})$ using the cross product of the vectors formed by the three surface points:
    $$ \hat{n}(r, r_{n1}, r_{n2}) = (p_s(r_{n1}) - p_s(r)) \times (p_s(r_{n2}) - p_s(r)) $$
    Normalize the resulting vector to ensure unit length, denoted as $n(r, r_{n1}, r_{n2})$ [S1].
5.  **Compute Loss:** Calculate the planarity loss by summing the squared L2 distance between the gravity axis $g$ and the estimated surface normals over all rays $R$:
    $$ L_{planar}(R) = \sum_{r \in R} \| g - n(r, r_{n1}, r_{n2}) \|^2 $$
    The total loss contribution is $\lambda_{planar} L_{planar}(R)$ [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gravity_aligned_planarity_regularization(rays, depth_estimator, gravity_axis=(0, 0, 1)):
    """
    Computes the Gravity-Aligned Planarity Regularization loss.
    
    Args:
        rays: List of Ray objects, each having origin 'o' and direction 'd'.
              Each ray must have a method to find adjacent rays.
        depth_estimator: Function that takes a ray and returns estimated depth d(r).
        gravity_axis: 3D vector representing the gravity prior. Default is (0, 0, 1).
        
    Returns:
        scalar: The planarity loss value.
    """
    total_loss = 0.0
    
    for ray in rays:
        # Step 1: Select two random adjacent rays from immediate neighbors
        # Constraint: One neighbor from X-axis adjacent pixels, one from Y-axis adjacent pixels
        neighbor_rays = ray.get_immediate_neighbors()
        # Filter neighbors to ensure spatial diversity (X and Y axis coverage)
        # Implementation detail: Select rn1 and rn2 such that they are not collinear 
        # with ray in a way that creates degenerate triangles, and cover X/Y axes.
        rn1, rn2 = select_diverse_neighbors(neighbor_rays)
        
        # Step 2: Estimate depths for all three rays
        d_r = depth_estimator(ray)
        d_rn1 = depth_estimator(rn1)
        d_rn2 = depth_estimator(rn2)
        
        # Step 3: Compute 3D surface points
        # p_s(r) = o + d(r) * d
        ps_r = ray.o + d_r * ray.d
        ps_rn1 = rn1.o + d_rn1 * rn1.d
        ps_rn2 = rn2.o + d_rn2 * rn2.d
        
        # Step 4: Estimate surface normal via cross product
        # Vector 1: from r to rn1
        vec1 = ps_rn1 - ps_r
        # Vector 2: from r to rn2
        vec2 = ps_rn2 - ps_r
        
        # Cross product
        normal_unnormalized = cross_product(vec1, vec2)
        
        # Normalize to unit length
        normal = normalize(normal_unnormalized)
        
        # Step 5: Compute squared L2 distance to gravity axis
        # || g - n ||^2
        diff = gravity_axis - normal
        loss_term = dot_product(diff, diff)
        
        total_loss += loss_term
        
    return total_loss

def select_diverse_neighbors(neighbors):
    """
    Selects two neighbors from the immediate neighbors list.
    Ensures one is from X-adjacent pixels and one from Y-adjacent pixels.
    """
    # Implementation depends on specific ray indexing/grid structure.
    # The source [S1] mandates: "sample one neighbor from each of the two 
    # adjacent pixels in X- and Y-axis".
    # This function should return two rays satisfying this constraint.
    # If not enough neighbors exist, handle edge case (e.g., return None or skip).
    raise NotImplementedError("Specific neighbor selection logic depends on ray grid topology.")
```

**Edge Cases and Pitfalls**
*   **Degenerate Planes:** If the three surface points are collinear, the cross product yields a zero vector. Normalization of a zero vector is undefined. The source notes that selecting neighbors from distinct X and Y axes helps prevent unstable calculations [S1].
*   **Camera Accuracy:** The accuracy of the surface points depends on the camera model. The method assumes Bundle Adjustment preprocessing and learned camera adjustments (XY-axis shifts per image) are applied to improve accuracy [S1].
*   **Training Timing:** Enabling this regularizer too early (before a coarse scene representation is learned) may hinder convergence. It is explicitly enabled at epoch three [S1].

## Operating characteristics
*   **Time Complexity:** Adds overhead proportional to the number of rays $|R|$ due to the evaluation of two additional rays per primary ray and the cross-product calculation.
*   **Memory Complexity:** Requires storing depth values for adjacent rays temporarily during the loss computation. No persistent large data structures are introduced beyond the standard NeRF representation.
*   **Strengths:** Effectively mitigates overfitting-induced geometric artifacts on flat surfaces (e.g., ground in satellite imagery) [S1]. Couples adjacent rays to facilitate cross-ray gradient flow, unlike independent ray training [S1].
*   **Limitations:** Specific to scenes with a known gravity axis (e.g., satellite imagery in UTM coordinates). Not applicable to arbitrary indoor or non-terrestrial scenes without a defined gravity prior [S1].
*   **Performance:** On the DFC2019 satellite reconstruction benchmark, SatGeo-NeRF (using this regularizer) improved Mean Altitude Error by 14.0% and 11.4% relative to baselines like EO-NeRF and EO-GS [S1].

## Background
Gravity-Aligned Planarity Regularization was introduced in the SatGeo-NeRF paper [S1]. It is designed specifically for geometrically regularized NeRFs for satellite imagery. The method leverages the known alignment of scene content in the UTM coordinate system, where the ground is in the XY-plane and the Z-axis approximates the inverse gravity vector [S1]. It is one of three model-agnostic regularizers proposed in SatGeo-NeRF, alongside Granularity Regularization and Depth-Supervised Regularization [S1].

## Sources
[S1] SatGeo-NeRF: Geometrically Regularized NeRF for Satellite Imagery, https://doi.org/10.5194/isprs-annals-xi-2-2026-367-2026, fetched 2026-10-27
