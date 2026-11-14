# STensoRRF

Also called: StensorR, tensorial able relighting radiance field for single-tensor scene representation

## Overview
STensoRRF is a differentiable volume rendering method that represents a 3D scene using a single 4D tensor to enable high-quality novel view synthesis and relighting [S1, unofficial]. It distinguishes itself from standard NeRF architectures by replacing the multi-network feature grid with a decomposed single tensor and explicitly estimating a BRDF value per point to facilitate relighting via spherical Gaussian environment maps [S1, unofficial]. Unlike sibling methods that rely on complex geometric priors or multiple voxel meshes, STensoRRF achieves a 50% faster convergence rate and 1.5x faster rendering speed by combining tensor decomposition with a simplified reflectance equation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of multi-view images of a scene with unknown illumination; camera poses $(\mathbf{o}, \mathbf{d})$ [S1, unofficial].
*   **Outputs:** Rendered pixel colors for new viewpoints; relit images under new illumination conditions; surface normals and BRDF values for geometry [S1, unofficial].

**Data Structures:**
The core representation is a single 4D tensor $\mathcal{G}_S$ [S1, unofficial].
*   **Dimensions:** The first three dimensions correspond to spatial coordinates $(x, y, z)$ [S1, unofficial]. The fourth dimension is the feature vector [S1, unofficial].
*   **Feature Vector Layout:** The first element is the volume density $\sigma$ [S1, unofficial]. The remaining elements are appearance features $\mathbf{f}$ [S1, unofficial].
*   **Decomposition:** To manage memory and accelerate optimization, the 4D tensor is decomposed into low-rank components using Vector-Matrix (VM) decomposition [S1, unofficial]. This allows efficient retrieval of $\sigma$ and $\mathbf{f}$ given a 3D coordinate [S1, unofficial].

**Decoders:**
Distinct Multi-Layer Perceptrons (MLPs) decode the appearance features $\mathbf{f}$ retrieved from the tensor:
1.  **Color Network:** Decodes $\mathbf{f}$ into view-dependent color $\mathbf{c}$ [S1, unofficial].
2.  **Normal Network:** Decodes $\mathbf{f}$ into surface normal $\mathbf{n}$ [S1, unofficial].
3.  **BRDF Decoder:** Decodes $\mathbf{f}$ into a scalar or vector "BRDF value" $\beta$, which represents the amount of light reflected from the point [S1, unofficial].

**Lighting Model:**
The incident light field (environment map) is simulated using 32 Spherical Gaussians (SGs) [S1, unofficial]. The reflected light integral is approximated by combining the BRDF value, the normal vector, and the SG-based lighting field [S1, unofficial].

**Rendering and Training Steps:**
1.  **Ray Sampling:** For a camera ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$, sample $N$ points $\mathbf{x}_i$ along the ray [S1, unofficial].
2.  **Feature Retrieval:** For each $\mathbf{x}_i$, query the decomposed 4D tensor to obtain density $\sigma_i$ and appearance features $\mathbf{f}_i$ [S1, unofficial].
3.  **Decoding:**
    *   Compute color $\mathbf{c}_i$ from $\mathbf{f}_i$ [S1, unofficial].
    *   Compute normal $\mathbf{n}_i$ from $\mathbf{f}_i$ [S1, unofficial].
    *   Compute BRDF value $\beta_i$ from $\mathbf{f}_i$ [S1, unofficial].
4.  **Volume Rendering (Training):** Render the pixel color $C(\mathbf{r})$ using standard volumetric rendering to supervise the tensor during initial training:
    $$
    C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
    $$
    where $\delta_i$ is the distance between samples [S1, unofficial].
