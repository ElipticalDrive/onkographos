# Dynamic Tomographic Reconstruction with Motion Identification (DynTomo)

Also called: DynART, DynTomo, Dynamic ART-P-DVC, Dynamic Tomographic Reconstruction with Displacement Field Identification, Dynamic Tomography, Dynamic reconstruction algorithm, PDVC

## Overview
DynTomo is a staggered iterative method for reconstructing deforming 3D volumes from 2D projection data acquired over time, simultaneously identifying the full 2D space-time displacement field [S1]. It solves the problem of motion artifacts in tomography by alternating between algebraic volume reconstruction (DynART) and projection-based digital volume correlation (PDVC) for motion identification [S1]. Unlike standard Algebraic Reconstruction Techniques (ART) which produce blurry or split reconstructions under large displacements, DynTomo corrects for motion to yield sharp edges and accurate gray levels [S1]. It builds on [Tomographic reconstruction as inverse rendering](45-tomographic-reconstruction-as-inverse-rendering.md) by incorporating a parametric displacement model into the optimization loop.

## How it works
**Inputs:**
*   $p(r, \theta, t)$: Acquired sinogram (projections) at detector coordinates $r$, angles $\theta$, and time steps $t \in \{1, \dots, N_t\}$.
*   $N_t$: Number of time steps (projections).
*   $N_\theta$: Number of angles.
*   $N_{scale}$: Number of multiscale levels.
*   $\epsilon$: Convergence threshold for inner loops.
*   Basis functions: $N_\tau$ time functions $\phi_i(t)$ and $N_s$ spatial shape functions $\Phi_j(x)$.

**Outputs:**
*   $f(x)$: Reconstructed reference volume.
*   $u(x, t)$: Space-time displacement field.

**Data Structures:**
*   **Displacement Field Parametrization:**
    The displacement field is represented as a linear combination of basis functions:
    $$ u(x, t) = \sum_{i=1}^{N_\tau} \sum_{j=1}^{N_s} \alpha_{ij} \phi_i(t) \Phi_j(x) $$
    where $\alpha_{ij}$ are the amplitude coefficients. The reference state is defined at $t=0$ such that $u(x, 0) = 0$, implying $\phi_i(0) = 0$ [S1].
*   **Spatial Basis:**
    $\Phi_j(x)$ are composed of $C^4$ mesh elements (4-node square elements with bilinear interpolation) [S1]. In the source tests, $N_s = 18$ degrees of freedom [S1].
*   **Time Basis:**
    $\phi_i(t)$ are time functions. In the source tests, $N_\tau = 2$ [S1].
*   **Gaussian Kernel:**
    Defined as $G_\lambda(r) = \frac{1}{2\pi\lambda^2} \exp\left(-\frac{|r|^2}{2\lambda^2}\right)$ [S1].

**Algorithm Steps:**

