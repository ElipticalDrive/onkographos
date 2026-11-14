# Ensemble Transport Kernel

Also called: T(x, four transport functions

## Overview
The Ensemble Transport Kernel (ETK) is a method for computing light transport in non-exponential participating media by replacing the classical transmittance term with an ensemble-averaged kernel that accounts for spatial correlations between scatterers [S1]. It builds on the [Non-exponential path integral](14524-non-exponential-path-integral.md) by approximating the ensemble average of the path throughput as a product of independent segment kernels, thereby decorrelating path segments [S1]. Unlike its siblings, it explicitly defines four distinct transport functions based on the boundary configuration of segment endpoints [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A path $\bar{\mathbf{x}} = \mathbf{x}_0 \cdots \mathbf{x}_k$; the medium boundary $\partial\mathcal{V}$ and interior $\mathcal{V}$; the realization space $\mathcal{R}$; phase function $f$; geometric term $G$.
*   **Output:** The estimated ensemble-averaged path throughput $\langle g_\mu(\bar{\mathbf{x}}) \rangle$.

**Data Structures**
The method relies on the classification of segment endpoints relative to the medium:
*   $\mathbf{x} \in \mathcal{V}$: Interior point.
*   $\mathbf{x} \in \partial\mathcal{V}$: Boundary point.
*   Four transport functions $T_{\mu}$ corresponding to pairs: $(\partial, \partial)$, $(\mathcal{V}, \mathcal{V})$, $(\partial, \mathcal{V})$, $(\mathcal{V}, \partial)$ [S1].

**Formulas**
The classical transmittance and free-flight PDF for a single realization $\mu$ are [S1]:
$$ \text{Tr}_{\mu}(\mathbf{x}, \mathbf{x}_t) = e^{-\tau_{\mu}(\mathbf{x}, \mathbf{x}_t)}, \quad p_{\mu}(\mathbf{x}, \mathbf{x}_t) = e^{-\tau_{\mu}(\mathbf{x}, \mathbf{x}_t)} \sigma_{\mu}(\mathbf{x}_t) $$
where $\tau_{\mu}(\mathbf{x}, \mathbf{x}_t) = \int_0^t \sigma_{\mu}(\mathbf{x}_s) ds$ [S1].

The measurement $I_{\mu}$ for a length-$k$ path is [S1]:
$$ I_{\mu} = \int_{\mathcal{P}} L_e(\mathbf{x}_0, \mathbf{x}_1) g_{\mu}(\bar{\mathbf{x}}) W(\mathbf{x}_{k-1}, \mathbf{x}_k) \, d\bar{\mathbf{x}} $$
with path throughput [S1]:
$$ g_{\mu}(\bar{\mathbf{x}}) = \left[ \prod_{i=1}^{k-1} f_{\mu}(\mathbf{x}_i) \Sigma_{\mu}(\mathbf{x}_i) \right] \left[ \prod_{i=0}^{k-1} \text{Tr}_{\mu}(\mathbf{x}_i, \mathbf{x}_{i+1}) G(\mathbf{x}_i, \mathbf{x}_{i+1}) \right] $$
where $\Sigma_{\mu}(\mathbf{x}) = \sigma_{\mu}(\mathbf{x})$ if $\mathbf{x} \in \mathcal{V}$ and $1$ if $\mathbf{x} \in \partial\mathcal{V}$ [S1].

The ensemble average $\langle \cdot \rangle$ is defined over the space of realizations $\mathcal{R}$ [S1]:
$$ \langle f_{\mu}(\mathbf{x}) \rangle = \int_{\mathcal{R}} f_{\mu}(\mathbf{x}) \, dP\{\mu\} $$

Two simplifying assumptions are applied [S1]:
1.  Phase function and albedo are uncorrelated with $\mu$, allowing them to move outside the ensemble average.
2.  Segment decorrelation: The ensemble average of the product of segment terms is approximated by the product of the ensemble averages [S1]:
    $$ \left\langle \prod_{i=0}^{k-1} \text{Tr}_{\mu}(\mathbf{x}_i, \mathbf{x}_{i+1}) \Sigma_{\mu}(\mathbf{x}_{i+1}) \right\rangle \approx \prod_{i=0}^{k-1} \langle T_{\mu}(\mathbf{x}_i, \mathbf{x}_{i+1}) \rangle $$
    where $T_{\mu}$ is a transport kernel resembling $\text{Tr} \cdot \Sigma$ [S1].

The final approximation for the path throughput is [S1]:
$$ \langle g_\mu(\bar{\mathbf{x}}) \rangle \approx \left[ \prod_{i=1}^{k-1} f(\mathbf{x}_i) \Sigma(\mathbf{x}_i) \right] \left[ \prod_{i=0}^{k-1} G(\mathbf{x}_i, \mathbf{x}_{i+1}) \right] \left[ \prod_{i=0}^{k-1} T(\mathbf{x}_i, \mathbf{x}_{i+1}) \right] $$

**Steps**
1.  Classify each vertex $\mathbf{x}_i$ of the path as either interior ($\mathcal{V}$) or boundary ($\partial\mathcal{V}$).
2.  For each segment $i$ from $0$ to $k-1$, determine the endpoint configuration pair $(\mathbf{x}_i, \mathbf{x}_{i+1})$.
3.  Evaluate the appropriate transport kernel $T(\mathbf{x}_i, \mathbf{x}_{i+1})$ based on the configuration:
    *   Boundary-to-Boundary ($\partial, \partial$)
    *   Interior-to-Interior ($\mathcal{V}, \mathcal{V}$)
    *   Mixed ($\partial, \mathcal{V}$ or $\mathcal{V}, \partial$)
4.  Compute the geometric term $G(\mathbf{x}_i, \mathbf{x}_{i+1})$ for each segment.
5.  Compute the vertex term $f(\mathbf{x}_i) \Sigma(\mathbf{x}_i)$ for internal vertices $i=1 \dots k-1$.
6.  Multiply all terms to obtain the estimated throughput.

**Pitfalls**
The initial transport kernel $T$ is not reciprocal, meaning $T(\mathbf{x}, \mathbf{y}) \neq T(\mathbf{y}, \mathbf{x})$ in the general case [S1]. This occurs because free-flight PDFs are not generally reciprocal, and swapping endpoints changes the case statement (e.g., $fp$ vs $pf$) [S1].

## Subtopics

### Reciprocal Transport Kernel
The Reciprocal Transport Kernel ($\tilde{T}$) addresses the non-reciprocity of the standard ETK by evaluating only averages of transmittance for all four possible configurations of a segment [S1]. It ensures reciprocity because classical transmittance and its averages are reciprocal [S1]. For segments with both endpoints in the medium, it evaluates $pp(\mathbf{x}, \mathbf{y})/\sigma(\mathbf{y})$ [S1]. The final reciprocal path throughput is [S1]:
$$ \langle g_\mu(\bar{\mathbf{x}}) \rangle \approx \left[ \prod_{i=1}^{k-1} f(\mathbf{x}_i) \Sigma(\mathbf{x}_i) \right] \left[ \prod_{i=0}^{k-1} \tilde{T}(\mathbf{x}_i, \mathbf{x}_{i+1}) G(\mathbf{x}_i, \mathbf{x}_{i+1}) \right] $$
This formulation differs from the exponential case only by the replacement of transmittance $\text{Tr}$ with $\tilde{T}$, making it straightforward to simulate with existing bidirectional rendering methods [S1].

## Operating characteristics
Not given in the sources.

## Background
The Ensemble Transport Kernel was introduced in the paper "A radiative transfer framework for non-exponential media" [S1]. It is derived from the general Boltzmann Equation of Larsen and Vasques [S1], specifically in cases where all path vertices lie in the medium, reducing the transport kernel to the particle-to-particle free-flight PDF $pp$ used in the GBE [S1].

## Sources
[S1] A radiative transfer framework for non-exponential media, https://doi.org/10.1145/3272127.3275103, fetched 2026-11-08
