# Differential Edge Sampling

Also called: Boundary Path Integral Estimation via Edge Sampling, Differential Spherical Integral / Edge Sampling, LADL18, Li et al, Monte Carlo edge sampling, Stochastic Monte Carlo Gradient Estimation

## Overview
Differential Edge Sampling is a method for computing the gradient of physically-based rendered images with respect to arbitrary scene parameters (camera, lights, materials, geometry) by explicitly sampling visibility discontinuities [S1, S2, unofficial]. It solves the problem that standard Monte Carlo estimators fail to capture derivatives at boundaries where visibility changes abruptly, such as object silhouettes and occlusion edges [S1, unofficial]. Unlike its sibling [Differential Vertex Sampling with MIS](6338-differential-vertex-sampling-with-mis.md), which focuses on vertex-based contributions, this method decomposes the rendering derivative into a smooth interior integral (estimated via automatic differentiation) and a boundary integral (estimated by sampling triangle edges) [S1, S2, unofficial].

## How it works
The method computes the gradient of the pixel filter’s integral by splitting the integrand into smooth and discontinuous regions [S1, unofficial].

**Inputs and Outputs:**
- **Inputs:** Scene geometry (triangle meshes), material properties, light sources, camera parameters, scene parameters $\theta$ to differentiate with respect to.
- **Outputs:** Rendered image value and its gradient $\nabla_\theta I$.

**Data Structures:**
- **Mesh Representation:** Triangle meshes where discontinuities are identified at triangle edges [S2, unofficial].
- **Edge Hierarchy:** A spatial hierarchy (e.g., bounding volume hierarchy) used to efficiently reject interior edges and find silhouette edges for secondary boundaries [S2, unofficial].

**Steps:**
1. **Decomposition:** Split the rendering integral into smooth parts and discontinuous parts [S1, unofficial].
2. **Smooth Region Estimation:** For smooth parts of the integrand, employ traditional area sampling combined with automatic differentiation to estimate the gradient [S1, unofficial].
3. **Discontinuous Region Estimation (Edge Sampling):** For discontinuous parts, explicitly sample edges to estimate the boundary integral contribution [S1, S2, unofficial].
   - **Primary Boundaries:** For boundaries directly observed by the detector (camera), project and clip all meshes into screen space and sample from the visible edges [S2, unofficial].
   - **Secondary Boundaries:** For boundaries occurring at light bounces, the silhouette depends on the shading point. Use an importance sampling scheme with a hierarchy to quickly reject interior edges and find the relevant silhouette, avoiding per-bounce projection [S2, unofficial].
4. **Combination:** Combine the gradient estimates from smooth and discontinuous regions [S1, unofficial].

**Formulas:**
The rendering equation derivative is decomposed. For mesh scenes, discontinuities occur at triangle edges [S2, unofficial]. If shading is smooth, the focus narrows to silhouette edges [S2, unofficial]. The complete equation for differentiating the radiative transfer equation (extended by Zhang et al. for volumes) involves derivatives of the transport operator $K_T$, collision operator $K_C$, and source term $Q$ [S2, unofficial]. Due to complexity, the specific formula is referenced as Eq. 32 in the original paper [S2, unofficial].

**Parameters:**
- **Sampling Strategy:** Importance sampling for secondary boundaries using a hierarchy [S2, unofficial].
- **Assumptions:** Meshes do not have interpenetration, no point light sources, no perfectly specular surfaces, and the scene is static [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Interpenetration:** The method assumes meshes do not interpenetrate [S1, unofficial].
- **Light Sources:** Point light sources are not supported [S1, unofficial].
- **Specular Surfaces:** Perfectly specular surfaces are not supported [S1, unofficial].
- **Dynamic Scenes:** The scene must be static [S1, unofficial].
- **Variance:** Large variance in estimated gradients due to the difficulty of finding all object edges and sampling them [S1, unofficial].
- **Performance:** Rendering speed is slow [S1, unofficial].
- **Secondary Boundaries:** Inefficient to project for every light bounce due to shading-point-dependent silhouettes [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Slow rendering speed due to the computational cost of finding all object edges and sampling them [S1, unofficial].
- **Memory Complexity:** Requires storage for mesh geometry and edge hierarchies [S2, unofficial].
- **Variance:** Large variance in estimated gradients [S1, unofficial].
- **Strengths:** First method to achieve unbiased differentiation of the rendering equation for mesh scenes by explicitly handling discontinuities [S2, unofficial].
- **Limitations:** Limited to static scenes, no point lights, no perfect speculars, and high computational cost [S1, unofficial].
- **When to Use:** When unbiased gradients with respect to geometry and camera parameters are required for mesh-based scenes, and computational resources allow for high-variance, low-speed estimation.
- **When Not to Use:** For dynamic scenes, scenes with point lights or perfect speculars, or when real-time performance is required.

## Background
Differential Edge Sampling was introduced by Li et al. in 2018 [LADL18] as the first work to compute derivatives of scalar functions over physically-based rendered images with respect to arbitrary input parameters using a stochastic Monte Carlo ray tracing approach [S1, unofficial]. It achieved unbiased differentiation of the rendering equation [Kaj86] by directly sampling boundary discontinuities [S2, unofficial]. Zhang et al. [ZWZ* 19] later extended this method to support participating media by differentiating the radiative transfer equation [S2, unofficial]. This work laid the foundation for path-space differentiable rendering (PSDR) [S2, unofficial].

## Sources
[S1] Differentiable Rendering: A Survey, https://doi.org/10.48550/arxiv.2006.12057, fetched 2026-10-26 (unofficial)
[S2] A Survey on Physics-based Differentiable Rendering - arXiv.org, https://arxiv.org/pdf/2504.01402, fetched 2026-10-29 (unofficial)
