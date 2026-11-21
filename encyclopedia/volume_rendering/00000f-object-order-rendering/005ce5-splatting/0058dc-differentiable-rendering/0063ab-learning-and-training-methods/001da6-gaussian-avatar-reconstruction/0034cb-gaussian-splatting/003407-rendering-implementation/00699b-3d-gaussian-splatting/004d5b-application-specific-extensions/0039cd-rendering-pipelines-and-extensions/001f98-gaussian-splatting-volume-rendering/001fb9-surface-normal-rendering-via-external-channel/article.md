# Surface Normal Rendering via External Channel

Also called: External Channel Normal Rendering, PyTorch3D Normal Rendering for Gaussians

## Overview
This method renders surface normals for 3D Gaussian Splatting by treating normals as an extra attribute in an external channel, since Gaussian ellipsoids do not natively contain surface normal information [S1, unofficial]. It sets itself apart from sibling methods like [Learnable Normals Parameterization](9615-learnable-normals-parameterization.md) by explicitly modifying the Gaussian's internal attribute representation rather than relying on external gradient supervision, and from [Normal Field Supervision via Rendering and Depth Derivatives](9627-normal-field-supervision-via-rendering-and-depth-derivatives.md) by focusing on the rendering pipeline mechanics rather than the supervision loss formulation.

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussian ellipsoids $\{G_i\}$, where each Gaussian $i$ is defined by a mean position $\mu_i$, a covariance matrix $\Sigma_i$ (or rotation $R_i$ and scale $S_i$), and a base color. For normal rendering, an additional attribute $n_i$ (the surface normal vector) is associated with each Gaussian.
*   **Output:** A rendered image containing surface normal vectors (typically encoded as RGB values) and optionally a depth map.

**Data Structures:**
*   **Gaussian Attributes:** Each Gaussian $i$ stores:
    *   $\mu_i \in \mathbb{R}^3$: Mean position.
    *   $n_i \in \mathbb{R}^3$: Surface normal vector (unit length).
    *   $c_i \in \mathbb{R}^3$: Base color (RGB).
    *   $\Sigma_i \in \mathbb{R}^{3 \times 3}$: Covariance matrix defining the ellipsoid shape and orientation.
    *   $\alpha_i \in \mathbb{R}$: Opacity.

**Steps:**
1.  **Normal Computation (Pre-processing):** If the Gaussians are derived from a mesh (e.g., Stanford 3D Scanning Repository models like Bunny, Dragon, Armadillo), compute the surface normals for the source mesh vertices using a library such as PyTorch3D [S1, unofficial]. These normals are then assigned to the corresponding Gaussians.
2.  **Attribute Assignment:** Assign the computed normal $n_i$ to each Gaussian $i$. This normal is treated as an RGB value for the purpose of rendering.
3.  **Rendering:** Render the Gaussian ellipsoids using standard alpha-blending splatting. During the splatting process, interpolate the normal attribute $n_i$ into the image plane, similar to how color $c_i$ is interpolated. The result is an image-like map of surface normals [S1, unofficial].
4.  **Lighting Integration (Optional):** To integrate lighting, first render the surface normals into an image-like map. Then, use a diffuse lighting function (e.g., `PyTorch3D.renderer.lighting`) to compute the brightness under a point light source based on the rendered normals and the light direction [S1, unofficial].

**Formulas:**
The rendering of the normal field follows the standard alpha-compositing equation, where the normal $N$ at a pixel is the weighted average of the normals of the splatted Gaussians:
$$ N = \frac{\sum_{i \in \text{sorted}} n_i \cdot \alpha_i \cdot \prod_{j < i} (1 - \alpha_j)}{\sum_{i \in \text{sorted}} \alpha_i \cdot \prod_{j < i} (1 - \alpha_j)} $$
where the Gaussians are sorted back-to-front along the view ray.

