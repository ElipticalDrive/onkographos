# Volumetric Primitive Radiance Field Integrator

## Overview
The Volumetric Primitive Radiance Field Integrator is a path-tracing method for rendering scattering and emissive media represented by mixtures of kernel-based volumetric primitives (e.g., Gaussians, Epanechnikov) rather than voxel grids or point clouds [S1, S2, unofficial]. It modifies standard volumetric path tracing by replacing medium interaction routines with primitive-specific transmittance evaluation, free-flight distance sampling, and albedo evaluation [S1, S2, unofficial]. It distinguishes itself from its sibling, the Volumetric Primitive Tomography Integrator, by supporting scattering and emissive media via multiple importance sampling with PDF balancing, whereas the tomography integrator integrates density with no scattering [S1, S2, unofficial].

## How it works
The integrator inherits from `RBIntegrator` and operates on a scene containing `EllipsoidMesh` primitives [S1, unofficial]. Each primitive is defined by a center, rotation, scale, and opacity, optionally accompanied by spherical harmonic (SH) coefficients for emission [S1, unofficial]. The method supports both primal (forward) and adjoint (backward) modes for differentiable rendering [S1, unofficial].

### Inputs and Outputs
*   **Input**: A scene graph containing `EllipsoidMesh` primitives. Each primitive stores geometric parameters (center $\mathbf{c}$, rotation matrix $\mathbf{R}$, scale vector $\mathbf{s}$), opacity $\alpha$, and optionally SH coefficients $\mathbf{c}_{SH}$ for emission [S1, unofficial].
*   **Output**: Radiance $L$ (primal) or differential radiance $\delta L$ (adjoint).

### Data Structures
*   **Primitive Attributes**:
    *   `center`: 3D vector $\mathbf{c}$.
    *   `rotation`: $3 \times 3$ rotation matrix $\mathbf{R}$.
    *   `scale`: 3D vector $\mathbf{s}$.
    *   `opacity`: Scalar $\alpha \in [0, 1]$.
    *   `sh_coeffs`: Array of spherical harmonic coefficients.
*   **State Variables (per ray)**:
    *   `L`: Radiance accumulator.
    *   $\delta L$: Differential radiance accumulator (adjoint).
    *   $\beta$: Path throughput.
    *   `depth`: Current path depth.
    *   `active`: Boolean mask for active rays.

### Parameters
*   `max_depth` (int): Maximum path depth. -1 indicates no limit, mapped to `0xFFFFFFFF` internally [S1, unofficial].
*   `rr_depth` (int): Minimum path depth before enabling Russian roulette. -1 indicates no limit, mapped to `0xFFFFFFFF` [S1, unofficial].
*   `kernel_type` (str): Name of the kernel to use for rendering the volumetric primitives, one of `['gaussian', 'epanechnikov']` [S1, unofficial].
*   `srgb_primitives` (bool): If true, converts sRGB light transport to linear color space at the end [S1, unofficial].
*   `use_rr` (bool): Whether to use Russian Roulette. Determined by `rr_depth >= 0` and (`rr_depth < max_depth` or `max_depth == -1`) [S1, unofficial].

### Algorithm Steps

1.  **Initialization**:
    *   Set `max_depth` to `0xFFFFFFFF` if input is -1 [S1, unofficial].
    *   Set `rr_depth` to `0xFFFFFFFF` if input is -1 [S1, unofficial].
    *   Determine `use_rr` based on `rr_depth` and `max_depth` [S1, unofficial].
    *   Initialize kernel with `kernel_full_range=True` and `kernel_normalized=True` [S1, unofficial].
    *   For each pixel/ray:
        *   Initialize $L = 0$ (or state input), $\delta L = 0$ (or state input), $\beta = 1.0$, `depth` = 0 [S1, unofficial].
        *   Generate ray $\mathbf{r}$.
        *   Set `active` mask to true for all rays [G].

2.  **Ray Intersection Loop**:
    *   While `active` is not empty:
        *   Perform ray intersection with scene using `scene.ray_intersect` with flags `All | BackfaceCulling` and coherent intersection for `depth == 0` [S1, unofficial].
        *   Filter active rays to only those intersecting ellipsoids:
            $$ \text{active} \leftarrow \text{active} \ \& \ (\text{si.is\_valid}() \ \& \ (\text{si.shape.shape\_type}() == \text{ShapeType.Ellipsoids})) $$
        *   If no active rays remain, break loop [G].

