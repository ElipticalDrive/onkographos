# Non-exponential path integral

Also called: Correlated heterogeneous path integral, Stochastic medium ensemble average framework

## Overview
This family of methods models light transport in participating media with spatial correlations (non-exponential decay) by computing the ensemble average of the classical path throughput over all realizations of a stochastic medium, rather than assuming independent random scatterers [S1]. It differs from classical exponential models by explicitly accounting for the statistical dependence of scattering events, and from null-scattering methods by retaining the physical density field $\sigma(\mathbf{x})$ while correcting the transmittance statistics [S1].

## Shared principles
The core mechanism replaces the classical path integral measurement $I_\mu$ with its ensemble average $\langle I_\mu \rangle$ over the space of realizations $\mathcal{R}$ [S1].

**1. Stochastic Medium Model**
Each realization $\mu$ is modeled as a heterogeneous classical medium with a micro-scale extinction coefficient field $\sigma_\mu(\mathbf{x})$ [S1]. The macroscopic extinction coefficient is $\sigma(\mathbf{x}) = \langle \sigma_\mu(\mathbf{x}) \rangle$ [S1].

**2. Classical Segment Terms**
For a single realization $\mu$, the transmittance and free-flight PDF for a segment from $\mathbf{x}$ to $\mathbf{x}_t$ are [S1]:
$$ \text{Tr}_{\mu}(\mathbf{x}, \mathbf{x}_t) = e^{-\tau_{\mu}(\mathbf{x}, \mathbf{x}_t)} $$
$$ p_{\mu}(\mathbf{x}, \mathbf{x}_t) = e^{-\tau_{\mu}(\mathbf{x}, \mathbf{x}_t)} \sigma_{\mu}(\mathbf{x}_t) $$
where the optical depth is $\tau_{\mu}(\mathbf{x}, \mathbf{x}_t) = \int_0^t \sigma_{\mu}(\mathbf{x}_s) ds$ [S1].

**3. Path Throughput**
The measurement for a length-$k$ path $\bar{\mathbf{x}} = \mathbf{x}_0 \cdots \mathbf{x}_k$ in realization $\mu$ is [S1]:
$$ I_{\mu} = \int_{\mathcal{P}} L_e(\mathbf{x}_0, \mathbf{x}_1) g_{\mu}(\bar{\mathbf{x}}) W(\mathbf{x}_{k-1}, \mathbf{x}_k) \, d\bar{\mathbf{x}} $$
with path throughput [S1]:
$$ g_{\mu}(\bar{\mathbf{x}}) = \left[ \prod_{i=1}^{k-1} f_{\mu}(\mathbf{x}_i) \Sigma_{\mu}(\mathbf{x}_i) \right] \left[ \prod_{i=0}^{k-1} \text{Tr}_{\mu}(\mathbf{x}_i, \mathbf{x}_{i+1}) G(\mathbf{x}_i, \mathbf{x}_{i+1}) \right] $$
where:
*   $\Sigma_{\mu}(\mathbf{x}) = \sigma_{\mu}(\mathbf{x})$ if $\mathbf{x} \in \mathcal{V}$ (interior), and $1$ if $\mathbf{x} \in \partial\mathcal{V}$ (boundary) [S1].
*   $f_{\mu}(\mathbf{x}_i)$ is the phase function (or BRDF if on boundary) [S1].
*   $G(\mathbf{x}_i, \mathbf{x}_{i+1}) = \frac{D(\mathbf{x}_i)V(\mathbf{x}_i, \mathbf{x}_{i+1})D(\mathbf{x}_{i+1})}{\|\mathbf{x}_i - \mathbf{x}_{i+1}\|^2}$ is the geometry term, with $D(\mathbf{x}) = 1$ in $\mathcal{V}$ and $\cos \theta$ on $\partial\mathcal{V}$ [S1].

**4. Ensemble Averaging Assumptions**
Two simplifying assumptions allow the separation of the ensemble average [S1]:
1.  **Uncorrelated Phase/Albedo:** The phase function $f$ and albedo $\alpha$ are uncorrelated with the realization $\mu$. By linearity, they move outside the average:
    $$ \langle g_{\mu}(\bar{\mathbf{x}}) \rangle \approx \prod_{i=1}^{k-1} f(\mathbf{x}_i) \prod_{i=0}^{k-1} G(\mathbf{x}_i, \mathbf{x}_{i+1}) \left\langle \prod_{i=0}^{k-1} \text{Tr}_{\mu}(\mathbf{x}_i, \mathbf{x}_{i+1}) \Sigma_{\mu}(\mathbf{x}_{i+1}) \right\rangle $$
2.  **Segment Decorrelation:** The ensemble average of the product of segment terms is approximated by the product of the ensemble averages [S1]:
    $$ \left\langle \prod_{i=0}^{k-1} \text{Tr}_{\mu}(\mathbf{x}_i, \mathbf{x}_{i+1}) \Sigma_{\mu}(\mathbf{x}_{i+1}) \right\rangle \approx \prod_{i=0}^{k-1} \langle T_{\mu}(\mathbf{x}_i, \mathbf{x}_{i+1}) \rangle $$
    where $T_{\mu}$ is a transport kernel resembling $\text{Tr} \cdot \Sigma$ [S1].

