# Selecting Occluders

Also called: Occluder Selection, Occluder Identification

## Overview
Selecting Occluders is a technique used in differentiable volume rendering to identify the specific object responsible for a discontinuity (silhouette) when multiple surface intersections are detected within an integrand's support [S1]. It solves the problem of determining which intersection point's motion should drive the reparameterization of the rendering integral, ensuring that gradients flow correctly through the occluding geometry [S1]. Unlike methods that explicitly search for silhouette edges, this approach relies on local geometric heuristics derived from ray intersections, making it robust for high-depth complexity scenes [S1].

## How it works
The method operates under the assumption that discontinuities in the integrand consist of the silhouette of a single object masking another object, a part of the same object, or the background [S1]. The process identifies the occluding surface among multiple intersection points to enable accurate gradient estimation.

**Inputs:**
*   A set of surface intersection points $\{P_i\}$ obtained by sampling the integrand.
*   Object identifiers (IDs) associated with each intersection point.
*   Surface normals $\{N_i\}$ at each intersection point.

**Outputs:**
*   A single selected point $P_{sel}$ belonging to the occluding object.

**Data Structures:**
*   Intersection List: A list of tuples $(P_i, N_i, ID_i)$, where $P_i$ is the 3D position, $N_i$ is the normalized surface normal, and $ID_i$ is the unique identifier of the geometry object.

**Steps:**
1.  **Sample Intersections:** Sample the integrand using a fixed number of rays (typically 4) within the support of the integrand to yield one or more surface intersections [S1].
2.  **Identify Distinct Objects:** Iterate through the intersection points. Select two points, $P_0$ and $P_1$, that belong to different objects (i.e., $ID_0 \neq ID_1$) [S1]. If all intersections belong to the same object, the method may default to the first point or handle it as a self-occlusion case [G].
3.  **Construct Tangent Planes:** Construct two planes, $\Pi_0$ and $\Pi_1$, tangent to the surfaces at $P_0$ and $P_1$ respectively. The plane $\Pi_k$ is defined by the point $P_k$ and the normal $N_k$.
4.  **Evaluate Masking:** Determine which object occludes the other by analyzing the relative positions of the intersection points with respect to the tangent planes [S1]. Specifically, check if $P_1$ lies "behind" $\Pi_0$ (from the viewer's perspective) or if $P_0$ lies "behind" $\Pi_1$.
    *   If $P_1$ is occluded by the surface at $P_0$, select $P_0$ as the occluder.
    *   If $P_0$ is occluded by the surface at $P_1$, select $P_1$ as the occluder.
    *   If neither is clearly occluded by the other in this first-order approximation, additional heuristics or higher-order checks may be required [G].
5.  **Select Occluder:** Return the point corresponding to the occluding object as $P_{sel}$ [S1].

**Formulas:**
The selection logic relies on the dot product between the view direction (or the vector from the camera to the point) and the surface normal to determine visibility and occlusion relationships. Let $V$ be the view direction. The condition for $P_1$ being occluded by $P_0$ can be approximated by checking if the signed distance of $P_1$ to plane $\Pi_0$ is negative (assuming normals point outward) and $P_0$ is visible.

**Parameters:**
*   Number of samples: 4 [S1].

**Edge Cases and Pitfalls:**
*   **First-Order Estimation:** The method uses a first-order estimation of occlusion based on tangent planes. This may be insufficient for highly complex or curved surfaces but is considered robust for most practical gradient estimations [S1].
*   **Interpenetrating Objects:** The technique can handle interpenetrating objects that create implicit edges (edges not explicitly represented in the geometry), which is a benefit over previous work [S1].
*   **Single Object Sampling:** If the sampling only hits one object within the support, it may miss discontinuities from other objects, resulting in bias [S1].

## Subtopics

### Rotation Determination via Ray Tracing
A method to find the rotation $R(\theta)$ by intersecting a small number of rays within the integrand's support against scene geometry to estimate the displacement of discontinuities, avoiding explicit silhouette search [S1]. It intersects a small number of rays (4 in implementation) within the support of the integrand against scene geometry [S1]. It uses distance and surface normal information from intersections to apply a heuristic selecting a single point likely belonging to an occluder [S1]. It builds a differentiable rotation matrix $R(\theta)$ satisfying $R(\theta_0) \omega = \omega$ for all $\omega \in S^2$ and $\frac{\partial}{\partial \theta} R(\theta) \omega_{P0} = \frac{\partial}{\partial \theta} \omega_P(\theta)$, where $\omega_{P0}$ is the projection of the selected point at $\theta_0$ [S1]. A pitfall is that sampling only one object within the support can miss discontinuities from other objects, resulting in bias, and the number of rays impacts this probability [S1]. Unlike previous work, this approach does not explicitly search for silhouette edges, scaling better to large numbers of edges and high depth complexity [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of intersection points per integrand sample. The core operation involves a constant number of ray intersections (4) and simple geometric comparisons (dot products) [S1].
*   **Memory Complexity:** Minimal, requiring storage for the intersection points, normals, and IDs for the current sample [S1].
*   **Strengths:** Robust for most practical gradient estimations; handles interpenetrating objects with implicit edges; does not require explicit silhouette edge search, scaling well to high depth complexity [S1].
*   **Limitations:** First-order estimation may fail on highly complex surfaces; bias can occur if sampling misses other objects within the integrand support [S1].
*   **When to use:** When differentiable rendering requires accurate gradient flow through occluding silhouettes in complex scenes with high depth complexity [S1].
*   **When not to use:** When extremely high precision in occlusion boundaries is required for highly curved or complex self-occluding geometries where first-order approximations are insufficient [S1].

## Background
The technique originates from the work on reparameterizing discontinuous integrands for differentiable rendering [S1]. It was developed to address the challenge of computing gradients through silhouette discontinuities without the computational cost of explicit silhouette edge detection. The method is part of a broader framework for differentiable volume rendering that utilizes convolution kernels and change of variables [S1].

## Sources
[S1] Reparameterizing discontinuous integrands for differentiable rendering, https://doi.org/10.1145/3355089.3356510, fetched 2026-10-26
