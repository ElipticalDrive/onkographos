# Global Illumination Rendering with Gaussian Surfels

Also called: Forward Global Illumination with Gaussian Surfels

## Overview
This method extends the Gaussian surfel framework to model global illumination by deriving a forward rendering equation from the physically-based rendering equation [S1]. It solves for outgoing radiance in scenes with inter-reflections by treating light sources as special Gaussian surfels and assuming non-overlapping kernels to apply a weighted residual form [S1]. Unlike direct illumination methods, it accounts for indirect light transport, enabling accurate geometry reconstruction in scenes with rich inter-reflections [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ Gaussian surfels (kernels), each defined by position, orientation, opacity $\alpha$, and material properties (diffuse albedo, shininess, emission). Light sources are represented as specific kernels.
*   **Outputs:** Outgoing radiosity $\mathbf{B}_i^c$ for each kernel $i$, which is then rasterized to produce the final image [S1].

**Data Structures:**
*   **Kernel $i$:** Contains position $\mathbf{p}_i$, normal $\mathbf{n}_i$, opacity $\alpha_i(\mathbf{x})$, material BRDF $\mathbf{f}_i$, and emission $\mathbf{E}_i^c$.
*   **Visibility Term $V_{ji}$:** A continuous visibility function accounting for semi-opaque primitives and distance attenuation [S1].

**Assumptions:**
1.  Material, emission, and outgoing radiance are constant within each kernel [S1].
2.  There is no overlapping between any two Gaussian surfels [S1].

**Formulas:**
The forward rendering equation for global illumination is [S1]:

$$
\mathbf{B}_i^c = \mathbf{E}_i^c + \frac{\alpha_i(\mathbf{p}_i)}{\Lambda_i} \times \sum_{j=1}^N \int_{P_j} \int_{P_j} \mathbf{f}_j(\omega_I) \times \left( \mathbf{Y}(\omega_I)^T \mathbf{B}_j^c \right) \times \mathbf{V}_{ji}(\mathbf{x}', \mathbf{x}) d\mathbf{x}' d\mathbf{x}
$$

Where:
*   $\mathbf{B}_i^c$: Outgoing radiosity of kernel $i$.
*   $\mathbf{E}_i^c$: Emission of kernel $i$.
*   $\alpha_i(\mathbf{p}_i)$: Opacity at the center of kernel $i$.
*   $\Lambda_i$: Normalization factor for kernel $i$ (implied by context of Gaussian surfels, though exact definition not explicitly detailed in the provided excerpt beyond its use in the term $\alpha_i/\Lambda_i$) [S1].
*   $N$: Total number of kernels.
*   $\mathbf{f}_j(\omega_I)$: BRDF of kernel $j$ for incoming direction $\omega_I$.
*   $\mathbf{Y}(\omega_I)$: Spherical harmonic basis vector for direction $\omega_I$.
*   $\mathbf{B}_j^c$: Outgoing radiosity of source kernel $j$.
*   $\mathbf{V}_{ji}(\mathbf{x}', \mathbf{x})$: Visibility term from source point $\mathbf{x}'$ in kernel $j$ to target point $\mathbf{x}$ in kernel $i$.
*   $P_j$: Support of kernel $j$.

The visibility term $\mathbf{V}_{ji}$ is defined as [S1]:

$$
\mathbf{V}_{ji}(\mathbf{x}', \mathbf{x}) = \frac{|\mathbf{n}_i \cdot \omega_I| |\mathbf{n}_j \cdot \omega_I|}{\|\mathbf{x} - \mathbf{x}'\|_2^2} \kappa_i(\mathbf{x}) \alpha_j(\mathbf{x}') \prod_{k=1}^{j-1} (1 - \alpha_k)
$$

Where:
*   $\omega_I = \frac{\mathbf{x} - \mathbf{x}'}{\|\mathbf{x} - \mathbf{x}'\|_2^2}$: Incoming direction vector (note: the source uses $\|\mathbf{x} - \mathbf{x}'\|_2^2$ in the denominator of the direction vector, which is dimensionally inconsistent for a direction vector; typically this should be $\|\mathbf{x} - \mathbf{x}'\|_2$. However, the source explicitly writes $\|\mathbf{x} - \mathbf{x}'\|_2^2$ in the definition of $\omega_I$ and $\|\mathbf{x} - \mathbf{x}'\|_2^2$ in the visibility term. We retain the source's notation).
*   $\mathbf{n}_i, \mathbf{n}_j$: Normals of kernels $i$ and $j$.
*   $\kappa_i(\mathbf{x})$: Kernel function (e.g., Gaussian) for kernel $i$ at point $\mathbf{x}$.
*   $\alpha_j(\mathbf{x}')$: Opacity of kernel $j$ at point $\mathbf{x}'$.
*   $\prod_{k=1}^{j-1} (1 - \alpha_k)$: Visibility accumulation from previous kernels (implies an ordering of kernels) [S1].

**Simplified Equation:**
The term $\frac{\alpha_i(\mathbf{p}_i)}{\Lambda_i}$ is absorbed into $\mathbf{f}_i(\omega_l)$ to simplify the deduction [S1]:

$$
\mathbf{B}_i^c = \mathbf{E}_i^c + \sum_{j=1}^N \int_{P_i} \int_{P_j} \mathbf{f}_i(\omega_l) (\mathbf{Y}(\omega_l)^T \mathbf{B}_j^c) V_{ji} d\mathbf{x}' d\mathbf{x}
$$

**Special Cases for Light Sources:**
When kernel $j$ represents a light source, the integration over $P_j$ is skipped [S1]:
1.  Directly sample the central point of kernel $j$ as $\mathbf{x}'$.
2.  Disable the dot product between the normal and incoming direction for kernel $j$ in $V_{ji}$ (i.e., remove $|\mathbf{n}_j \cdot \omega_I|$) [S1].
3.  If the light is directional, further disable the inverse square falloff in $V_{ji}$ (i.e., remove $\frac{1}{\|\mathbf{x} - \mathbf{x}'\|_2^2}$) [S1].

**Algorithm Steps:**
1.  **Initialize:** Set initial outgoing radiosity $\mathbf{B}_j^c$ for all kernels (e.g., based on emission or direct illumination).
2.  **Iterate:** Solve the recursive system of equations for $\mathbf{B}_i^c$. Since $\mathbf{B}^c$ appears on both sides, a solver (e.g., progressive refinement) is required [S1].
    *   For each kernel $i$:
        *   Compute direct emission $\mathbf{E}_i^c$.
        *   For each kernel $j$ (including light sources):
            *   Determine if $j$ is a light source. If so, set $\mathbf{x}'$ to the center and adjust $V_{ji}$ as per special cases.
            *   Compute the visibility term $V_{ji}(\mathbf{x}', \mathbf{x})$.
            *   Integrate over the support $P_j$ and $P_i$ (or just sample center for lights) the term $\mathbf{f}_i(\omega_l) (\mathbf{Y}(\omega_l)^T \mathbf{B}_j^c) V_{ji}$.
        *   Update $\mathbf{B}_i^c$.
3.  **Rasterize:** Use the computed $\mathbf{B}_i^c$ values to render the image via efficient rasterization [S1].

**Pitfalls:**
*   The equation is recursive, requiring an iterative solver [S1].
*   Optimization using only direct illumination fails in scenes with rich inter-reflections (e.g., enclosed boxes) [S1].

## Subtopics

### Scene Component Decomposition
Separates a fully rendered scene into distinct material properties (diffuse albedo, shininess) and illumination components (direct illumination, indirect illumination) [S1]. To extract direct illumination, run the progressive refinement solver for exactly one step [S1]. To extract indirect illumination, compute the difference between the full rendering result and the direct illumination result [S1]. This method is demonstrated on the "CupPlane" scene and a synthetic enclosed box with 7 objects [S1]. Optimization using only direct illumination fails in scenes with rich inter-reflections, requiring global illumination for reasonable geometry reconstruction [S1].

## Operating characteristics
*   **Time Complexity:** The time cost roughly increases linearly as the number of primitives increases, but also depends on the specific distribution of primitives [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Supports both near-field and far-field light sources; derives from physically-based rendering for correctness; enables view-independent rendering with diffuse and specular effects; allows for scene decomposition and relighting [S1].
*   **Limitations:** Requires solving a recursive system; assumes non-overlapping kernels; optimization with direct illumination only fails in complex inter-reflection scenes [S1].
*   **When to use:** When global illumination effects (inter-reflections) are significant, or when scene decomposition (material/illumination separation) is required [S1].
*   **When not to use:** When direct illumination is sufficient, or when the non-overlapping kernel assumption is violated [S1].

## Background
This method is derived from the classic radiosity theory and the rendering equation [Kajiya 1986; Sillion et al. 1991] [S1]. It is part of the first work to derive differentiable global illumination on Gaussian surfels, enabling applications such as sparse-view relighting, material editing, and accurate geometry reconstruction [S1]. The approach was presented in "Differentiable Light Transport with Gaussian Surfels via Adapted Radiosity for Efficient Relighting and Geometry Reconstruction" [S1].

## Sources
[S1] Differentiable Light Transport with Gaussian Surfels via Adapted Radiosity for Efficient Relighting and Geometry Reconstruction, https://doi.org/10.1145/3763305, fetched 2026-10-27
