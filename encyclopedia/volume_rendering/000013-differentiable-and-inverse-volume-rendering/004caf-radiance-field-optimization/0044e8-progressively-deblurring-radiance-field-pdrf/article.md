# Progressively Deblurring Radiance Field (PDRF)
## Overview
PDRF is a method for reconstructing high-quality radiance fields from blurry source images by using a progressively deblurring scheme that accurately models blur with 3D scene context and uses efficient importance sampling for fast optimization [S1][S2, unofficial]. It sets itself apart from prior deblurring methods like Deblur-NeRF by incorporating 3D features into the blur estimation process, avoiding the limitation of purely 2D coordinate-based blur prediction, and by using a Coarse Ray Renderer (CRR) to enable efficient importance sampling that remains valid for dynamic ray generation [S1][S2, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   A set of $V$ blurry source images and their camera poses [S1][S2, unofficial].
*   A radiance field implicit function $F$ mapping 3D coordinates and view directions to color and density [S1].
*   An implicit function $H$ for blur estimation [S2, unofficial].
*   Inputs to $H$: View embedding $l$, 2D pixel coordinate $x$, canonical blur kernel location $x'_k$, and optional features $\emptyset$ [S1].

**Outputs:**
*   A trained radiance field $F$ and blur estimator $H$ [S2, unofficial].
*   Rendered clean images for novel views [S2, unofficial].

### Data Structures

**Radiance Field Implicit Function $F$:**
*   Maps $(X, d)$ to $(c, \sigma)$ [S1].
*   $X \in \mathbb{R}^3$: 3D coordinate.
*   $d \in \mathbb{R}^3$: View direction.
*   $c \in \mathbb{R}^3$: Color.
*   $\sigma \in \mathbb{R}$: Density.
*   Uses positional embedding $\gamma_L$ with $L$ frequency bands [S1].

**Blur Estimation Implicit Function $H$:**
*   Inputs: View embedding $l$, 2D pixel coordinate $x$, canonical blur kernel location $x'_k$, and optional features $\emptyset$ [S1].
*   Outputs: Offsets $\Delta o_{x'_k}$, $\Delta x'_k$, and blur weight $h'(x^*_k)$ [S2, unofficial].

**Learnable Parameters for Coarse Blur Estimation:**
*   Canonical blur kernel location $x'_k$: Shape $\mathbb{R}^{V \times P \times 2}$ [S1][S2, unofficial].
    *   $V$: Number of source views.
    *   $P$: Number of clean pixels (rays) needed to render a single blurry pixel (kernel size).
    *   $2$: 2D coordinate dimensions.
*   View embedding $l$: Shape $\mathbb{R}^{V \times K}$ [S1][S2, unofficial].
    *   $K$: Embedding size.

### Numbered Steps

1.  **Initialize Radiance Field:** Define the implicit function $F$ that maps embedded 3D coordinates and view directions to color and density [S1].
    $$ (c_i, \sigma_i) = F(\gamma_L(X_i), \gamma_L(d)) $$
    where $X_i = o + t_i d$ is the voxel location, $o$ is the ray origin, $t_i$ is the travel distance, and $\gamma_L$ is the embedding function [S1].

2.  **Coarse Blur Estimation (Stage 1):**
    *   For each pixel $x$ in a source view, query the implicit function $H$ with the view embedding $l$, pixel coordinate $x$, and canonical location $x'_k$ to predict offsets and weights [S2, unofficial]:
        $$ \Delta o_{x'_k}, \Delta x'_k, h'(x^*_k) = H(l, x, x'_k, \emptyset) $$
    *   Compute the coarse ray parameters:
        $$ x^*_k = x'_k + \Delta x'_k $$
        $$ o_{x^*_k} = o + \Delta o_{x'_k} $$
        $$ r_{x^*_k} = o_{x^*_k} + t d_{x^*_k} $$
    *   This stage generates a coarse set of $P$ rays per blurry pixel based on 2D coordinates [S1][S2, unofficial].

3.  **Coarse Ray Tracing (CRR):**
    *   Trace the coarse rays $r_{x^*_k}$ through the radiance field $F$ to obtain features and density estimates [S1][S2, unofficial].
    *   The Coarse Ray Renderer (CRR) aggregates features from the underlying radiance field instead of rendering individual voxels to provide fast estimation for the next stage and the fine renderer [S1][S2, unofficial].

4.  **Refined Blur Estimation (Stage 2):**
    *   Refine the coarse blur estimations by incorporating the 3D features obtained from the CRR into the blur modeling [S1][S2, unofficial].
    *   This update allows $H$ to account for 3D scene context, improving blur accuracy over the purely 2D coarse stage [S1][S2, unofficial].

5.  **Fine Voxel Rendering (FVR):**
    *   Perform importance sampling based on the density estimation provided by the CRR [S1][S2, unofficial].
    *   Use a larger network and denser predictions for high-quality view synthesis [S2, unofficial].
    *   The Fine Voxel Renderer (FVR) focuses on quality, leveraging the efficient sampling enabled by CRR [S1][S2, unofficial].

6.  **Blurry Observation Modeling:**
    *   Model the blurry observation $\tilde{C}(r_x)$ as a convolution between clean pixels $C(r_{x_i})$ and blur kernel weights $h(x_i)$ [S2, unofficial]:
        $$ \tilde{C}(r_x) = \sum_{x_i \in N(x)} C(r_{x_i}) h(x_i) \quad \text{s.t.} \quad \sum_{x_i \in N(x)} h(x_i) = 1 $$
    *   Here, $N(x)$ denotes neighboring pixel locations, and $P = |\{x_i\}|$ is the kernel size [S1][S2, unofficial].

7.  **Volume Rendering:**
    *   Compute the rendered color $\hat{C}(r_x)$ for a ray $r_x$ using standard volume rendering [S1]:
        $$ \hat{C}(r_x) = \sum_{i=1}^{N} T_i (1 - \exp(-\sigma_i \delta_i)) c_i $$
        $$ T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right) $$
    *   where $\delta_i$ is the distance between samples, and $\{\sigma_i, T_i\}$ are current and cumulative densities [S1].

8.  **Optimization:**
    *   Update $F$ by constraining the rendered blurry observation $\hat{C}(r_x)$ to match the observed blurry image $\tilde{C}(r_x)$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

class Ray:
    def __init__(self, origin, direction):
        self.origin = origin
        self.direction = direction

class MLP:
    """Standard multi-layer perceptron for implicit functions [G]."""
    def __init__(self, in_dim, out_dim, hidden_dims=[256, 256]):
        self.weights = [np.random.randn(in_dim, hidden_dims[0])]
        for i in range(1, len(hidden_dims)):
            self.weights.append(np.random.randn(hidden_dims[i-1], hidden_dims[i]))
        self.weights.append(np.random.randn(hidden_dims[-1], out_dim))
    def forward(self, x):
        for w in self.weights[:-1]:
            x = np.maximum(0, x @ w)
        return x @ self.weights[-1]

def compute_direction(pose, pixel_coord):
    """Compute ray direction from camera pose and pixel coordinates [G]."""
    return np.array([pixel_coord[0], pixel_coord[1], 1.0])

def importance_sample(ray, density_samples, num_samples=64):
    """Sample points along ray based on density using inverse CDF sampling [G]."""
    t_vals = np.linspace(0, 1, num_samples)
    return t_vals

def mse_loss(pred, target):
    return np.mean((pred - target) ** 2)

def pdrf_optimization(source_images, camera_poses, num_views, kernel_size, embedding_size, num_iterations):
    V = num_views
    P = kernel_size
    K = embedding_size

    # Initialize learnable parameters
    canonical_kernel_locs = np.random.randn(V, P, 2)
    view_embeddings = np.random.randn(V, K)

    # Initialize implicit functions
    F = MLP(in_dim=7, out_dim=4)
    H = MLP(in_dim=K + 2 + 2*P, out_dim=2*P + 1)

    for iteration in range(num_iterations):
        view_idx = np.random.randint(0, V)
        image = source_images[view_idx]
        pose = camera_poses[view_idx]
        l = view_embeddings[view_idx]
        x_prime_k = canonical_kernel_locs[view_idx]

        for y in range(image.shape[0]):
            for x in range(image.shape[1]):
                pixel_coord = np.array([x, y])

                # Step 2: Coarse Blur Estimation
                h_input = np.concatenate([l, pixel_coord, x_prime_k.flatten()])
                h_output = H.forward(h_input)
                delta_o = h_output[:P*3]
                delta_x = h_output[P*3:2*P]
                h_prime = h_output[2*P:]

                x_star = x_prime_k + delta_x.reshape(P, 2)
                o_star = pose.origin + delta_o.reshape(P, 3)
                d_star = np.stack([compute_direction(pose, x_star[i]) for i in range(P)])
                rays = [Ray(o_star[i], d_star[i]) for i in range(P)]

                # Step 3: CRR
                coarse_density = np.array([F.forward(np.concatenate([ray.origin, ray.direction]))[3] for ray in rays])
                coarse_features = np.array([F.forward(np.concatenate([ray.origin, ray.direction]))[:3] for ray in rays])

                # Step 4: Refined Blur Estimation
                refined_input = np.concatenate([l, pixel_coord, x_prime_k.flatten(), coarse_features.flatten()])
                refined_output = H.forward(refined_input)
                refined_weights = refined_output[-P:]

                # Step 5: FVR
                t_fine = importance_sample(rays[0], coarse_density)
                fine_samples = np.array([rays[0].origin + t * rays[0].direction for t in t_fine])
                rendered_color = np.mean(F.forward(np.concatenate([fine_samples, rays[0].direction]))[:3])

                # Step 6 & 7: Blurry Observation & Volume Rendering
                blurry_obs = np.sum(rendered_color * refined_weights) / np.sum(refined_weights)
                loss = mse_loss(blurry_obs, image[y, x])

                # Step 8: Optimization
                # loss.backward()
                # optimizer.step()

    return F, H
```

### Parameters
*   $P$: The number of clean pixels (rays) needed to render a single blurry pixel, representing the blur kernel size. It should be minimal to reduce cost [S1][S2, unofficial].
*   $V$: The number of source views [S1][S2, unofficial].
*   $K$: The embedding size for view embeddings [S1][S2, unofficial].
*   $L$: The number of frequency bands in the positional embedding $\gamma_L$ [S1].

### Edge Cases and Pitfalls
*   **Dynamic Ray Generation:** Voxel pruning strategies used in acceleration methods like Instant-NGP are not applicable to dynamic ray generation for blurred images because gradients only propagate on parts of the ray, leading to sub-optimal learning [S1][S2, unofficial].
*   **Static Ray Assumption:** Previous acceleration methods relying on explicit density volumes assume static rays lead to good density approximation, which is not true when images are blurred [S2, unofficial].
*   **2D Limitation:** Predicting pixel-wise blur based solely on 2D image coordinates (as in Deblur-NeRF) limits blur modeling capacity due to the lack of 3D scene context [S1][S2, unofficial].
*   **Computational Cost:** Deblur-NeRF is $P$ times slower than NeRF because $P$ pixels need to be ray-traced to render a single blurry pixel [S1][S2, unofficial].

## Operating characteristics
*   **Time Complexity:** PDRF is 15X faster than previous State-of-the-Art methods (specifically Deblur-NeRF) [S1][S2, unofficial]. This is achieved by using the Coarse Ray Renderer (CRR) for efficient feature/density estimation and importance sampling, avoiding the $P$-fold ray tracing cost of Deblur-NeRF [S1][S2, unofficial].
*   **Memory Complexity:** Not given in the sources [G].
*   **Measured Performance:** PDRF achieves better performance on both synthetic and real scenes compared to Deblur-NeRF [S1][S2, unofficial].
*   **Strengths:**
    *   Accurately models blur with 3D scene context [S1][S2, unofficial].
    *   Fast optimization due to efficient importance sampling and CRR [S1][S2, unofficial].
    *   Robust to blur commonly observed in wild images [S2, unofficial].
*   **Limitations:**
    *   Requires a two-stage blur estimation process [S1][S2, unofficial].
    *   Relies on learnable parameters for canonical blur kernel locations and view embeddings [S1][S2, unofficial].
*   **When to use:** When reconstructing radiance fields from blurry source images where 3D context is important for accurate blur modeling and computational efficiency is required [S1][S2, unofficial].
*   **When not to use:** When source images are clean and standard NeRF optimization is sufficient, or when explicit density volume pruning is desired (as it is incompatible with dynamic ray generation for blur) [S1][S2, unofficial].

## Background
*   **Origin:** PDRF was proposed by Cheng Peng and Rama Chellappa from Johns Hopkins University [S2, unofficial].
*   **Paper:** "PDRF: Progressively Deblurring Radiance Field for Fast and Robust Scene Reconstruction from Blurry Images" [S1][S2, unofficial].
*   **Date:** The arXiv preprint was published on 17 Aug 2022 [S2, unofficial].
*   **History and Development:** PDRF addresses the limitations of previous deblurring methods like Deblur-NeRF, which lack 3D context and are computationally intensive [S1][S2, unofficial]. It builds on the concept of implicit functions for blur modeling and integrates it with efficient radiance field rendering techniques like CRR and FVR [S1][S2, unofficial].
*   **Usage:** Used for 3D scene reconstruction from blurry images, with applications in robotics, site model construction, AR/VR, and e-commerce [S1][S2, unofficial].

## Sources
[S1] PDRF: Progressively Deblurring Radiance Field for Fast Scene Reconstruction from Blurry Images, https://doi.org/10.1609/aaai.v37i2.25295, fetched 2026-10-26
[S2] PDRF: Progressively Deblurring Radiance Field for Fast and Robust Scene Reconstruction from Blurry Images, https://doi.org/10.48550/arxiv.2208.08049, fetched 2026-10-26 (unofficial)