**Parameters:**
*   **Light Position:** Defined by the user for lighting integration [S1, unofficial].
*   **Number of Gaussians:** Affects rendering quality. Increasing the number of Gaussians significantly improves rendering quality, though boundaries may be slightly fuzzier compared to mesh renderers [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Boundary Fuzziness:** The VoGE renderer may produce slightly fuzzier boundaries compared to traditional mesh renderers due to the volumetric nature of Gaussians [S1, unofficial].
*   **Normal Consistency:** The quality of the rendered normals depends heavily on the accuracy of the initial normal computation from the source mesh or point cloud [S1, unofficial].

## Subtopics

### [Learnable Normals Parameterization](9615-learnable-normals-parameterization.md)
A parameterization for learnable surface normals of Gaussians that uses a sign scalar and a direction vector to stabilize optimization and allow normal flipping without rotation. It solves the instability inherent in naive vector parameterization, where flipping a normal requires a 180-degree rotation that causes convergence to inaccurate local minima. It sets itself apart from sibling methods like [Normal Field Supervision via Rendering and Depth Derivatives](9627-normal-field-supervision-via-rendering-and-depth-derivatives.md) by explicitly modifying the Gaussian's internal attribute representation rather than relying on external gradient supervision.

### [Normal Field Supervision via Rendering and Depth Derivatives](9627-normal-field-supervision-via-rendering-and-depth-derivatives.md)
This method supervises the learnable surface normal attributes of 3D Gaussians by enforcing consistency between the rendered normal map and the geometric normals derived from the rendered depth map. It solves the problem of noisy or inconsistent normal estimates in Gaussian Splatting by using the depth map as a geometry prior. It differs from pure gradient-based approaches by explicitly optimizing normal attributes per Gaussian and supervising them against pseudo-normals computed from expected depth.

### Rendering Normal Field with Gaussians
To render the normal field, attribute a single value $N_i$ per Gaussian to approximate the integral of the normal field along the ray [S2]. For the vector field $V$, select $V_i$ proportional to the integral of the i-th term of the sum multiplied by the normalized density of the Gaussian [S2]. Due to symmetry around $n_i$, $V_i \propto n_i$, and thus the normalized vector field $N_i \sim n_i$. Render plane normals $n_i$ with Gaussian splatting to estimate the average normal field value [S2]. The integral is defined as:
$$ V_i \propto \int_{\mathbb{R}^3} G_i(x) \cdot \mathbf{1}_{n_i^T(x-\mu_i) \geq 0} \nabla \log(1 - G_i(x)) dx $$
[S2].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of Gaussians $N$ and pixels $P$, similar to standard Gaussian Splatting rendering.
*   **Memory Complexity:** Additional memory required to store the normal vector $n_i$ for each Gaussian.
*   **Performance:** Rendering quality improves significantly with an increased number of Gaussians [S1, unofficial].
*   **Strengths:** Allows for lighting integration in Gaussian Splatting pipelines [S1, unofficial]. Simple to implement by treating normals as an RGB attribute.
*   **Limitations:** Boundaries may be fuzzier than mesh-based renderers [S1, unofficial]. Requires pre-computed normals if starting from a mesh.

## Background
This method was introduced in the context of VoGE (Volume Gaussian Ellipsoids), a differentiable volume renderer using Gaussian ellipsoids for analysis-by-synthesis [S1, unofficial]. It was developed to address the lack of native surface normal information in Gaussian ellipsoids, which are represented as volumes. The approach leverages PyTorch3D for normal computation and lighting integration [S1, unofficial].

## Sources
[S1] VoGE: A Differentiable Volume Renderer using Gaussian Ellipsoids for Analysis-by-Synthesis, https://doi.org/10.48550/arxiv.2205.15401, fetched 2026-10-26 (unofficial)
[S2] From Blobs to Spokes: High-Fidelity Surface Reconstruction via Oriented Gaussians, https://doi.org/10.1007/978-3-032-37095-2_4, fetched 2026-10-29
