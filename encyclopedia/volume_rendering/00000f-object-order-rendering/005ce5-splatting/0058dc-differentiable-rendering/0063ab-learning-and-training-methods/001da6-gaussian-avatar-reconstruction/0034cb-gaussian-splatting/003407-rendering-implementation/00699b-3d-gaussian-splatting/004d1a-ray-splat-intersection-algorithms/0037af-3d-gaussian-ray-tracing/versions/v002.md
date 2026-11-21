# 3D Gaussian Ray Tracing

Also called: 3DGRT

## Overview
3D Gaussian Ray Tracing (3DGRT) is a rendering method that replaces the rasterization-based projection of 3D Gaussian Splatting with hardware-accelerated ray tracing to evaluate semi-transparent Gaussian primitives [S1, unofficial]. Unlike sibling methods that rasterize all Gaussians overlapping a screen tile (linear scaling with density), 3DGRT uses a Bounding Volume Hierarchy (BVH) to traverse only intersected Gaussians, enabling logarithmic scaling in dense scenes [S1, unofficial]. It serves as the foundational hardware-accelerated ray tracing approach for 3D Gaussians, upon which later optimizations like GRay [10042-gray.md] and EPBRR [8102-epbrr.md] build.

## How it works
3DGRT renders a scene by casting rays from the camera through each pixel and accumulating color and opacity contributions from 3D Gaussians intersected by the ray, using a BVH for spatial acceleration [S1, unofficial].

**Inputs and Outputs:**
- **Input:** A set of $N$ 3D Gaussians, each defined by a mean position $\mu$, a $3 \times 3$ covariance matrix $\Sigma$ (or rotation $R$ and scale $S$), opacity $\alpha$, and color $c$. A camera ray defined by origin $o$ and direction $d$.
- **Output:** A pixel color $C(p)$ and accumulated opacity.

**Data Structures:**
- **Bounding Volume Hierarchy (BVH):** A tree structure where each node encloses a subset of Gaussians. The leaves contain the proxy meshes for the Gaussians [S2].
- **Proxy Triangle Mesh:** Each 3D Gaussian is bounded by a proxy triangle mesh (typically a sphere or axis-aligned box approximated by triangles) to facilitate intersection testing with hardware ray tracers [S2].

**Steps:**
1. **BVH Construction:** Build a BVH over the proxy triangle meshes of all $N$ Gaussians [S2].
2. **Ray Casting:** For each pixel, cast a ray from the camera origin through the pixel center [S1, unofficial].
3. **BVH Traversal:** Traverse the BVH to find all proxy meshes intersected by the ray [S2].
4. **Gaussian Evaluation:** For each intersected Gaussian, compute its contribution to the ray. The contribution is determined by the Gaussian's density along the ray and its opacity [S1, unofficial].
5. **Accumulation:** Accumulate the color and opacity contributions of the intersected Gaussians in front-to-back order [S1, unofficial]. To optimize performance and avoid register spilling on GPU hardware, contributions are accumulated in fixed-size groups of $N$ Gaussians, requiring repeated BVH traversals for each group [S2].

**Formulas:**
The color $C(p)$ at pixel $p$ is computed as a discrete approximation of the continuous integral along the ray $t \in [t_{\min}, t_{\max}]$ [S1, unofficial]:

$$
C(p) = \int_{t_{\min}}^{t_{\max}} \mathbf{c}(t) \tau(t) dt
$$

where $\tau(t)$ is the transmittance:

$$
\tau(t) = \exp\left(-\int_{t_{\min}}^t \sigma(s) ds\right)
$$

Here, $\mathbf{c}(t)$ is the color at distance $t$, and $\sigma(s)$ is the density of the Gaussian at distance $s$ along the ray [S1, unofficial]. The density $\sigma$ is derived from the 3D Gaussian's probability density function projected onto the ray.

**Parameters:**
- **Group Size $N$:** The number of Gaussians processed per BVH traversal batch to optimize register usage [S2]. The exact value depends on hardware constraints and is not specified in the sources.

**Edge Cases and Pitfalls:**
- **Register Spilling:** Processing too many Gaussians per traversal can cause GPU register spilling, degrading performance. This is mitigated by batching [S2].
- **Proxy Mesh Accuracy:** The proxy mesh must tightly bound the Gaussian to avoid false positives in intersection tests, which would waste computation [S2].
- **Transparency Artifacts:** Naive ray tracing of semi-transparent Gaussians can violate physical absorption-emission models if not handled carefully. 3DGRT relies on the underlying Gaussian definition to provide closed-form transmittance [S1, unofficial].

## Subtopics

### [GRay](10042-gray.md)
A fast ray-tracing method for 3D Gaussian Splatting designed to close the performance gap between ray tracing and rasterization by leveraging the algorithmic advantage of ray tracing: it evaluates only intersected Gaussians, allowing logarithmic scaling in dense scenes with tiny Gaussians. Unlike 3DGS rasterization which operates on tiles and evaluates all Gaussians overlapping a tile for every pixel (linear scaling), GRay uses a Bounding Volume Hierarchy (BVH) to select and evaluate exactly the set of Gaussians intersecting each pixel. It builds on [3D Gaussian Ray Tracing](8101-3d-gaussian-ray-tracing.md), [EPBRR](8102-epbrr.md), and [EDGS](8103-edgs.md).

## Operating characteristics
- **Time Complexity:** Logarithmic scaling with respect to the number of Gaussians in dense scenes, as only intersected Gaussians are evaluated [S1, unofficial]. However, the constant factor is high due to BVH traversal overhead.
- **Memory Complexity:** Linear in the number of Gaussians to store the BVH and proxy meshes [S2].
- **Performance:** On tested hardware, 3DGRT is approximately 9× slower than a recent version of 3DGS rasterization [S2]. This performance gap is attributed to the overhead of ray tracing and BVH traversal compared to the highly optimized rasterization pipeline of 3DGS [S2].
- **Strengths:** Efficient handling of dense scenes with many small Gaussians; physically plausible transparency when combined with proper compositing [S1, unofficial].
- **Limitations:** Significantly slower than rasterization for typical scene densities; sensitive to BVH quality and construction time [S2].

## Background
3D Gaussian Ray Tracing was first studied by Moenne-Loccoz et al. in 2024 as an alternative to rasterization for rendering 3D Gaussians, utilizing hardware-accelerated ray tracing via NVIDIA OptiX [S2]. It was introduced to address the limitations of rasterization in dense scenes and to provide a more physically accurate rendering of semi-transparent Gaussian primitives [S1, unofficial]. The method has since served as a baseline for subsequent research in ray-traced 3D Gaussian rendering, such as GRay and EPBRR [S1, unofficial][S2].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] GRay: Ray Tracing 3D Gaussians Near the Speed of Splats, https://doi.org/10.1145/3804496, fetched 2026-10-29