**5. Transport Kernel Definition**
The final estimated throughput is [S1]:
$$ \langle g_\mu(\bar{\mathbf{x}}) \rangle \approx \left[ \prod_{i=1}^{k-1} f(\mathbf{x}_i) \Sigma(\mathbf{x}_i) \right] \left[ \prod_{i=0}^{k-1} G(\mathbf{x}_i, \mathbf{x}_{i+1}) \right] \left[ \prod_{i=0}^{k-1} T(\mathbf{x}_i, \mathbf{x}_{i+1}) \right] $$
The kernel $T(\mathbf{x}_i, \mathbf{x}_{i+1})$ depends on the boundary configuration of the endpoints:
*   $\text{ff}(t)$: Boundary-to-Boundary ($\partial, \partial$)
*   $\text{pp}(t)$: Interior-to-Interior ($\mathcal{V}, \mathcal{V}$)
*   $\text{pf}(t)$ or $\text{fp}(t)$: Mixed ($\partial, \mathcal{V}$ or $\mathcal{V}, \partial$) [S1].

## Subtopics

### [Non-exponential transport kernel via macroscopic optical depth](14530-non-exponential-transport-kernel-via-macroscopic-optical-dep.md)
This method models light transport by decomposing the medium into an explicit 3D macroscopic density field $\sigma(\mathbf{x})$ and a 1D base transport kernel $T_{\text{base}}(t)$. It solves the difficulty of numerically evaluating the ensemble-averaged path integral by reducing the transport kernel to a function of macroscopic optical depth $\tau_i$. It differs from fully homogenized models by allowing arbitrary spatial distribution of correlations, and from general heterogeneous models by assuming statistical homogeneity of the underlying correlations.

### [Ensemble-averaged transmittance via characteristic functions](14531-ensemble-averaged-transmittance-via-characteristic-functions.md)
This method computes the ensemble-averaged transmittance by modeling the line-averaged micro-scale density as a random variable and evaluating its characteristic function (CF). It solves the problem of calculating light transport in media with spatial correlations without instantiating discrete particles. Unlike the macroscopic optical depth method, it derives closed-form transmittance directly from the probability distribution of density fluctuations.

### [Ensemble Transport Kernel](14536-ensemble-transport-kernel.md)
The Ensemble Transport Kernel (ETK) replaces the classical transmittance term with an ensemble-averaged kernel that accounts for spatial correlations. It builds on the Non-exponential path integral by approximating the ensemble average of the path throughput as a product of independent segment kernels, thereby decorrelating path segments. Unlike its siblings, it explicitly defines four distinct transport functions based on the boundary configuration of segment endpoints.

### Data-driven transmittance via ensemble averaging
A physically based approach to obtain transmittance profiles by numerically approximating the ensemble-averaging process through stochastic realizations of heterogeneous media or discrete point distributions [S1].
1.  Generate $N$ stochastic realizations of the medium (discrete particles or heterogeneous fields) [S1].
2.  For each realization $i$, calculate the visibility $V_{\mu_i}(\mathbf{x}, \mathbf{x}_t)$ over distance $t$ [S1].
3.  Average the visibilities to estimate $f_f(t)$ [S1]:
    $$ f_f(t) = \langle V_\mu(\mathbf{x}, \mathbf{x}_t) \rangle \approx \frac{1}{N} \sum_{i=1}^N V_{\mu_i}(\mathbf{x}, \mathbf{x}_t) $$
4.  Obtain other functions ($pf, pp, fp$) via integration, normalization, or differentiation using the relationships between transport functions [S1].
The number of realizations $N$ determines the accuracy of the approximation [S1].

## Background
The classical radiative transfer equation (RTE) assumes that scattering particles are distributed independently at random, leading to exponential transmittance decay [S1]. This assumption corresponds to a white noise distribution of particle positions, which is physically impossible as particles cannot occupy the same point in space [S2, book]. The non-exponential path integral framework was introduced to model media with spatial correlations (e.g., fractal noise, clustered particles) by treating the medium as a stochastic process [S1]. The foundational paper "A radiative transfer framework for non-exponential media" formalized the ensemble average approach, distinguishing between discrete particle distributions and heterogeneous field realizations [S1]. The characteristic function method leverages the mathematical property that the ensemble average of the exponential attenuation is the characteristic function of the line-averaged density evaluated at an imaginary argument [S1].

## Sources
[S1] A radiative transfer framework for non-exponential media, https://doi.org/10.1145/3272127.3275103, fetched 2026-11-08
[S2] 11.1 Volume Scattering Processes, https://pbr-book.org/4ed/Volume_Scattering/Volume_Scattering_Processes, fetched 2026-10-29 (book)
