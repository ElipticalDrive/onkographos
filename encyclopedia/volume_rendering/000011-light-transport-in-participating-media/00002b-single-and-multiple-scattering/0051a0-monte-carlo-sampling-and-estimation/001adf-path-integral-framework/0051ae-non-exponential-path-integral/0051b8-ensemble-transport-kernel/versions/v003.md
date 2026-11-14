# Ensemble Transport Kernel

Also called: T(x, four transport functions

## Overview
The Ensemble Transport Kernel (ETK) is a method for estimating light transport in non-exponential stochastic media by replacing the classical transmittance term with an ensemble-averaged transport kernel $T(\mathbf{x}_i, \mathbf{x}_{i+1})$ [S1]. It solves the problem of computing the path integral expectation $\langle I_\mu \rangle$ by decorrelating path segments, allowing the ensemble average to be computed independently for each segment rather than over the entire path [S1]. Unlike the [Non-exponential transport kernel via macroscopic optical depth](14530-non-exponential-transport-kernel-via-macroscopic-optical-dep.md) which relies on macroscopic optical depth approximations, or [Ensemble-averaged transmittance via characteristic functions](14531-ensemble-averaged-transmittance-via-characteristic-functions.md) which uses characteristic functions, the ETK explicitly defines four distinct transport functions ($\text{pp}, \text{fp}, \text{pf}, \text{ff}$) based on the boundary configuration of segment endpoints [S1].

## How it works
**Inputs:**
*   Path vertices $\bar{\mathbf{x}} = \mathbf{x}_0, \dots, \mathbf{x}_k$.
*   Macroscopic extinction coefficient field $\sigma(\mathbf{x}) = \langle \sigma_\mu(\mathbf{x}) \rangle$ [S1].
*   Phase function $f(\mathbf{x})$ and geometry term $G(\mathbf{x}_i, \mathbf{x}_{i+1})$ [S1].

**Outputs:**
*   Estimated path throughput $\langle g_\mu(\bar{\mathbf{x}}) \rangle$ [S1].

**Data Structures:**
The method requires evaluating the transport kernel $T(\mathbf{x}_i, \mathbf{x}_{i+1})$. The kernel is not a single function but a set of four functions determined by the location of the endpoints relative to the medium volume $\mathcal{V}$ and its boundary $\partial\mathcal{V}$ [S1]:
1.  $\text{pp}(t)$: Interior-to-Interior ($\mathbf{x}_i, \mathbf{x}_{i+1} \in \mathcal{V}$).
2.  $\text{ff}(t)$: Boundary-to-Boundary ($\mathbf{x}_i, \mathbf{x}_{i+1} \in \partial\mathcal{V}$).
3.  $\text{pf}(t)$: Interior-to-Boundary ($\mathbf{x}_i \in \mathcal{V}, \mathbf{x}_{i+1} \in \partial\mathcal{V}$).
4.  $\text{fp}(t)$: Boundary-to-Interior ($\mathbf{x}_i \in \partial\mathcal{V}, \mathbf{x}_{i+1} \in \mathcal{V}$) [S1].

**Steps:**
1.  **Define $\Sigma(\mathbf{x})$:** For each vertex, determine the scaling factor $\Sigma(\mathbf{x})$ [S1]:
    $$ \Sigma(\mathbf{x}) = \begin{cases} \sigma(\mathbf{x}) & \text{if } \mathbf{x} \in \mathcal{V} \\ 1 & \text{if } \mathbf{x} \in \partial\mathcal{V} \end{cases} $$
2.  **Evaluate Transport Kernels:** For each segment $i$ from $\mathbf{x}_i$ to $\mathbf{x}_{i+1}$, select the appropriate kernel function based on endpoint locations [S1]:
    *   If $\mathbf{x}_i, \mathbf{x}_{i+1} \in \mathcal{V}$, use $T(\mathbf{x}_i, \mathbf{x}_{i+1}) = \text{pp}(\|\mathbf{x}_i - \mathbf{x}_{i+1}\|)$.
    *   If $\mathbf{x}_i, \mathbf{x}_{i+1} \in \partial\mathcal{V}$, use $T(\mathbf{x}_i, \mathbf{x}_{i+1}) = \text{ff}(\|\mathbf{x}_i - \mathbf{x}_{i+1}\|)$.
    *   If $\mathbf{x}_i \in \mathcal{V}, \mathbf{x}_{i+1} \in \partial\mathcal{V}$, use $T(\mathbf{x}_i, \mathbf{x}_{i+1}) = \text{pf}(\|\mathbf{x}_i - \mathbf{x}_{i+1}\|)$.
    *   If $\mathbf{x}_i \in \partial\mathcal{V}, \mathbf{x}_{i+1} \in \mathcal{V}$, use $T(\mathbf{x}_i, \mathbf{x}_{i+1}) = \text{fp}(\|\mathbf{x}_i - \mathbf{x}_{i+1}\|)$.
    *   *Note: The specific mathematical definitions of $\text{pp}, \text{fp}, \text{pf}, \text{ff}$ depend on the underlying stochastic medium model (e.g., Poisson Boolean model) and are derived from the ensemble average of the classical free-flight PDFs [S1].*
3.  **Compute Throughput:** Calculate the estimated path throughput using the product of phase functions, geometry terms, and transport kernels [S1]:
    $$ \langle g_\mu(\bar{\mathbf{x}}) \rangle \approx \left[ \prod_{i=1}^{k-1} f(\mathbf{x}_i) \Sigma(\mathbf{x}_i) \right] \left[ \prod_{i=0}^{k-1} G(\mathbf{x}_i, \mathbf{x}_{i+1}) \right] \left[ \prod_{i=0}^{k-1} T(\mathbf{x}_i, \mathbf{x}_{i+1}) \right] $$

**Pitfalls:**
*   **Non-Reciprocity:** The standard ETK is not reciprocal, i.e., $T(\mathbf{x}, \mathbf{y}) \neq T(\mathbf{y}, \mathbf{x})$ in the general case [S1]. This occurs because free-flight PDFs (like $\text{pp}$) are not generally reciprocal, and swapping endpoints changes the case statement (e.g., $\text{fp}$ vs $\text{pf}$) [S1]. This can cause issues in bidirectional rendering methods that rely on reciprocity.

## Subtopics

### Reciprocal Transport Kernel
The Reciprocal Transport Kernel ($\tilde{T}$) is a variant designed to address the non-reciprocity of the standard ETK [S1]. It is defined such that $\tilde{T}(\mathbf{x}, \mathbf{y}) = T(\mathbf{x}, \mathbf{y}) / \Sigma(\mathbf{y})$ [S1]. This formulation ensures reciprocity because it effectively evaluates only averages of transmittance for all four possible configurations of a segment, and classical transmittance averages are reciprocal [S1]. The final path throughput using the reciprocal kernel is:
$$ \langle g_\mu(\bar{\mathbf{x}}) \rangle \approx \left[ \prod_{i=1}^{k-1} f(\mathbf{x}_i) \Sigma(\mathbf{x}_i) \right] \left[ \prod_{i=0}^{k-1} \tilde{T}(\mathbf{x}_i, \mathbf{x}_{i+1}) G(\mathbf{x}_i, \mathbf{x}_{i+1}) \right] $$
[S1]. This makes it straightforward to simulate non-exponential transport with existing bidirectional rendering methods, as the only change from the exponential case is replacing transmittance $\text{Tr}$ with $\tilde{T}$ [S1].

## Operating characteristics
*   **Complexity:** The method reduces the complexity of ensemble averaging from path-wide to segment-wise, allowing independent computation for each segment [S1].
*   **Strengths:** It explicitly handles heterogeneity in the most general form, supporting varying macroscopic density and correlation between scattering parameters [S1]. When using the reciprocal variant, it integrates easily into existing bidirectional renderers [S1].
*   **Limitations:** The standard kernel is non-reciprocal, requiring the modified $\tilde{T}$ formulation for bidirectional methods [S1]. The accuracy depends on the segment decorrelation assumption, which approximates the ensemble average of the product of segment terms by the product of ensemble averages [S1].

## Background
The Ensemble Transport Kernel was introduced in the paper "A radiative transfer framework for non-exponential media" [S1]. It is based on the General Boltzmann Equation of Larsen and Vasques [S1]. The method decorrelates path segments to allow independent ensemble averaging, moving the average into the product of segment terms [S1]. It is used in volume rendering for participating media that do not follow exponential transmittance laws.

## Sources
[S1] A radiative transfer framework for non-exponential media, https://doi.org/10.1145/3272127.3275103, fetched 2026-11-08
