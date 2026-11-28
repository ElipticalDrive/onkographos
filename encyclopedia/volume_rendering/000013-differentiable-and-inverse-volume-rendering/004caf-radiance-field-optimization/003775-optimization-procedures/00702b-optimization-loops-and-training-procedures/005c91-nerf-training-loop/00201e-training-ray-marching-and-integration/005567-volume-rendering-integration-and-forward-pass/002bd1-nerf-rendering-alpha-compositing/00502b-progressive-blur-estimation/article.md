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
    *   $\{r_{x_k}\}_{k=1}^P$: Final set of $P$ rays.
    *   $\{h(x_k)\}_{k=1}^P$: Blending weights for each ray.

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
    *   Query implicit function $H$ with view embedding $l$, 2D coordinate $x$, and canonical kernel location $x'_k$ to predict offsets $\Delta o_{x'_k}$, $\Delta x'_k$ and initial weights $h'(x^*_k)$ [S1][S2, unofficial].
    *   Formula:
        $$ \Delta o_{x'_k}, \Delta x'_k, h'(x^*_k) = H(l, x, x'_k, \emptyset) $$
        where $\emptyset$ is a zero vector placeholder for additional features [S2, unofficial].
    *   Compute refined location $x^*_k = x'_k + \Delta x'_k$ and origin $o_{x^*_k} = o + \Delta o_{x'_k}$ [S1][S2, unofficial].
    *   Compute coarse ray $r_{x^*_k} = o_{x^*_k} + t d_{x^*_k}$ [S1][S2, unofficial].

2.  **Feature Aggregation:**
    *   Trace ray $r_{x^*_k}$ through the coarse radiance field to aggregate per-voxel features into $f_{ray}(r_{x^*_k})$ [S1][S2, unofficial].
    *   Note: The exact formulation for aggregation is elaborated in the underlying radiance field rendering equations (e.g., Eq. 7 in [S1]), typically involving alpha compositing of features along the ray [S1].

3.  **Refinement Stage:**
    *   Query implicit function $H$ with view embedding $l$, 2D coordinate $x$, coarse location $x^*_k$, and ray feature $f_{ray}(r_{x^*_k})$ to predict offsets $\Delta o_{x^*_k}$, $\Delta x^*_k$ and final weights $h(x_k)$ [S1][S2, unofficial].
    *   Formula:
        $$ \Delta o_{x^*_k}, \Delta x^*_k, h(x_k) = H(l, x, x^*_k, f_{ray}(r_{x^*_k})) $$
    *   Compute final location $x_k = x^*_k + \Delta x^*_k$ and origin $o_{x_k} = o + \Delta o_{x^*_k}$ [S1][S2, unofficial].
    *   Compute final ray $r_{x_k} = o_{x_k} + t d_{x_k}$ [S1][S2, unofficial].

4.  **Rendering:**
    *   Use the final set of rays $\{r_{x_k}\}$ and weights $\{h(x_k)\}$ to render the blurry observation using Eq. (3) from the source [S1][S2, unofficial].

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
    delta_o_coarse, delta_x_coarse, h_coarse = H(l, x, x_prime, np.zeros(x_prime.shape))

    # Compute coarse ray parameters
    x_star = x_prime + delta_x_coarse
    o_star = o + delta_o_coarse
    # dx_star is the direction vector corresponding to 2D coordinate x_star
    # Computed via standard pinhole projection from camera intrinsics [G]
    dx_star = compute_direction_from_2d(x_star)

    # Feature Aggregation
    # Trace ray rx* through radiance field to aggregate per-voxel features
    # Exact formulation uses alpha compositing of features along the ray [G]
    f_ray = aggregate_features_along_ray(radiance_field, o_star, dx_star)

    # Refinement Stage
    # H predicts final offsets and weights
    # Input to H: l, x, x_star, f_ray
    delta_o_refined, delta_x_refined, h_final = H(l, x, x_star, f_ray)

    # Compute final ray parameters
    x_final = x_star + delta_x_refined
    o_final = o + delta_o_refined
    dx_final = compute_direction_from_2d(x_final)

    # Final rays and weights
    rays = []
    weights = []
    for k in range(P):
        rays.append((o_final[k], dx_final[k]))
        weights.append(h_final[k])

    return rays, weights

def compute_direction_from_2d(x_2d):
    # Standard pinhole camera projection to 3D direction [G]
    pass

def aggregate_features_along_ray(radiance_field, origin, direction):
    # Standard volume rendering integration to aggregate per-voxel features [G]
    pass
```

**Edge Cases and Pitfalls:**
*   When observations are sharp, offsets $\Delta o$ and $\Delta x'_k$ should converge to similar spatial locations [S2, unofficial].
*   When observations are blurry, offsets $\Delta o$ and $\Delta x'_k$ should be dissimilar to account for blur [S2, unofficial].
*   The two-stage design is preferred over a single-stage design with ray features because the coarse stage computes quickly and provides an accurate $x^*$ for a more informative $f_{ray}$ in the refinement stage [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The two-stage design is efficient because the coarse stage (Eq. 4) computes quickly, and the refinement stage benefits from a more accurate initial estimate [S2, unofficial].
*   **Memory Complexity:** The model size for a single scene is 160MB in PDRF, which is smaller than TensoRF's 200MB for forward-facing scenes [S2, unofficial].
*   **Strengths:** Can clearly distinguish foreground details from blurry background pixels, as shown by variance of predicted offsets [S1][S2, unofficial].
*   **Limitations:** Tracing multiple rays is costly, so the kernel size $P$ should be minimal [S1][S2, unofficial].

## Background
Progressive Blur Estimation (PBE) was introduced in the PDRF (Progressively Deblurring Radiance Field) paper [S1][S2, unofficial]. It builds on prior works by introducing an implicit function $H$ to model blur [S1][S2, unofficial].

## Sources
[S1] PDRF: Progressively Deblurring Radiance Field for Fast Scene Reconstruction from Blurry Images, https://doi.org/10.1609/aaai.v37i2.25295, fetched 2026-10-26
[S2] PDRF: Progressively Deblurring Radiance Field for Fast and Robust Scene Reconstruction from Blurry Images, https://doi.org/10.48550/arxiv.2208.08049, fetched 2026-10-26 (unofficial)
