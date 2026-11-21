# Ensemble Transport Kernel

Also called: T(x, four transport functions

## Overview
The Ensemble Transport Kernel (ETK) is a method for estimating light transport in non-exponential stochastic media by replacing the classical transmittance term with an ensemble-averaged transport kernel $T(\mathbf{x}_i, \mathbf{x}_{i+1})$ [S1]. It solves the problem of computing the path integral expectation $\langle I_\mu \rangle$ by decorrelating path segments, allowing the ensemble average to be computed independently for each segment rather than over the entire path [S1]. Unlike the [Non-exponential transport kernel via macroscopic optical depth](14530-non-exponential-transport-kernel-via-macroscopic-optical-dep.md) which relies on macroscopic optical depth approximations, or [Ensemble-averaged transmittance via characteristic functions](14531-ensemble-averaged-transmittance-via-characteristic-functions.md) which uses characteristic functions, the ETK explicitly defines four distinct transport functions ($\text{pp}, \text{fp}, \text{pf}, \text{ff}$) based on the boundary configuration of segment endpoints [S1].

## How it works
**Inputs:**
*   Path vertices $\bar{\mathbf{x}} = \mathbf{x}_0, \dots, \mathbf{x}_k$ [S1].
*   Macroscopic extinction coefficient field $\sigma(\mathbf{x}) = \langle \sigma_\mu(\mathbf{x}) \rangle$ [S1].
*   Phase function $f(\mathbf{x})$ and geometry term $G(\mathbf{x}_i, \mathbf{x}_{i+1})$ [S1].

**Outputs:**
*   Estimated path throughput $\langle g_\mu(\bar{\mathbf{x}}) \rangle$ [S1].

**Data Structures:**
The method requires evaluating the transport kernel $T(\mathbf{x}_i, \mathbf{x}_{i+1})$ [S1]. The kernel is not a single function but a set of four functions determined by the location of the endpoints relative to the medium volume $\mathcal{V}$ and its boundary $\partial\mathcal{V}$ [S1]:
1.  $\text{pp}(t)$: Interior-to-Interior ($\mathbf{x}_i, \mathbf{x}_{i+1} \in \mathcal{V}$) [S1].
2.  $\text{ff}(t)$: Boundary-to-Boundary ($\mathbf{x}_i, \mathbf{x}_{i+1} \in \partial\mathcal{V}$) [S1].
3.  $\text{pf}(t)$: Interior-to-Boundary ($\mathbf{x}_i \in \mathcal{V}, \mathbf{x}_{i+1} \in \partial\mathcal{V}$) [S1].
4.  $\text{fp}(t)$: Boundary-to-Interior ($\mathbf{x}_i \in \partial\mathcal{V}, \mathbf{x}_{i+1} \in \mathcal{V}$) [S1].

**Steps:**
1.  **Define $\Sigma(\mathbf{x})$:** For each vertex, determine the scaling factor $\Sigma(\mathbf{x})$ [S1]:
    $$ \Sigma(\mathbf{x}) = \begin{cases} \sigma(\mathbf{x}) & \text{if } \mathbf{x} \in \mathcal{V} \\ 1 & \text{if } \mathbf{x} \in \partial\mathcal{V} \end{cases} $$
2.  **Evaluate Transport Kernels:** For each segment $i$ from $\mathbf{x}_i$ to $\mathbf{x}_{i+1}$, select the appropriate kernel function based on endpoint locations [S1]:
    *   If $\mathbf{x}_i, \mathbf{x}_{i+1} \in \mathcal{V}$, use $T(\mathbf{x}_i, \mathbf{x}_{i+1}) = \text{pp}(\|\mathbf{x}_i - \mathbf{x}_{i+1}\|)$ [S1].
    *   If $\mathbf{x}_i, \mathbf{x}_{i+1} \in \partial\mathcal{V}$, use $T(\mathbf{x}_i, \mathbf{x}_{i+1}) = \text{ff}(\|\mathbf{x}_i - \mathbf{x}_{i+1}\|)$ [S1].
    *   If $\mathbf{x}_i \in \mathcal{V}, \mathbf{x}_{i+1} \in \partial\mathcal{V}$, use $T(\mathbf{x}_i, \mathbf{x}_{i+1}) = \text{pf}(\|\mathbf{x}_i - \mathbf{x}_{i+1}\|)$ [S1].
    *   If $\mathbf{x}_i \in \partial\mathcal{V}, \mathbf{x}_{i+1} \in \mathcal{V}$, use $T(\mathbf{x}_i, \mathbf{x}_{i+1}) = \text{fp}(\|\mathbf{x}_i - \mathbf{x}_{i+1}\|)$ [S1].
    *   *Note: The specific mathematical definitions of $\text{pp}, \text{fp}, \text{pf}, \text{ff}$ depend on the underlying stochastic medium model (e.g., Poisson Boolean model) and are derived from the ensemble average of the classical free-flight PDFs [S1].*
3.  **Compute Throughput:** Calculate the estimated path throughput using the product of phase functions, geometry terms, and transport kernels [S1]:
    $$ \langle g_\mu(\bar{\mathbf{x}}) \rangle \approx \left[ \prod_{i=1}^{k-1} f(\mathbf{x}_i) \Sigma(\mathbf{x}_i) \right] \left[ \prod_{i=0}^{k-1} G(\mathbf{x}_i, \mathbf{x}_{i+1}) \right] \left[ \prod_{i=0}^{k-1} T(\mathbf{x}_i, \mathbf{x}_{i+1}) \right] $$

**Parameters:**
*   Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Non-Reciprocity:** The standard ETK is not reciprocal, i.e., $T(\mathbf{x}, \mathbf{y}) \neq T(\mathbf{y}, \mathbf{x})$ in the general case [S1]. This occurs because free-flight PDFs (like $\text{pp}$) are not generally reciprocal, and swapping endpoints changes the case statement (e.g., $\text{fp}$ vs $\text{pf}$) [S1]. This can cause issues in bidirectional rendering methods that rely on reciprocity [S1].

## Subtopics

### Reciprocal Transport Kernel
A variant that modifies the standard kernel to satisfy reciprocity by dividing by the boundary scaling factor $\Sigma(\mathbf{y})$ [S1]. It sets itself apart by evaluating only averages of transmittance for all four segment configurations, guaranteeing $T(\mathbf{x}, \mathbf{y}) = T(\mathbf{y}, \mathbf{x})$ [S1]. Choose this variant when implementing bidirectional rendering methods that require symmetric kernel evaluation [S1]. It differs from the exponential case only by replacing transmittance with $\tilde{T}$, allowing direct integration into existing renderers [S1].

## Operating characteristics
*   **Time and Memory Complexity:** Reduces ensemble averaging from path-wide to segment-wise, allowing independent computation per segment [S1]. Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Explicitly handles general heterogeneity, supporting varying macroscopic density and scattering correlations [S1]. Integrates easily into existing bidirectional renderers when using the reciprocal variant [S1].
*   **Limitations:** Standard kernel lacks reciprocity, requiring the modified $\tilde{T}$ formulation for bidirectional methods [S1]. Accuracy relies on the segment decorrelation assumption [S1].
*   **When to use:** Non-exponential participating media where segment decorrelation holds and bidirectional methods are needed (use reciprocal variant) [S1].
*   **When not to use:** Scenarios requiring strict reciprocity without the $\tilde{T}$ modification, or where segment decorrelation is invalid [S1].

## Background
Introduced in "A radiative transfer framework for non-exponential media" [S1]. It builds on the General Boltzmann Equation of Larsen and Vasques [S1]. The method decorrelates path segments to enable independent ensemble averaging [S1]. It is used in volume rendering for participating media that do not follow exponential transmittance laws [S1].

## Sources
[S1] A radiative transfer framework for non-exponential media, https://doi.org/10.1145/3272127.3275103, fetched 2026-11-08