3.  **Primitive Evaluation**:
    *   For active rays intersecting ellipsoids:
        *   **Evaluate Transmission**:
            *   Gather ellipsoid parameters: center $\mathbf{c}$, rotation $\mathbf{R}$, scale $\mathbf{s}$, opacity $\alpha$ [S1, unofficial].
            *   Transform ray origin and direction to local space:
                $$ \mathbf{o}' = \mathbf{R}^T (\mathbf{r}.o - \mathbf{c}) / \mathbf{s} $$
                $$ \mathbf{d}' = \mathbf{R}^T \mathbf{r}.d / \mathbf{s} $$
            *   Compute peak location along ray:
                $$ t_{peak} = -\frac{\mathbf{o}' \cdot \mathbf{d}'}{\mathbf{d}' \cdot \mathbf{d}'} $$
            *   Evaluate density at peak: $\rho = \text{density}(t_{peak})$ based on `kernel_type` (Gaussian or Epanechnikov) [S1, S2, unofficial].
            *   Compute transmission:
                $$ \text{transmission} = 1.0 - \min(\alpha \cdot \rho, 0.9999) $$
                *Note: Clamping to 0.9999 avoids numerical issues with $1.0 - 1.0$ [S1, unofficial].*
        *   **Evaluate Emission**:
            *   Gather SH coefficients $\mathbf{c}_{SH}$ [S1, unofficial].
            *   Compute SH degree: $n = \sqrt{\text{sh\_coeffs.shape}[0] / 3 - 1}$ [S1, unofficial].
            *   Evaluate SH basis for viewing direction $\mathbf{d}$: $\mathbf{sh\_dir\_coef} = \text{sh\_eval}(\mathbf{d}, n)$ [G].
            *   Compute emission:
                $$ \text{emission} = \max(\sum (\mathbf{c}_{SH} \odot \mathbf{sh\_dir\_coef}) + 0.5, 0.0) $$
                *Note: Adding 0.5 before clamping handles potential negative values from SH evaluation [S1, unofficial].*
        *   **Clamp Infinite Values**:
            $$ L_e[\sim \text{dr.is\_finite}(L_e)] = 0.0 $$

4.  **Accumulation and Throughput Update**:
    *   Compute local emission contribution:
        $$ L_e = \beta \cdot (1.0 - \text{transmission}) \cdot \text{emission} $$
    *   Update radiance accumulator:
        $$ L[\text{active}] = L[\text{active}] + L_e \quad (\text{if primal}) $$
        $$ L[\text{active}] = L[\text{active}] - L_e \quad (\text{if adjoint}) $$
    *   Update path throughput:
        $$ \beta[\text{active}] *= \text{transmission} $$

5.  **Adjoint Indirect Radiance (Adjoint Mode Only)**:
    *   Compute differentiable reflected indirect radiance:
        $$ L_{r\_ind} = L \cdot \text{transmission} / \text{dr.detach}(\text{transmission}) $$
    *   Compute total local contribution:
        $$ L_o = L_e + L_{r\_ind} $$
    *   Select finite values:
        $$ L_o = \text{dr.select}(\text{active} \ \& \ \text{dr.is\_finite}(L_o), L_o, 0.0) $$
    *   Propagate gradients:
        *   If Backward: `dr.backward_from(δL * Lo)` [S1, unofficial]
        *   If Forward: `δL += dr.forward_to(Lo)` [S1, unofficial]

6.  **Ray Propagation and Termination**:
    *   Spawn new ray origin to avoid self-intersection:
        $$ \mathbf{r}.o[\text{active}] = \text{si.p} + \mathbf{r}.d \cdot 10^{-4} $$
    *   Update active mask and depth:
        $$ \text{active} \leftarrow \text{active} \ \& \ \text{si.is\_valid}() $$
        $$ \text{depth}[\text{active}] += 1 $$
    *   Kill paths with insignificant contribution:
        $$ \beta_{max} = \text{dr.max}(\beta) $$
        $$ \text{active} \leftarrow \text{active} \ \& \ (\beta_{max} > 0.01) $$
    *   **Russian Roulette** (if `primal` and `use_rr`):
        *   Calculate probability: $p_{rr} = \max(\beta_{max}, 0.1)$ [S1, unofficial]
        *   Identify candidates: $\text{rr\_active} = (\text{depth} \ge \text{rr\_depth}) \ \& \ (\beta_{max} < 0.1)$ [S1, unofficial]
        *   Scale throughput: $\beta[\text{rr\_active}] *= 1.0 / p_{rr}$ [S1, unofficial]
        *   Determine continuation: $\text{rr\_continue} = \text{sample\_rr} < p_{rr}$ [S1, unofficial]
        *   Update active: $\text{active} \leftarrow \text{active} \ \& \ \text{rr\_continue}$ [G]
    *   Limit recursion depth:
        $$ \text{active} \leftarrow \text{active} \ \& \ (\text{depth} < \text{max\_depth}) $$

