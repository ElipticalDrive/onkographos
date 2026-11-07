# Multiple Importance Sampling for Boundary Paths

Also called: MIS for direct boundary paths

## Overview
This method combines the unidirectional and bidirectional volume boundary estimators using path-level Multiple Importance Sampling (MIS) with the balanced heuristic to reduce variance in differentiable volume rendering [S1]. It addresses the limitation of single-estimator approaches by allowing independent sampling of boundary paths from both strategies, weighting their contributions based on their respective sampling probabilities [S1]. Unlike the unidirectional estimator which traces from the detector, or the bidirectional estimator which connects a sampled boundary vertex to both light and detector, this method treats the boundary path as a complete entity sampled by either strategy and blends them [S1].

## How it works
**Inputs:**
*   Scene geometry, participating media properties (optical density, phase function), and light sources.
*   Two independent volume boundary paths: $\tilde{\mathbf{p}}_1$ sampled by the unidirectional estimator and $\tilde{\mathbf{p}}_2$ sampled by the bidirectional estimator [S1].

**Outputs:**
*   The combined radiance estimate for the boundary path contribution [S1].

**Data Structures:**
*   Path structures containing vertices $\mathbf{p}_0, \dots, \mathbf{p}_K$, boundary vertex $\mathbf{p}_B$, and light source vertex $\mathbf{p}_{S0}$ [S1][S2, unofficial].
*   Probability density values $\text{pdf}_1(\tilde{\mathbf{p}})$ and $\text{pdf}_2(\tilde{\mathbf{p}})$ associated with each path [S1].

**Steps:**
1.  **Sample Unidirectional Path:** Generate a volume boundary path $\tilde{\mathbf{p}}_1$ using the unidirectional estimator. This involves starting from the detector, building a main path, and for a vertex $\mathbf{p}_D^n$, explicitly drawing a volume boundary vertex $\mathbf{p}_B^n \in \partial BV$ and connecting it to a light source via a subpath [S1].
2.  **Sample Bidirectional Path:** Generate a volume boundary path $\tilde{\mathbf{p}}_2$ using the bidirectional estimator. This involves first drawing the volume boundary vertex $\mathbf{p}_B$, then constructing two subpaths to connect it to the light source and the detector [Parent].
3.  **Compute PDFs:** Calculate the probability density $\text{pdf}_1(\tilde{\mathbf{p}}_1)$ for the unidirectional path and $\text{pdf}_2(\tilde{\mathbf{p}}_2)$ for the bidirectional path. Note that computing these PDFs is difficult when delta tracking is used to sample free-flight distances [S1].
4.  **Apply Balanced Heuristic:** Weight the contribution of each path using the balanced heuristic formula [S1]:

    $$
    \langle \hat{\mathbf{l}}_{\text{bnd}} \rangle_{\text{mis}} = \frac{f(\tilde{\mathbf{p}}_1) v_{\perp}(\mathbf{p}_{K1})}{\text{pdf}_1(\tilde{\mathbf{p}}_1) + \text{pdf}_2(\tilde{\mathbf{p}}_1)} + \frac{f(\tilde{\mathbf{p}}_2) v_{\perp}(\mathbf{p}_{K2})}{\text{pdf}_1(\tilde{\mathbf{p}}_2) + \text{pdf}_2(\tilde{\mathbf{p}}_2)}
    $$

    Where:
    *   $f(\tilde{\mathbf{p}})$ is the path throughput (product of phase functions, transmittance, and emission).
    *   $v_{\perp}(\mathbf{p}_{K})$ is the geometric term involving the perpendicular component of the boundary segment [S1].
    *   $\mathbf{p}_{K1}$ and $\mathbf{p}_{K2}$ are the final vertices of the respective paths. The paths can have varying lengths, so $K_1$ does not have to equal $K_2$ [S1].

**Parameters:**
*   The MIS estimator is used only for objects with homogeneous optical densities. For objects with spatially varying albedo, the bidirectional estimator alone is used [S1].

**Edge Cases and Pitfalls:**
*   **Delta Tracking:** Computing $\text{pdf}_1$ and $\text{pdf}_2$ is difficult when delta tracking is used to sample free-flight distances [S1].
*   **Occlusion in Direct Sampling:** When estimating the direct component, sampling the direction $\omega_B$ by first drawing a point on an area light can yield invalid (occluded) directions in complex illumination (e.g., image-based lighting). In such cases, directly sampling the direction $\omega_B$ is more efficient [S2, unofficial].

## Subtopics

### [Multiple Importance Sampling for Boundary Segments](8833-multiple-importance-sampling-for-boundary-segments.md)
This variant combines MIS with primary-sample-space guiding to estimate the boundary segment contribution in differentiable volume rendering. It addresses the issue where light-source sampling yields invalid directions due to occlusion. It differs from a full 5D MIS sampler by applying guiding separately to two lower-dimensional mappings (light-source and direct direction), thereby avoiding the computational expense of guiding a higher-dimensional space while maintaining unbiased estimation.

## Operating characteristics
*   **Complexity:** The method requires generating two independent paths (unidirectional and bidirectional) and computing their respective PDFs. The balancing step is $O(1)$ per path pair [S1].
*   **Strengths:** Reduces variance by combining the strengths of unidirectional (good for deep volumes) and bidirectional (good for boundary proximity) estimators [S1].
*   **Limitations:** Requires computing PDFs for both estimators, which is non-trivial with delta tracking [S1]. Restricted to homogeneous optical densities for the MIS combination; spatially varying albedo requires the bidirectional estimator alone [S1].
*   **When to use:** When rendering translucent objects with homogeneous optical densities where both unidirectional and bidirectional sampling strategies are viable [S1].

## Background
The method was introduced by Zhang et al. (2020) as part of a framework for differentiable volume rendering [S1][S2, unofficial]. It builds upon the unidirectional and bidirectional volume boundary estimators which support the full radiative transfer model including multiple scattering, anisotropic phase functions, and heterogeneity [S1]. The implementation in the source literature is based on the system developed by Yu et al. [YZN* 22] utilizing the Enzyme automatic differentiation framework [S1].

## Sources
[S1] PDF Efficient Path-Space Differentiable Volume Rendering With Respect To Shapes, https://www.shuangz.com/projects/psdr-vol2-egsr23/psdr-vol2-egsr23.pdf, fetched 2026-10-27
[S2] Variance-aware Differentiable Rendering, https://core.ac.uk/works/337282320, fetched 2026-11-02 (unofficial)
[S3] Path-Space Differentiable Rendering, https://core.ac.uk/download/541724175.pdf, fetched 2026-10-29 (unofficial)