1.  **Initialization:**
    *   Set displacement field $u(x, t) \leftarrow 0$ [S1].
    *   Iterate scales from $i = N_{scale}$ down to $0$.
    *   Set scale parameter $\lambda = 2^i$ [S1].
    *   Convolve recorded projections $p(r, \theta)$ with Gaussian kernel $G_\lambda$ to create filtered projections $\check{p}_\lambda(r, \theta)$:
        $$ \check{p}_\lambda(r, \theta) = \sum_{r'} G_\lambda(r') p(r - r', \theta) $$
        [S1].
    *   Initialize residual tracking: $\rho \leftarrow \check{p}_\lambda$, $\text{Residual} \leftarrow 1 + \epsilon$ (to force first loop) [S1].

2.  **Multiscale Iteration Loop:**
    While $\text{Residual} > \epsilon$:
    
    2.1. **Volume Reconstruction (DynART):**
    Reconstruct volume $f(x)$ using DynART with filtered projections $\check{p}_\lambda$ and current displacement $u$.
    
    *DynART Procedure:*
    *   Initialize $f^{(0)}(x) \leftarrow 0$, iteration counter $n \leftarrow 1$ [S1].
    *   Choose a permutation $\pi$ over $N_t$ acquisition times [S1].
    *   For $i \leftarrow 1$ to $N_{\text{DynART}}$ (number of inner iterations):
        *   For $t \leftarrow 1$ to $N_t$:
            *   $\tau \leftarrow \pi(t)$ [S1].
            *   **Warp:** Compute warped volume estimate $\tilde{f}^{(n)}(x, \tau) = f^{(n)}(x + u(x, \tau))$ [S1].
            *   **Residual:** Compute projected residual $\rho(r, \tau) = p(r, \tau) - R_\theta(\tau)[\tilde{f}^{(n)}(x, \tau)]$, where $R_\theta$ is the forward projection operator [S1].
            *   **Backproject:** Compute update in warped space $\Delta \tilde{f}^{(n+1)}(x) = B_\theta(\tau)[\rho(r, \tau)]$, where $B_\theta$ is the backprojection operator [S1].
            *   **Unwarp:** Compute unwarped update $\Delta f^{(n+1)}(x) = \Delta \tilde{f}^{(n+1)}(x - u(x, \tau))$ [S1].
            *   **Update:** Update volume $f^{(n+1)}(x) = f^{(n)}(x) + \frac{\Delta f^{(n+1)}(x)}{\Delta \tilde{f}}$. Apply constraints (e.g., positivity) [S1].
            *   Increment $n \leftarrow n + 1$ [S1].
    *   Return $f(x) = f^{(n)}(x)$ [S1].

    2.2. **Motion Identification (PDVC):**
    Update displacement $u$ using PDVC with volume $f$, filtered projections $\check{p}_\lambda$, and previous displacement $u$.
    
    *PDVC Procedure:*
    *   Initialize $u(x, t) \leftarrow u_0(x, t)$ (current estimate) [S1].
    *   Initialize $\rho_{\text{old}}(r, t) \leftarrow p(r, t) - R_\theta(t)[f^\lambda(x, t)]$ [S1]. Note: $f^\lambda$ is the volume reconstructed at the current scale.
    *   Initialize $\text{Progress} \leftarrow 1 + \epsilon$ [S1].
    *   While $\text{Progress} > \epsilon$:
        *   **Advection:** Compute warped volume $\tilde{f}(x, t) = f^\lambda(x + u(x, t))$ [S1].
        *   **Residual:** Compute current residual $\rho(r, t) = p(r, t) - R_\theta(t)[\tilde{f}(x, t)]$ [S1].
        *   **Convergence Check:** $\text{Progress} \leftarrow \|\rho(r, t) - \rho_{\text{old}}(r, t)\|$ [S1].
        *   **Sensitivity Computation:**
            For $i \leftarrow 1$ to $N_\tau$ and $j \leftarrow 1$ to $N_s$:
            *   Compute spatial gradient of warped volume $\nabla \tilde{f}(x, t)$ [S1].
            *   Compute $\chi_j(x, t) = R_\theta(t)[\Phi_j(x) \nabla \tilde{f}(x, t)]$ [S1].
            *   Compute sensitivity $S_{ij}(r, t) = \phi_i(t) \chi_j(x, t)$ [S1].
        *   **Linear System Assembly:**
            *   Hessian $H_{ijkl} = \sum_{r,t} S_{ij}(r, t) S_{kl}(r, t)$ [S1].
            *   Second member $b_{ij} = \sum_{r,t} \rho(r, t) S_{ij}(r, t)$ [S1].
        *   **Solve:** Solve linear system $[H] \delta\alpha = b$ for displacement amplitude correction $\delta\alpha$ [S1].
        *   **Update:** Update displacement $u(x, t) \leftarrow u(x, t) + \delta\alpha \phi(t) \Phi(x)$ [S1].
        *   **Store:** $\rho_{\text{old}}(r, t) \leftarrow \rho(r, t)$ [S1].
    *   Return updated $u(x, t)$ [S1].

    2.3. **Scale Residual Update:**
    *   $\rho_{\text{old}} \leftarrow \rho$ [S1].
    *   Compute $\rho(r, t)$ (using current $f$ and $u$) [S1].
    *   $\text{Residual} \leftarrow \|\rho - \rho_{\text{old}}\|$ [S1].

**Parameters:**
*   **Convergence Threshold $\epsilon$:** Stops inner loops when residual or progress drops below this value [S1].
*   **Basis Functions:** $N_\tau$ time functions and $N_s$ spatial shape functions. Reference state $u(x, 0)=0$ implies $\phi_i(0)=0$ [S1].
*   **Iterations:** The source tests used 60 complete iterations per reconstruction cycle [S1].
*   **Gaussian Kernel Width:** Starts large ($\lambda = 2^{N_{scale}}$), progressively reduced by a factor of two at each scale level [S1].

**Pitfalls:**
*   **Motion Regularization:** Rapid variations in the displacement field can cause ill-conditioning. Weak regularization penalizes spatial/temporal variations; strong regularization uses smooth basis functions [S1].
*   **Optical Flow Assumption:** The base implementation assumes constant optical flow and does not model gray level variations [S1].
*   **Sensitivity Bottleneck:** Sensitivity computation requires deformed volumes over the entire time range, making it the computational bottleneck [S1].

## Operating characteristics
*   **Complexity:** The bottleneck is the computation of sensitivity fields, which requires projecting deformed volumes over the entire time range for every PDVC iteration [S1].
*   **Performance:** In tests with 512×512 pixel images and 300 projections, 60 iterations took approximately 1 hour [S1]. The checkerboard test (38 degrees of freedom) took approximately 2 hours [S1].
*   **Accuracy:**
    *   Shepp-Logan test (max displacement 37 pixels, 27% strain): Displacement error standard deviation of 3.10 pixels after 60 iterations [S1].
    *   Checkerboard test (max strain ~25%): Displacement error standard deviation < 1.2 pixel after 60 iterations [S1].
*   **Strengths:** Produces sharp edges and correct gray levels even with large displacements, unlike standard ART which produces blurry reconstructions with split parts [S1].
*   **Limitations:** Assumes constant optical flow; not robust to gray level variations without extension [S1]. Computationally expensive due to sensitivity calculation [S1].
*   **Usage:** Applicable to in situ mechanical tests, synchrotron beam-lines, and data assimilation [S1].

## Background
DynTomo was introduced in 2018 by [S1]. It is derived from Algebraic Reconstruction Techniques (ART) coupled with Projection-based Digital Volume Correlation (P-DVC) [S1]. The method was designed to handle large displacements and strains in dynamic tomography, such as those found in in situ mechanical testing [S1]. It was tested on numerical phantoms (Shepp-Logan and checkerboard) with parallel projection geometry [S1].

## Sources
[S1] Dynamic Tomographic Reconstruction of Deforming Volumes, https://doi.org/10.3390/ma11081395, fetched 2026-10-26