5.  **Surface Point Estimation:** After a certain number of iterations, identify surface points on the ray (e.g., using the density peak or transmittance threshold) [S1, unofficial].
6.  **Relighting Rendering:** For the identified surface points, compute the reflected color using the BRDF value, normal, and SG environment map. This result is used to jointly optimize the tensor field alongside the standard volume rendering loss [S1, unofficial].
7.  **Optimization:** Iteratively update the 4D tensor parameters by minimizing the difference between rendered pixel colors and real image pixel values [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def stensorrrf_forward(ray_origin, ray_direction, t_near, t_far, N, tensor_decomposition, sg_lighting):
    # 1. Sample points along the ray
    t_values = [t_near + i * (t_far - t_near) / N for i in range(N)]
    points = [ray_origin + t * ray_direction for t in t_values]

    densities = []
    colors = []
    normals = []
    brdf_values = []

    for p in points:
        # 2. Retrieve features from the 4D tensor via VM decomposition
        sigma, features = tensor_decomposition.query(p)

        # 3. Decode features using distinct MLPs
        c = color_network(features)
        n = normal_network(features)
        beta = brdf_decoder(features)

        densities.append(sigma)
        colors.append(c)
        normals.append(n)
        brdf_values.append(beta)

    # 4. Volume Rendering for training supervision
    deltas = []
    for i in range(N):
        if i < N - 1:
            deltas.append(t_values[i+1] - t_values[i])
        else:
            deltas.append(t_far - t_values[i])

    alphas = [1.0 - math.exp(-sigma * delta) for sigma, delta in zip(densities, deltas)]
    transmittances = []
    T = 1.0
    for a in alphas:
        transmittances.append(T)
        T *= (1.0 - a)

    rendered_color = sum(T_i * a * c for T_i, a, c in zip(transmittances, alphas, colors))

    # 5. Surface Point Estimation
    surface_idx = max(range(len(densities)), key=lambda i: densities[i])

    if surface_idx is not None:
        surface_normal = normals[surface_idx]
        surface_brdf = brdf_values[surface_idx]

        # 6. Relighting Calculation
        relit_color = compute_reflected_light(surface_brdf, surface_normal, sg_lighting)

        return rendered_color, {
            'normal': surface_normal,
            'brdf_value': surface_brdf,
            'relit_color': relit_color,
            'depth': t_values[surface_idx]
        }
    else:
        return rendered_color, None

def color_network(features):
    # MLP decoding color from appearance features [G]
    return features

def normal_network(features):
    # MLP decoding normal from appearance features [G]
    return features

def brdf_decoder(features):
    # MLP decoding BRDF value from appearance features [G]
    return features

def compute_reflected_light(brdf_value, normal, sg_lighting):
    # Approximates the reflected light integral using Spherical Gaussians [G]
    reflected_light = [0.0, 0.0, 0.0]
    for sg in sg_lighting:
        # Evaluate SG contribution: intensity * BRDF * max(0, dot(normal, direction))
        dot_prod = normal[0]*sg['direction'][0] + normal[1]*sg['direction'][1] + normal[2]*sg['direction'][2]
        contribution = sg['intensity'] * brdf_value * max(0.0, dot_prod)
        reflected_light[0] += contribution
        reflected_light[1] += contribution
        reflected_light[2] += contribution
    return reflected_light
```

**Parameters:**
*   **Spherical Gaussians:** 32 SGs are used to fit the environment map [S1, unofficial].
*   **Tensor Dimensions:** Not given in the sources.
*   **Decomposition Rank:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Surface Estimation:** The method relies on estimating surface points after a certain number of iterations to decode BRDF and normals. If the density field is noisy or under-converged, surface point identification may fail [S1, unofficial].
*   **BRDF Approximation:** The "BRDF value" is a network output approximating reflection properties, not a full analytical BRDF [S1, unofficial].

## Operating characteristics
*   **Convergence:** 50% faster convergence rate compared to existing relighting radiation field models [S1, unofficial].
*   **Speed:** Rendering and relighting speed is approximately 1.5x faster than previous state-of-the-art models [S1, unofficial].
*   **Memory:** Uses a single 4D tensor with VM decomposition, reducing memory overhead compared to methods using multiple voxel meshes or tensors [S1, unofficial].
*   **Strengths:** High-quality novel view synthesis; efficient relighting without complex geometric surface properties; fast convergence [S1, unofficial].
*   **Limitations:** Relies on a simplified reflectance equation; BRDF is estimated rather than physically modeled; performance depends on the accuracy of the 32 SG environment map fit [S1, unofficial].

## Background
STensoRRF (Tensorial Able Relighting Radiance Field for Single-Tensor Scene Representation) was proposed in a preprint published in January 2024 [S1, unofficial]. It addresses the limitations of previous radiance field methods that required multiple voxel meshes or tensors for scene representation and complex geometric priors for relighting [S1, unofficial]. By leveraging tensor decomposition and a single 4D tensor, it aims to accelerate training and enable efficient relighting by directly estimating reflection properties (BRDF value) from appearance features [S1, unofficial].

## Sources
[S1] STensoRRF:Tensorial able Relighting Radiance Field for Single-Tensor Scene Representation, https://doi.org/10.20944/preprints202401.1085.v1, fetched 2026-10-30 (unofficial)
