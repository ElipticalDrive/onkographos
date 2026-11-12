# Local Piecewise Integration for Light Propagation

Also called: Convection-Diffusion Volume Lighting, Grid-based Volume Illumination, Grid-based illumination method, Light Volume Method, Light propagation method, Local Piecewise Integration, Upwind scheme illumination

## Overview
Local Piecewise Integration for Light Propagation is a precomputation technique that approximates global light transport in participating media by decomposing the radiative transfer equation into convection (absorption) and diffusion (scattering) steps solved on a regular grid [S1, S2]. It solves the problem of expensive volumetric illumination by computing high-resolution local shadows via piecewise opacity segments and propagating light energy through the volume using a convection-diffusion solver, allowing real-time rendering via single texture lookups [S2]. It is distinguished from full path-tracing methods by its reliance on precomputed segment data and from simple direct illumination by its inclusion of first-order inscattering approximations [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Volume dataset (resampled to grid), Transfer Function (TF), light source definitions (position, intensity, type), grid resolution $\Delta x$.
*   **Outputs:** Final illumination volume $L(x)$ (RGB 3D texture), piecewise opacity segments $\alpha_{lp}(s_n)$.

**Data Structures:**
1.  **Opacity Volume:** 3D grid derived from the transfer function and resampled data volume. Resolution matches the light volume. Used to compute absorption coefficient $\sigma_a(x)$ [S1].
2.  **Light Volume:** 3D grid of energy density values $\rho_{i,j,k}$. Resolution $N_x, N_y, N_z$. Typically half the resolution of the original volume data [S1].
3.  **Light Color Field:** 3D grid of RGB vectors $L(x)$, composed from individual light energy densities $\rho_k(x)$ and light colors $L_k$ [S1].
4.  **Piecewise Segments:** Stored transparency values $T_{lp}(s_n)$ or opacity $\alpha_{lp}(s_n)$ for segments of length $\Delta s$ [S2].

**Parameters:**
*   $\Delta x$: Grid resolution. Can be lower than volume dataset for performance [S1].
*   $c$: Speed of light (used in CFL condition). Implementation uses $c\Delta t = \Delta x/2$ [S1].
*   $\sigma_a(x)$: Absorption coefficient, $0 \le \sigma_a \le 1$ [S1].
*   $\sigma_s$: Scattering coefficient. Scalar for Eq. 13, vector $\hat{\sigma}_s$ for Eq. 23 (RGB channels) [S1].
*   $N$: Number of samples along a local piecewise segment [S2].
*   $\Delta s$: Step length for global light integration, equal to segment length. Typically 8-16 voxels [S2].
*   $I_{bias}$: Minimum lighting adjustment to prevent completely dark regions [S2].
*   $R^*\Omega$: Scattering sphere radius; larger values result in less distinct shadows [S2].
*   Iterations per layer: Typically 4-5 iterations for the upwind scheme [S1].

**Algorithm Steps:**

1.  **Volume Discretization:**
    Discretize the volume domain using a regular grid with resolution $\Delta x$. Energy density $\rho(x)$ is discretized as $\rho_{i,j,k} = \rho(x_{i,j,k})$ where $x_{i,j,k} = (i\Delta x, j\Delta x, k\Delta x)$ [S1].

2.  **Local Piecewise Integration (Shadowing):**
    For each voxel location in the volume, cast rays in the direction of the light source. Compute the transparency $T_{lp}(s_n)$ for segments starting at $s_n = n\Delta s$, where $s_0$ is the current voxel location [S2].
    The intensity of direct light reaching position $s$ is calculated as:
    $$I(s) = I_0 \times \prod_{n=0}^k T_{lp}(s_n)$$
    where $I_0$ is the light source intensity and $T_{lp}(s_n)$ is the transparency of the $n$-th segment [S2].
    The transparency of a single segment is defined as:
    $$T_{lp}(s_n) = e^{-\int_{s_n}^{s_n+1} \tau(t) dt}$$
    where $\tau(t)$ is the absorption coefficient [S2].
    The segment opacity $\alpha_{lp}(s_n)$ is discretized by sampling $N$ points along the segment:
    $$\alpha_{lp}(s_n) = 1 - \prod_{i=0}^N (1 - \alpha(n_i))$$
    where $\alpha(n_i)$ is the opacity of the $i$-th sample point in the current Transfer Function (TF) domain [S2].
    *Note: These segments must be recomputed whenever the TF changes or the volume moves relative to the light source [S2].*

3.  **Light Volume Initialization:**
    Initialize the light volume grid $\rho_{i,j,k}$. Set boundary values according to light source type (Dirichlet/Neumann) and set interior values to zero, except for internal point lights [S1].
    *   **External Point Lights:** If $x_P \notin \Omega$, use hybrid Dirichlet-Neumann boundary conditions. $\rho(x)|_{\partial \Omega} = G(x, x_P)\rho_P$ if $n(x) \cdot (x - x_P) < 0$, else $\frac{\partial \rho(x)}{\partial n} = 0$. $G(x, x_P)$ is a Gaussian falloff function [S1].
    *   **Internal Point Lights:** If $x_P \in \Omega$, set voxels within distance $2\Delta x$ from $x_P$ to the light intensity $\rho_P$. These voxels are treated as an internal boundary and not updated during simulation: $\rho(x_P) = \rho_P$ [S1].

4.  **Convection Step (Absorption):**
    Solve the convection equation with absorption for each light source individually:
    $$\frac{\partial \rho(x)}{\partial t} = -c u(x) \cdot \nabla \rho(x) - \sigma_a(x)\rho(x)$$
    where $c$ is speed of light, $u(x)$ is unit light direction, and $\sigma_a(x)$ is absorption coefficient [S1].
    Use a first-order upwind scheme. Update layer by layer along the light propagation direction. Repeat until steady state ($-c u(x) \cdot \nabla \rho(x) - \sigma_a(x)\rho(x) = 0$) is reached, typically 4-5 iterations per layer [S1].
    The upwind gradient component $x$ is computed as:
    $$\rho_{i,j,k}^x = \begin{cases} \frac{\rho_{i+1,j,k} - \rho_{i,j,k}}{\Delta x} & \text{if } u_{i,j,k} < 0 \\ \frac{\rho_{i,j,k} - \rho_{i-1,j,k}}{\Delta x} & \text{if } u_{i,j,k} > 0 \end{cases}$$
    (Similar formulas apply for $y$ and $z$ components based on $v_{i,j,k}$ and $w_{i,j,k}$) [S1].
    The update rule is:
    $$\rho_{i,j,k}(t+\Delta t) = (1 - \sigma_a(x))\rho_{i,j,k}(t) - \frac{\Delta t}{\Delta x} \Delta \rho_{i,j,k}(t)$$
    with CFL condition $C = c\Delta t/\Delta x \le 1$. Implementation uses $c\Delta t = \Delta x/2$, so $C=1/2$ [S1].

5.  **Light Field Composition:**
    Composite solutions from all light sources into a single light color field $L(x)$:
    $$L(x) = \sum_{k=1}^N (\rho_k(x)L_k)$$
    where $\rho_k(x)$ is the solution for the $k$-th light source and $L_k$ is its color [S1].

6.  **Diffusion Step (Scattering):**
    Solve the diffusion equation for scattering using Gauss-Seidel iteration on the combined light field $L(x)$:
    $$\frac{\partial L(x)}{\partial t} = \hat{\sigma}_s \nabla^2 L(x)$$
    where $\hat{\sigma}_s$ consists of three scattering coefficients for different color channels [S1].
    Use the solution at $t=1$ as the final illumination volume. Solving this once for the combined field is significantly cheaper than solving for each light source separately [S1].

7.  **Rendering:**
    Render using volume ray casting. For each sample point, lookup illumination $L(x)$ from the final volume and combine with Phong shading:
    $$L_{final}(x) = L(x)(I_a(x) + I_d(x) + I_s(x))$$
    where $I_a, I_d, I_s$ are ambient, diffuse, and specular lighting intensities [S1].
    Only one additional texture lookup is performed for each sample to apply the pre-computed light intensities [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def local_piecewise_integration(volume_data, transfer_function, lights, grid_resolution):
    # 1. Discretize and Initialize
    opacity_volume = compute_opacity_volume(volume_data, transfer_function, grid_resolution)
    light_volume = initialize_light_volume(grid_resolution)
    
    # 2. Compute Local Piecewise Segments (Shadows)
    # Recompute if TF or volume-light relation changes
    piecewise_segments = []
    for voxel in grid_voxels:
        segment_opacities = []
        for n in range(num_segments):
            samples = sample_ray(voxel, light_direction, segment_length, transfer_function)
            alpha_segment = 1.0
            for sample in samples:
                alpha_segment *= (1.0 - sample.opacity)
            segment_opacities.append(1.0 - alpha_segment)
        piecewise_segments.append(segment_opacities)

    # 3. Solve Convection for each light
    for light in lights:
        current_energy = initialize_energy_volume(light, grid_resolution)
        
        # Initialize boundaries based on light type (Dirichlet/Neumann/Internal)
        set_boundary_conditions(current_energy, light, opacity_volume)
        
        # Upwind Scheme Iteration
        for layer in layers_along_light_direction:
            for iteration in range(4, 6): # 4-5 iterations
                update_convection_step(current_energy, opacity_volume, light.direction, grid_resolution)
        
        accumulate_energy(light_volume, current_energy, light.color)

    # 4. Solve Diffusion for Scattering
    # Solve once on combined light field
    final_illumination = solve_diffusion_gauss_seidel(light_volume, scattering_coefficient)
    
    return final_illumination, piecewise_segments

def update_convection_step(energy, opacity, direction, dx):
    # First-order upwind scheme
    # CFL condition: c*dt = dx/2
    for voxel in energy:
        grad_x = upwind_gradient(energy, direction.x, dx)
        grad_y = upwind_gradient(energy, direction.y, dx)
        grad_z = upwind_gradient(energy, direction.z, dx)
        divergence = direction.x * grad_x + direction.y * grad_y + direction.z * grad_z
        sigma_a = opacity[voxel]
        energy[voxel] = (1.0 - sigma_a) * energy[voxel] - 0.5 * divergence
```

## Subtopics

### [Piecewise Integration](10689-piecewise-integration.md)
Piecewise Integration is a precomputation technique for real-time volume rendering that approximates global light transport by dividing light paths into discrete segments. It solves the problem of expensive volumetric illumination by storing transmittance data in a multiresolution 3D texture, allowing incoming radiance to be computed via a single texture lookup per sample. It is distinguished from full path-tracing methods by its reliance on precomputed segment data and from simple direct illumination by its inclusion of first-order inscattering approximations.

### Source Term Computation for External Lighting
A variant for handling external directional light $L_l$. It computes the multiple scattering source term $q_{ri}(x)$ for external directional light as $q_{ri}(x) = L_l \cdot \sigma_s(x) \cdot T(x)$, where $\sigma_s$ is the scattering coefficient and $T(x)$ is transmittance. This term is computed via raymarching and stored at the full resolution of the dataset extinction field [S3].

### Radial Grids for Local Light Sources
A propagation scheme using radial grids (topologically equivalent to rectilinear grids but cyclic in two angular dimensions) for local point lights, allowing approximate treatment of space as locally Euclidean. It extends the Principal-Ordinates Propagation (POP) method to local light sources by using spherical grids aligned with the initial principal directions of the point source. VPLs with large importance have high initial anisotropy, which is accounted for in the grid spacing [S4].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** The whole volume data is stored in GPU memory, constraining maximum volume size by total available GPU memory [S1].
*   **Time:** Integration of light intensities without the local piecewise approximation is approximately 10 times slower [S2]. Global light integration updates in 284-552 ms (for 8 voxel segments) depending on Shadow Volume Representation (SVR) size; first order scattering incremental refinements take 62-233 ms [S2]. Light simulation time for a $128 \times 128 \times 128$ light volume: Absorption takes 34ms, Scattering takes 18ms [S1].

**Measured Performance:**
*   Hardware: Laptop with Intel 1.7GHz CPU and NVIDIA Geforce 460M GPU [S1].
*   Accuracy: Root Mean Square (RMS) error of 0.19 for $\Delta E$ (CIE 1976 L*u*v* color space), with 6% of pixels having $\Delta E$ above 6 [S2].

**Strengths and Limitations:**
*   **Strengths:** Interactive frame rates (15-22 fps) once light calculations are updated; nearly physically correct scattering effects compared to blur-based methods like Kniss et al. [KPH*03]; light source can be positioned anywhere, even within the volume; handles large scale volumes via multi-resolution data management [S2].
*   **Limitations:** Numerical dissipation from first-order upwind scheme causes soft shadows instead of hard shadows (though considered visually plausible) [S1]; not as fast as precomputation-based techniques that do not require per-frame updates; block artifacts and banding effects appear if SVR resolution is too low [S2].
*   **When to use:** Real-time volume visualization where lighting changes (TF, light position) occur frequently, and precomputation is not feasible.
*   **When not to use:** When hard shadows are required, or when the volume is extremely large and GPU memory is insufficient.

## Background
The method was introduced by Cheng and Cheng in 2012 [S1] and further refined by Larsson et al. in 2008 [S2]. It is based on solving the convection-diffusion equation to model light propagation, absorption, and scattering in optically thick media. The upwind scheme originates from Courant et al. (1952) [S1]. The technique is implemented using NVIDIA CUDA for simulation and OpenGL for rendering, with the final light volume stored in an OpenGL 3D texture [S1].

## Sources
[S1] Fast global illumination for interactive volume visualization, https://doi.org/10.1145/2448196.2448205, fetched 2026-10-30
[S2] Interactive Global Light Propagation in Direct Volume Rendering using Local Piecewise Integration, https://doi.org/10.2312/vg/vg-pbg08/105-112, fetched 2026-10-26
[S3] Flux-Limited Diffusion for Multiple Scattering in Participating Media, https://archive.org/details/arxiv-1403.8105, fetched 2026-10-26
[S4] Principal-Ordinates Propagation for Real-Time Rendering ..., https://cg.ivd.kit.edu/publications/2015/principal-ordinates/ElekCAG2014.pdf, fetched 2026-10-29