7.  **Final Output**:
    *   If `srgb_primitives` is true, convert $L$ to linear space: $L = \text{mi.math.srgb\_to\_linear}(L)$ [S1, unofficial].
    *   Return $L$ if primal, else $\delta L$ [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_volprim_prb(scene, rays_o, rays_d, mode='primal', max_depth=-1, rr_depth=-1, kernel_type='gaussian', srgb_primitives=True):
    # Initialize parameters
    if max_depth == -1: max_depth = 0xFFFFFFFF
    if rr_depth == -1: rr_depth = 0xFFFFFFFF
    use_rr = (rr_depth >= 0) and ((rr_depth < max_depth) or (max_depth == 0xFFFFFFFF))

    N = len(rays_o)
    L = [[0.0, 0.0, 0.0] for _ in range(N)]
    delta_L = [[0.0, 0.0, 0.0] for _ in range(N)]
    beta = [1.0] * N
    depth = [0] * N
    active = [True] * N

    kernel = Kernel(kernel_type, full_range=True, normalized=True)

    while any(active):
        # Ray intersection
        si = scene.ray_intersect(rays_o, rays_d, flags=ALL | BACKFACE_CULLING, coherent=(depth[0] == 0))
        
        # Filter for ellipsoids
        is_valid = [si.is_valid(i) for i in range(N)]
        is_ellipsoid = [si.shape.shape_type(i) == SHAPE_ELLIPSOID for i in range(N)]
        active = [a & v & e for a, v, e in zip(active, is_valid, is_ellipsoid)]
        if not any(active): break

        idxs = [i for i in range(N) if active[i]]
        centers = [si.shape.centers[si.primitive_index(i)] for i in idxs]
        rotations = [si.shape.rotations[si.primitive_index(i)] for i in idxs]
        scales = [si.shape.scales[si.primitive_index(i)] for i in idxs]
        opacities = [si.shape.opacities[si.primitive_index(i)] for i in idxs]
        sh_coeffs = [si.shape.sh_coeffs[si.primitive_index(i)] for i in idxs]

        # Transform to local space
        local_o = [scale_vector(dot_product(transpose(rot), subtract(o, c)), 1.0/s) for rot, o, c, s in zip(rotations, [rays_o[i] for i in idxs], centers, scales)]
        local_d = [scale_vector(dot_product(transpose(rot), d), 1.0/s) for rot, d, s in zip(rotations, [rays_d[i] for i in idxs], scales)]

        # Compute peak location
        t_peak = [-dot_product(lo, ld) / dot_product(ld, ld) for lo, ld in zip(local_o, local_d)]
        
        # Evaluate density at peak
        density = [kernel.evaluate_density(lo, ld, tp) for lo, ld, tp in zip(local_o, local_d, t_peak)]
        
        # Compute transmission
        transmittance = [1.0 - min(op * den, 0.9999) for op, den in zip(opacities, density)]

        # Evaluate emission
        emission = []
        for i in range(len(idxs)):
            sh_deg = sqrt(len(sh_coeffs[i]) / 3.0 - 1.0)
            sh_basis = sh_eval(rays_d[idxs[i]], sh_deg)
            val = sum(c * b for c, b in zip(sh_coeffs[i], sh_basis)) + 0.5
            emission.append(max(val, 0.0))
        emission = [0.0 if not isfinite(e) else e for e in emission]

        # Compute local contribution
        Le = [beta[idxs[i]] * (1.0 - transmittance[i]) * emission[i] for i in range(len(idxs))]

        # Update radiance accumulator
        if mode == 'primal':
            for i, idx in enumerate(idxs):
                L[idx] = add_vectors(L[idx], Le[i])
        else:
            for i, idx in enumerate(idxs):
                L[idx] = subtract_vectors(L[idx], Le[i])

        # Update throughput
        for i, idx in enumerate(idxs):
            beta[idx] *= transmittance[i]

        # Adjoint indirect radiance
        if mode == 'adjoint':
            Lr_ind = []
            for i, idx in enumerate(idxs):
                Lr_ind.append(scale_vector(add_vectors(L[idx], Le[i]), transmittance[i] / transmittance[i]))
            Lo = [add_vectors(Le[i], Lr_ind[i]) for i in range(len(idxs))]
            Lo = [0.0 if not isfinite(v) else v for v in Lo]
            for i, idx in enumerate(idxs):
                delta_L[idx] = add_vectors(delta_L[idx], Lo[i])

        # Spawn new ray origin
        for idx in idxs:
            rays_o[idx] = add_vectors(si.p[idx], scale_vector(rays_d[idx], 1e-4))
            active[idx] = active[idx] & si.is_valid(idx)
            depth[idx] += 1

        # Kill insignificant paths
        beta_max = max(beta)
        for idx in range(N):
            active[idx] = active[idx] & (beta_max > 0.01)

        # Russian Roulette
        if use_rr and any(active):
            rr_prob = max(beta_max, 0.1)
            rr_active = [(depth[idx] >= rr_depth) and (beta[idx] < 0.1) for idx in range(N)]
            rr_candidates = [idx for idx in range(N) if active[idx] and rr_active[idx]]
            for idx in rr_candidates:
                beta[idx] /= rr_prob
            rr_continue = [random() < rr_prob for _ in rr_candidates]
            for i, idx in enumerate(rr_candidates):
                active[idx] = active[idx] & rr_continue[i]

        # Limit depth
        for idx in range(N):
            active[idx] = active[idx] & (depth[idx] < max_depth)

    # Convert to linear space if needed
    if srgb_primitives:
        L = [srgb_to_linear(c) for c in L]
    return L if mode == 'primal' else delta_L

# Helper definitions
def Kernel(kernel_type, full_range, normalized):
    class K:
        def evaluate_density(self, o, d, t):
            # Placeholder for kernel-specific density evaluation
            return 1.0 
    return K()

def sh_eval(d, degree):
    # Standard spherical harmonic basis evaluation
    return [1.0] * (degree + 1)**2

def srgb_to_linear(c):
    return [max(0, (x/12.92) if x <= 0.04045 else ((x+0.055)/1.055)**2.4) for x in c]

def scene.ray_intersect(o, d, flags, coherent):
    class SI:
        def is_valid(self, i): return True
        def shape(self): 
            class S:
                def shape_type(self, i): return SHAPE_ELLIPSOID
                def centers(self, i): return [0,0,0]
                def rotations(self, i): return [[1,0,0],[0,1,0],[0,0,1]]
                def scales(self, i): return [1,1,1]
                def opacities(self, i): return 1.0
                def sh_coeffs(self, i): return [1.0]
            return S()
        def p(self, i): return [0,0,0]
    return SI()

ALL, BACKFACE_CULLING, SHAPE_ELLIPSOID = 1, 2, 10
def transpose(m): return [list(row) for row in zip(*m)]
def subtract(a, b): return [a[i]-b[i] for i in range(3)]
def scale_vector(v, s): return [x*s for x in v]
def dot_product(a, b): return sum(x*y for x, y in zip(a, b))
def add_vectors(a, b): return [a[i]+b[i] for i in range(3)]
def subtract_vectors(a, b): return [a[i]-b[i] for i in range(3)]
def isfinite(x): return x == x
def random(): return 0.5
```

### Edge Cases and Pitfalls
*   **Numerical Stability**: Transmission calculation clips `opacity * density` to 0.9999 to avoid $1.0 - 1.0$ numerical issues [S1, unofficial].
*   **SH Emission**: Emission calculation adds 0.5 before clamping to 0.0 to handle potential negative values from SH evaluation [S1, unofficial].
*   **Self-Intersection**: Ray origin is offset by $10^{-4}$ along the ray direction to prevent self-intersection after scattering [S1, unofficial].

## Subtopics

### [Volumetric Primitive Tomography Integrator](2542-volumetric-primitive-tomography-integrator.md)
The Volumetric Primitive Tomography Integrator is a simplified rendering method for volumetric primitives that accounts exclusively for absorption, ignoring scattering and emission. It is designed for tomography applications where the goal is to visualize density or attenuation rather than photometric appearance. Unlike its sibling [Volumetric Primitive PRB Integrator](2543-volumetric-primitive-prb-integrator.md), which handles scattering media, this integrator treats the volume as a purely absorbing medium.

### [Volumetric Primitive PRB Integrator](2543-volumetric-primitive-prb-integrator.md)
The Volumetric Primitive PRB Integrator is a path tracing method for rendering scattering and emissive media represented by mixtures of kernel-based volumetric primitives (e.g., Gaussians, Epanechnikov) rather than voxel grids or point clouds. It modifies standard volumetric path tracing by replacing medium interaction routines with primitive-specific transmittance evaluation, free-flight distance sampling, and albedo evaluation. It distinguishes itself from its sibling, the Volumetric Primitive Tomography Integrator, by supporting scattering and emissive media via multiple importance sampling with PDF balancing, whereas the tomography integrator integrates density with no scattering.

### Epanechnikov Kernel Volumetric Primitive
The Epanechnikov kernel variant uses a closed-form solution for transmittance calculation [S2, unofficial]. Free-flight distance sampling requires numerical root-finding methods (e.g., Newton-Raphson) due to the complexity of the analytical solution for multiple primitives [S2, unofficial]. The 3D Epanechnikov kernel is defined as:
$$ E_i(\mathbf{x}) = \begin{cases} \frac{15}{8\pi} (\frac{7}{3} |\Sigma_i| )^{1/2} [1 - \frac{1}{7} d(\mathbf{x})] & \text{if } d(\mathbf{x}) \le 1 \\ 0 & \text{otherwise} \end{cases} $$
where $d(\mathbf{x}) = (\mathbf{x} - \mathbf{c}_i)^T \Sigma_i^{-1} (\mathbf{x} - \mathbf{c}_i)$ [S2, unofficial]. The Epanechnikov kernel has finite support, generating more compact primitive shells than Gaussian, enabling more efficient acceleration structures and faster rendering, especially suitable for modeling sharp edges [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: Rendering speed is generally faster than MLP-based neural volume rendering (e.g., Instant-NGP) because it avoids expensive MLP evaluations per sample [S2, unofficial]. The Epanechnikov kernel is faster than the Gaussian kernel due to its finite support and compact shells [S2, unofficial].
*   **Memory Complexity**: Memory usage is proportional to the number of primitives and their associated attributes (geometry, opacity, SH coefficients). It is significantly more memory-efficient than dense voxel grids for high-resolution scenes.
*   **Strengths**:
    *   Physically based rendering (PBR) integration, supporting global illumination, soft shadows, and complex camera models [S2, unofficial].
    *   High compression rate compared to explicit voxel-based methods [S2, unofficial].
    *   Flexible control over quality vs. performance via max primitive depth [S2, unofficial].
*   **Limitations**:
    *   Requires numerical root-finding for free-flight sampling in complex overlapping scenarios [S2, unofficial].
    *   Not directly compatible with rasterization pipelines without modification (unlike 3D Gaussian Splatting) [S2, unofficial].
*   **When to Use**: Suitable for applications requiring physically accurate rendering of scattering/emissive media with high quality and flexibility, such as scientific visualization, VFX, and PBR-based training data generation.
*   **When Not to Use**: Not suitable for real-time rasterization pipelines where extreme speed is required and physical accuracy is secondary (consider 3D Gaussian Splatting). Not suitable for tomography where only absorption is needed (consider Volumetric Primitive Tomography Integrator).

## Background
The Volumetric Primitive Radiance Field Integrator originates from the work on volumetric primitives for modeling and rendering scattering and emissive media [S1, S2, unofficial]. It was developed to address the limitations of MLP-based neural radiance fields (slow evaluation) and explicit voxel-based methods (lack of PBR support) [S2, unofficial]. The method builds on the principles of neural volume rendering but replaces the implicit MLP representation with a mixture of kernel-based volumetric primitives [S1, S2, unofficial]. It is implemented in the `volprim` library by Meta Platforms, Inc. [S1, unofficial].

## Sources
[S1] facebookresearch/volumetric_primitives, https://github.com/facebookresearch/volumetric_primitives, fetched 2026-10-26 (unofficial)
[S2] Don’t Splat your Gaussians: Volumetric Ray-Traced Primitives for Modeling and Rendering Scattering and Emissive Media (2025), https://qiita.com/matumu20/items/67b33681dc97ccbcc849, fetched 2026-10-27 (unofficial)
