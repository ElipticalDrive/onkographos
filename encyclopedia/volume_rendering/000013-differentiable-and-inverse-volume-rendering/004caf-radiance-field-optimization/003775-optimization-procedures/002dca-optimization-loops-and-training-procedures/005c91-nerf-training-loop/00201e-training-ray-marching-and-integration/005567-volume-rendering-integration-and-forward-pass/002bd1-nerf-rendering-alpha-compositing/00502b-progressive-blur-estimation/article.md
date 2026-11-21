# Progressive Blur Estimation

Also called: two-stage blur estimation

## Overview
Progressive Blur Estimation (PBE) is a two-stage implicit function module that estimates defocus and motion blur by predicting multiple ray origins, directions, and blending weights for each pixel [S1][S2, unofficial]. It solves the problem of rendering blurry observations from a radiance field by modeling blur as a distribution of rays rather than a single ray [S1][S2, unofficial]. It sets itself apart from single-stage blur models by using a coarse stage to predict initial kernel locations and a refinement stage that incorporates ray features for fine-grained blur modeling, which is more accurate and efficient than a single-stage approach with ray features [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   $x$: 2D pixel coordinate.
    *   $o$: Ray origin.
    *   $d$: Ray direction.
    *   $l$: View embedding for the current view.
    *   $V$: Number of source views.
    *   $P$: Number of rays (kernel size) estimated per pixel.
*   **Outputs:**
    *   $\{r_{x_i}\}_{i=1}^P$: Final set of $P$ rays.
    *   $\{h(x_i)\}_{i=1}^P$: Blending weights for each ray.

**Data Structures:**
*   **Canonical blur kernel location ($x'_k$):** Learnable parameter, shape $\mathbb{R}^{V \times P \times 2}$. Normalized to be at most $\pm 10$ pixels from $x$ [S1][S2, unofficial].
*   **View embedding ($l$):** Learnable parameter, shape $\mathbb{R}^{V \times K}$ [S1][S2, unofficial].
*   **Implicit Function ($H$):** A four-layer Multi-Layer Perceptron (MLP) with channel size 64 and ReLU activation [S1][S2, unofficial].
*   **Ray Feature ($f_{ray}$):** Aggregated per-voxel features from the underlying radiance field, dimension 15 [S1][S2, unofficial].

**Parameters:**
*   $P$: Kernel size. Typical values: 5 or 10 [S1].
*   $K$: View embedding size. Value: 32 [S1][S2, unofficial].
*   $H$: Architecture: 4-layer MLP, channel size 64, ReLU [S1][S2, unofficial].
*   $f_{ray}$ dimension: 15 [S1][S2, unofficial].

**Steps:**

1.  **Coarse Stage:**
    *   Query implicit function $H$ with view embedding $l$, 2D coordinate $x$, and canonical kernel location $x'_k$ (or $x'_i$) to predict offsets $\Delta o_{x'_k}$, $\Delta x'_k$ and initial weights $h'(x^*_k)$ (or $h_0(x^*_i)$) [S1][S2, unofficial].
    *   Formula:
        $$ \Delta o_{x'_k}, \Delta x'_k, h'(x^*_k) = H(l, x, x'_k, \emptyset) $$
        where $\emptyset$ is a zero vector placeholder for additional features [S2, unofficial].
    *   Compute refined location $x^*_k = x'_k + \Delta x'_k$ and origin $o_{x^*_k} = o + \Delta o_{x'_k}$ [S1][S2, unofficial].
    *   Compute coarse ray $r_{x^*_k} = o_{x^*_k} + t d_{x^*_k}$ [S1][S2, unofficial].

2.  **Feature Aggregation:**
    *   Trace ray $r_{x^*_k}$ through the coarse radiance field to aggregate per-voxel features into $f_{ray}(r_{x^*_k})$ [S1][S2, unofficial].
    *   Note: The exact formulation for aggregation is elaborated in the underlying radiance field rendering equations (e.g., Eq. 7 in [S1]), typically involving alpha compositing of features along the ray [S1].

3.  **Refinement Stage:**
    *   Query implicit function $H$ with view embedding $l$, 2D coordinate $x$, coarse location $x^*_k$, and ray feature $f_{ray}(r_{x^*_k})$ to predict offsets $\Delta o_{x^*_k}$, $\Delta x^*_k$ and final weights $h(x_k)$ (or $h(x_i)$) [S1][S2, unofficial].
    *   Formula:
        $$ \Delta o_{x^*_k}, \Delta x^*_k, h(x_k) = H(l, x, x^*_k, f_{ray}(r_{x^*_k})) $$
    *   Compute final location $x_k = x^*_k + \Delta x^*_k$ and origin $o_{x_k} = o + \Delta o_{x^*_k}$ [S1][S2, unofficial].
    *   Compute final ray $r_{x_k} = o_{x_k} + t d_{x_k}$ [S1][S2, unofficial].

4.  **Rendering:**
    *   Use the final set of rays $\{r_{x_i}\}$ and weights $\{h(x_i)\}$ to render the blurry observation using Eq. (3) from the source [S1][S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def progressive_blur_estimation(x, o, d, l, x_prime, H, radiance_field, P):
    """
    x: 2D pixel coordinate (float, shape (2,))
    o: Ray origin (float, shape (3,))
    d: Ray direction (float, shape (3,))
    l: View embedding (float, shape (K,))
    x_prime: Canonical blur kernel locations (float, shape (P, 2))
    H: Implicit function (MLP)
    radiance_field: Coarse radiance field for feature aggregation
    P: Number of rays per pixel
    """
    # Coarse Stage
    # H predicts offsets and initial weights
    # Input to H: l, x, x_prime, empty feature vector
    delta_o_coarse, delta_x_coarse, h_coarse = H(l, x, x_prime, zeros_like(x_prime))
    
    # Compute coarse ray parameters
    x_star = x_prime + delta_x_coarse
    o_star = o + delta_o_coarse
    d_star = x_star - x # Direction from pixel to kernel location? 
                        # Note: Sources define r = o + t*d. 
                        # In Eq 4: rx* = ox* + t dx*. 
                        # dx* is likely the direction component derived from x*.
                        # Assuming dx* is normalized direction from origin to x*?
                        # Or dx* is simply the offset direction.
                        # Let's assume dx* is the direction vector corresponding to x*.
                        # Standard ray definition: r(t) = o + t * d.
                        # Here d is fixed for the pixel. x* is a 2D coordinate.
                        # The sources say "rx* = ox* + t dx*". 
                        # This implies dx* is a direction vector.
                        # Usually, for blur, the ray direction changes slightly or the origin changes.
                        # Eq 4: rx* = ox* + t dx*. 
                        # If dx* is derived from x*, it might be the vector from camera center to x*?
                        # However, standard NeRF rays are fixed direction d.
                        # Let's look at Eq 5: rxi = oxi + t dxi.
                        # It seems dx* and dxi are direction vectors.
                        # If x* is a 2D coordinate, how is dx* defined?
                        # In many blur models, the ray direction is perturbed.
                        # Let's assume dx* is a direction vector computed from x*.
                        # A common interpretation: dx* is the direction from the camera center to the point projected at x*?
                        # Or dx* is simply the offset direction.
                        # Given "rx* = ox* + t dx*", and x* is 2D, dx* must be 3D.
                        # Let's assume dx* is the direction vector corresponding to the 2D coordinate x* relative to the camera intrinsics.
                        # For implementation, we need camera intrinsics. Not given in notes.
                        # We will assume dx* is provided or computed from x* and camera params.
                        # Since notes don't specify camera intrinsics, we assume dx* is a 3D vector.
                        # Let's define dx* as the direction vector for the coarse ray.
    
    # To be precise based on "rx* = ox* + t dx*":
    # dx* is likely the direction vector.
    # If x* is a 2D pixel coordinate, dx* is usually the ray direction for that pixel.
    # But here x* is a *kernel location* offset.
    # Let's assume dx* is the direction vector for the ray originating at ox* and passing through the point corresponding to x*?
    # Or dx* is simply the direction vector d perturbed?
    # The notes say "rx* = ox* + t dx*".
    # Let's assume dx* is a 3D vector.
    
    # For the sake of the algorithm, we assume dx* is computed from x*.
    # A safe assumption: dx* is the direction vector from the camera center to the point at depth t projected at x*.
    # But t is variable.
    # Actually, in blur models, the ray direction is often fixed, and the origin is perturbed, or vice versa.
    # Eq 4: ox* = o + delta_o. rx* = ox* + t dx*.
    # If dx* is the same as d, then the ray direction is fixed, only origin changes.
    # But x* is also updated: x* = x' + delta_x.
    # This suggests the ray direction might also change.
    # Let's assume dx* is the direction vector corresponding to the 2D coordinate x*.
    
    # We will write the code assuming dx* is a function of x*.
    def get_direction_from_2d_coord(x_2d, camera_intrinsics):
        # Placeholder for camera intrinsics conversion
        # Returns 3D direction vector
        pass

    dx_star = get_direction_from_2d_coord(x_star, camera_intrinsics) # [G] Assumption based on standard ray definitions
    
    # Feature Aggregation
    # Trace ray rx* through radiance field
    # Aggregate per-voxel features
    # This requires a ray marching function
    f_ray = aggregate_ray_features(radiance_field, o_star, dx_star)
    
    # Refinement Stage
    # H predicts final offsets and weights
    # Input to H: l, x, x_star, f_ray
    delta_o_refined, delta_x_refined, h_final = H(l, x, x_star, f_ray)
    
    # Compute final ray parameters
    x_final = x_star + delta_x_refined
    o_final = o + delta_o_refined
    dx_final = get_direction_from_2d_coord(x_final, camera_intrinsics) # [G] Assumption
    
    # Final rays
    rays = []
    weights = []
    for i in range(P):
        ray_origin = o_final[i]
        ray_dir = dx_final[i]
        rays.append((ray_origin, ray_dir))
        weights.append(h_final[i])
        
    return rays, weights

def aggregate_ray_features(radiance_field, origin, direction):
    """
    Traces a ray through the radiance field and aggregates features.
    Returns the aggregated feature vector.
    """
    # Standard volume rendering integration
    # Sample points along the ray
    # Query radiance field for features at each point
    # Composite features using alpha compositing
    # Return final feature vector
    pass
```

**Edge Cases and Pitfalls:**
*   When observations are sharp, offsets $\Delta o$ and $\Delta x$ should converge to similar spatial locations [S2, unofficial].
*   When observations are blurry, offsets $\Delta o$ and $\Delta x$ should be dissimilar to account for blur [S2, unofficial].
*   The two-stage design is preferred over a single-stage design with ray features because the coarse stage computes quickly and provides an accurate $x^*$ for a more informative $f_{ray}$ in the refinement stage [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The two-stage design is efficient because the coarse stage (Eq. 4) computes quickly, and the refinement stage benefits from a more accurate initial estimate [S2, unofficial].
*   **Memory Complexity:** The model size for a single scene is 160MB in PDRF, which is smaller than TensoRF's 200MB for forward-facing scenes [S2, unofficial].
*   **Strengths:** Can clearly distinguish foreground details from blurry background pixels, as shown by variance of predicted offsets [S1][S2, unofficial].
*   **Limitations:** Tracing multiple rays is costly, so the kernel size $P$ should be minimal [S1][S2, unofficial].
*   **When to use:** When reconstructing scenes from blurry images, especially when there is significant defocus or motion blur [S1][S2, unofficial].
*   **When not to use:** When the scene is sharp, as the blur estimation may introduce unnecessary complexity [S2, unofficial].

## Background
Progressive Blur Estimation (PBE) was introduced in the PDRF (Progressively Deblurring Radiance Field) paper [S1][S2, unofficial]. It builds on prior works by introducing an implicit function $H$ to model blur [S1][S2, unofficial]. The method was developed to address the challenges of reconstructing 3D scenes from blurry images, which is a common issue in real-world photography [S1][S2, unofficial].

## Sources
[S1] PDRF: Progressively Deblurring Radiance Field for Fast Scene Reconstruction from Blurry Images, https://doi.org/10.1609/aaai.v37i2.25295, fetched 2026-10-26
[S2] PDRF: Progressively Deblurring Radiance Field for Fast and Robust Scene Reconstruction from Blurry Images, https://doi.org/10.48550/arxiv.2208.08049, fetched 2026-10-26 (unofficial)
