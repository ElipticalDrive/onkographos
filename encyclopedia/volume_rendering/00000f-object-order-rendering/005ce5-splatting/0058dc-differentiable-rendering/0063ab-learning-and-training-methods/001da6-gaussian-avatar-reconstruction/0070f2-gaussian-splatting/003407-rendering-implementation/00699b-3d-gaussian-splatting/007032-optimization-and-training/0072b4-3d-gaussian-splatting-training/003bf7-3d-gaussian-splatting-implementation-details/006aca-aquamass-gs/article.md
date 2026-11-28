# AquaMass-GS

Also called: AM-GS

## Overview
AquaMass-GS is a per-scene underwater rendering method that decouples object appearance from water transport by coupling a standard object-Gaussian scene model to a separate field of normalized free-space water Gaussians [S1, unofficial]. It solves the entanglement of attenuation and backscatter with object appearance, and the inability to query water transport independently of object sampling, by using cell-volume weighting for optical mass semantics and analytic finite-path integration with coverage normalization [S1, unofficial]. Unlike global or direction-conditioned predictors, it keeps water transport spatially queryable in free space, evaluating only the traversed finite segment from camera to visible surface [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Posed underwater images; Structure-from-Motion (SfM) point cloud bounds; camera intrinsics/extrinsics.
*   **Outputs:** Rendered underwater image; optimized object Gaussian set $G_o$; optimized water Gaussian parameters.

**Data Structures:**
1.  **Object Gaussians ($G_o$):** Standard 3DGS parameters: position $\mu$, covariance $\Sigma$ (or rotation/scale), spherical harmonic coefficients $c_j$, opacity $\alpha$ [S1, unofficial].
2.  **Water Lattice Centers:** Fixed on a face-centered-cubic (FCC) lattice. For $N$ cells per axis, there are $4N^3$ centers. Centers are located within the axis-aligned bounds of SfM points plus a fixed world-space margin [S1, unofficial].
3.  **Water Gaussian Parameters ($G_w$):**
    *   Position $\mu_i$: Fixed to lattice centers.
    *   Covariance $\Sigma_i$: Fixed, axis-aligned. Standard deviation $s_\Delta$ along each dimension. Typical value $s = 1.3$; initial layout $s = 0.7$ [S1, unofficial].
    *   Densities: Two non-negative RGB density vectors per center, $\rho_{D,i}$ and $\rho_{B,i}$, optimized during training. These represent direct extinction and backscatter attenuation, respectively [S1, unofficial].

**Steps:**
1.  **Optimization:** Jointly optimize $G_o$ and $G_w$ from posed underwater images. $G_o$ follows standard 3DGS densification and parameter updates. $G_w$ centers and covariances are fixed; only $\rho_{D,i}$ and $\rho_{B,i}$ are updated [S1, unofficial].
2.  **Object Query:** Render the object branch $G_o$ to obtain a detached alpha-weighted surface endpoint (proxy depth) $z^*$ [S1, unofficial].
3.  **Ray Distance Conversion:** Convert proxy depth $z^*$ to ray distance $t^*$ along the viewing ray [S1, unofficial].
4.  **Analytic Integration:** Perform analytic finite-path integration over the ray segment $[0, t^*]$ to compute:
    *   Direct extinction optical depth $\tau_D$
    *   Backscatter optical depth $\tau_B$
    *   Unit-density coverage $q$
5.  **Coverage Normalization:** Compute per-pixel equivalent coefficients for direct mass $\beta_D$ and backscatter mass $\beta_B$:
    $$ \beta_D = \frac{\tau_D}{q}, \quad \beta_B = \frac{\tau_B}{q} $$
6.  **Composition:** Combine the object term (using $\beta_D$) and the medium term (using $\beta_B$ and a direction-conditioned backscatter head $B_\infty(d_p)$) to form the final rendered image [S1, unofficial].

**Formulas:**
The normalized basis function for water Gaussian $i$ is:
$$ \phi_i(x) = \frac{\exp\left(-\frac{1}{2}(x - \mu_i)^\top \Sigma_i^{-1} (x - \mu_i)\right)}{(2\pi)^{3/2} |\Sigma_i|^{1/2}} $$
where $\int_{\mathbb{R}^3} \phi_i(x) dx = 1$ [S1, unofficial].

Cell volume is $\Delta V = \Delta x \Delta y \Delta z$. Each lattice point is assigned volume $V_i = \Delta V / 4$. The optical mass for direct extinction and backscatter is:
$$ m_{D,i} = V_i \cdot \text{softplus}(\rho_{D,i}) $$
$$ m_{B,i} = V_i \cdot \text{softplus}(\rho_{B,i}) $$
where $\text{softplus}(x) = \ln(1 + e^x)$ [S1, unofficial].

The continuous optical fields are:
$$ \sigma_D(x) = \sum_i m_{D,i} \phi_i(x) $$
$$ \sigma_B(x) = \sum_i m_{B,i} \phi_i(x) $$
[Source note: The source text cuts off after $\sigma_B(x) = \dots$ but implies the same summation structure for backscatter].

**Parameters:**
*   $s_\Delta$: Water Gaussian standard deviation. Fixed. Typical value $1.3$ [S1, unofficial].
*   $N$: Cells per axis. Determines lattice resolution ($4N^3$ centers) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Geometric Resizing:** Position and covariance of water Gaussians are fixed to prevent transport mass from changing through geometric resizing [S1, unofficial].
*   **Boundary Conditions:** The lattice is bounded by SfM points plus a fixed world-space margin. Queries outside this range may require handling (not specified in sources) [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** On four SeaThru-NeRF development scenes, AM-GS provides competitive end-to-end novel-view quality, with the highest mean PSNR and the lowest LPIPS among six compared methods [S1, unofficial].
*   **Strengths:** Decouples water transport from object appearance; allows spatial querying of water independent of camera motion; handles finite path segments accurately [S1, unofficial].
*   **Limitations:** Requires separate optimization for water field; fixed lattice may not adapt to complex underwater structures dynamically [S1, unofficial].
*   **When to use:** Per-scene underwater rendering where accurate separation of object and water effects is critical [S1, unofficial].
*   **When not to use:** Real-time applications requiring dynamic adaptation of water density; scenes without clear object-water boundaries [S1, unofficial].

## Background
AquaMass-GS was proposed to address limitations in existing underwater rendering methods, which often entangle attenuation and backscatter with object appearance or use global/directional predictors that limit spatial queries [S1, unofficial]. It builds on the WaterSplatting implementation for the object branch and direction encoding [S1, unofficial]. The method introduces normalized free-space water Gaussians with cell-volume weighting to provide optical mass semantics [S1, unofficial].

## Sources
[S1] Optical-Mass Water Gaussians for Per-Scene Underwater Rendering, https://doi.org/10.21203/rs.3.rs-10938586/v1, fetched 2026-10-26 (unofficial)
