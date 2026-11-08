# Differential Sampling and Peak Finding

Also called: Adaptive Surface Detection, Peak Finding, Surface Ray Marching, Surface Raycasting with Adaptive Step Size

## Overview
Differential Sampling and Peak Finding is a volume rendering technique that combines adaptive ray sampling with explicit root-finding to render sharp, Dirac-like features in transfer functions as semi-transparent isosurfaces [S1]. Unlike standard post-classification or pre-integration methods which struggle with high-frequency transfer function features, this method solves directly for the roots of the transfer function to locate isosurface intersections, allowing for correct shading and opacity accumulation of narrow-band peaks [S1]. It sets itself apart from its siblings by treating the rendering integral as a sum of discrete impulses at peak locations rather than a continuous piecewise summation, thereby avoiding the Moiré patterns and shading errors associated with undersampling sharp features [S1].

## How it works
The method operates in two main phases: building a 2D lookup table for peak detection and executing a ray-casting loop that uses differential sampling and root-solving to render the volume [S1].

### Inputs and Outputs
*   **Inputs**: A 3D scalar volume field $V$, a piecewise-linear transfer function $T(f)$ mapping scalar values to color and opacity $\rho_\alpha$, camera parameters (origin, direction, near/far planes), and an isovalue $\tau$ (or a set of isovalue peaks derived from $T$) [S1].
*   **Outputs**: A 2D image where each pixel contains the accumulated color and opacity of the volume along the ray, with sharp peaks rendered as distinct isosurface layers [S1].

### Data Structures
1.  **Peak Lookup Table (`peakTable`)**: A 2D texture where each entry corresponds to a segment defined by entry field value $f_i$ and exit field value $f_o$ [S1]. The entry stores an isovalue $\upsilon$ (or a set of up to four isovalues $\upsilon_i$) representing peaks in the transfer function where opacity $\rho_\alpha(\upsilon)$ falls within the range $(f_i, f_o]$ (or $[f_i, f_o)$ depending on order) [S1].
    *   **Layout**: 2D texture sampled with `float2(fs0, fs1)` [S1].
    *   **Resolution**: Can use a coarser discretization (e.g., 128 bins) compared to preintegration tables [S1].
    *   **Invalid Flag**: A value outside the range of scalar values in the volume indicates no peak exists in the segment [S1].
2.  **Ray State Variables** (Global/Per-Thread):
    *   `ps0`, `ps1`: `vec4` (x, y, z, t), storing the position and time/parameter $t$ at the start and end of the current ray segment [S1].
    *   `fs0`, `fs1`: `float`, storing the scalar field values $f(\mathbf{p})$ at `ps0` and `ps1` [S1].
    *   `dt`: `float`, the step size parameter [S1].
    *   `iso`: `float`, the isovalue retrieved from the lookup table [S1].

### Algorithm Steps

#### 1. Building the Peak Lookup Table
1.  Analyze the transfer function $T(f)$ to identify local maxima (peaks) in the opacity component $\rho_\alpha$ [S1]. A point is a peak if it is greater than both immediate neighbors [S1].
2.  Construct the 2D table. For each table entry defined by range $[i, j]$:
    *   If $i < j$: Search for peaks where $\rho_\alpha(\upsilon) > i$ and $\rho_\alpha(\upsilon) \le j$ [S1].
    *   If $i > j$: Search in descending order for peaks where $\rho_\alpha(\upsilon) \le i$ and $\rho_\alpha(\upsilon) > j$ [S1].
    *   Store up to four sequential peaks in precomputed order to handle multiple peaks within a single segment [S1].
    *   If no peak exists, store an `invalid_flag` [S1].
3.  Complexity: $O(N^2)$ time, where $N$ is the number of bins [S1].

#### 2. Ray Casting Loop (Differential Sampling)
1.  **Initialization**:
    *   Set initial step size `dt = b` [S1].
    *   Set `ps1.w` (time/parameter $t$) to 0 [S1].
    *   Set `ps1.xyz` to `ray.origin` [S1].
    *   Compute `fs1 = volume_filter(ps1.xyz)` [S1].
2.  **Iteration**:
    *   Copy `ps1` to `ps0` and `fs1` to `fs0` [S1].
    *   Advance position: `ps1 += dt * ray.direction` [S1].
    *   **Differential Sampling**: If `DIFFERENTIAL_SAMPLING` is enabled, increment `dt` by `a` (adaptive step size adjustment based on image-space derivatives) [S1].
    *   Compute `fs1 = volume_filter(ps1.xyz)` [S1].
    *   **Peak Lookup**: If `!DVR_ONLY`, sample `peakTable` using `float2(fs0, fs1)` to get `iso` [S1].
    *   **Peak Finding**: If `iso != invalid_flag`, call `peak_find()` [S1].
    *   **Volume Rendering Sample**: If `!PEAK_ONLY`, if `PEAK_XOR_DVR` is set and peak finding was called, skip standard sampling; otherwise call `dvr_sample(ps1, fs1, dt)` [S1].
    *   **Termination**: If `color.a > termination_alpha`, break loop [S1].

#### 3. Peak Finding (Root Solving)
When a peak is detected in the lookup table, the method solves for the exact intersection point using a secant method variant [S1].

1.  **Bracketing**:
    *   Compute $f_0 = f_s0 - \tau$ and $f_1 = f_s1 - \tau$ [S1].
    *   Employ Descartes' rule of signs as a preliminary test; if it succeeds (or if $f_0 \cdot f_1 < 0$), assume the segment contains a root and set $p_0 = ps0$ and $p_1 = ps1$ [S1].
2.  **Secant Iteration**:
    *   Iterate 3 times [S1].
    *   Compute intermediate point:
        $$ p_{rt} = p_0 - (p_1 - p_0) \frac{f_0}{f_1 - f_0} $$
    *   Evaluate field at intermediate point: $f_{rt} = \text{volume\_filter}(p_{rt}.xyz) - \tau$ [S1].
    *   Update bounds:
        *   If $f_{rt} \cdot f_0 < 0$, set $p_1 = p_{rt}$, $f_1 = f_{rt}$ [S1].
        *   Else, set $p_0 = p_{rt}$, $f_0 = f_{rt}$ [S1].
3.  **Final Sample**:
    *   Recompute $p_{rt}$ using the final $p_0, p_1, f_0, f_1$ [S1].
    *   Set $f_{rt} = \tau$ (assume the value is the desired isovalue to avoid Moiré patterns) [S1].
    *   Set `dt = 1.0` (constant step size for opacity scaling) [S1].
    *   Call `dvr_sample(p_{rt}, f_{rt}, dt)` [S1].

#### 4. Volume Rendering Sample (`dvr_sample`)
*   Computes the color and opacity contribution at the given position [S1].
*   If rendering a peak, uses `dt = 1.0` to scale opacity, assuming sampling at the isovalue regardless of local sampling rate [S1].
*   If rendering standard DVR, uses the current `dt` [S1].
*   Lighting is computed using a Blinn-Phong model:
    $$ I_a = 0.1 $$
    $$ I_d = 1.0 \cdot \max(0, \mathbf{N} \cdot \mathbf{L}) $$
    $$ I_s = 8.0 \cdot \max(0, \mathbf{N} \cdot \mathbf{H})^{600} $$
    $$ \text{colour} = (I_a + I_d) \cdot \text{material\_colour} + I_s \cdot (1,1,1) $$
    where $\mathbf{L}$ is the normalized light direction, $\mathbf{V}$ is the normalized view direction ($-\text{ray}$), $\mathbf{H}$ is the normalized half-vector ($\mathbf{L}+\mathbf{V}$), and $\mathbf{N}$ is the surface normal computed from the gradient [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Parameters
b = 0.5              # Initial step size
a = 0.01             # Step size increment for differential sampling
termination_alpha = 0.95 # Alpha threshold for ray termination
invalid_flag = -1.0  # Value indicating no peak found
DVR_ONLY = False
PEAK_ONLY = False
PEAK_XOR_DVR = False
use_differential_sampling = True

# Globals
ps0 = vec4(0,0,0,0)
ps1 = vec4(0,0,0,0)
fs0 = 0.0
fs1 = 0.0
dt = b
iso = 0.0
color = vec4(0,0,0,0)

def volume_filter(pos: vec3) -> float:
    # Standard trilinear/tricubic texture lookup
    return texture(volume_texture, pos).r

def dvr_sample(pos: vec4, field_val: float, step: float):
    # Compute lighting (Blinn-Phong) and opacity contribution
    # Accumulate into global 'color' using front-to-back compositing
    pass

def peak_find():
    f0 = fs0 - iso
    f1 = fs1 - iso
    # Descartes' rule of signs or sign change check
    if f0 * f1 < 0:
        p0 = ps0
        p1 = ps1
        # Secant method: 3 iterations
        for _ in range(3):
            direction = p1 - p0
            fraction = f0 / (f1 - f0)
            prt = p0 - direction * fraction
            frt = volume_filter(prt.xyz) - iso
            if frt * f0 < 0:
                p1 = prt
                f1 = frt
            else:
                p0 = prt
                f0 = frt
        # Final sample
        direction = p1 - p0
        fraction = f0 / (f1 - f0)
        prt = p0 - direction * fraction
        frt = iso # Assume isovalue to avoid Moiré
        dt = 1.0  # Constant step for opacity scaling
        dvr_sample(prt, frt, dt)

def raycast(ray_origin: vec3, ray_direction: vec3):
    global color, dt, ps0, ps1, fs0, fs1
    color = vec4(0,0,0,0)
    dt = b
    ps1 = vec4(ray_origin, 0.0)
    fs1 = volume_filter(ps1.xyz)
    
    while True:
        ps0 = ps1
        fs0 = fs1
        ps1 = ps1 + dt * vec4(ray_direction, 0.0)
        
        # Differential sampling adjustment
        if use_differential_sampling:
            dt += a
        
        fs1 = volume_filter(ps1.xyz)
        
        # Peak lookup and finding
        if not DVR_ONLY:
            iso = peak_table_lookup(fs0, fs1)
            if iso != invalid_flag:
                peak_find()
        
        # Standard DVR sampling
        if not PEAK_ONLY:
            if PEAK_XOR_DVR and iso != invalid_flag:
                pass # Skip DVR sample
            else:
                dvr_sample(ps1, fs1, dt)
                
        if color.a > termination_alpha:
            break
```

### Parameters
*   `termination_alpha`: Alpha threshold for ray termination [S1].
*   `invalid_flag`: Value indicating no peak found in the lookup table [S1].
*   `b`: Initial step size [S1].
*   `a`: Step size increment for differential sampling [S1].
*   `iso`: Isovalue retrieved from the lookup table [S1].

### Edge Cases and Pitfalls
*   **Moire Patterns**: Avoid evaluating the field $f(\mathbf{R}(t))$ at the root; assume the value is the isovalue to prevent artifacts [S1].
*   **Opacity Scaling**: Do not scale opacity $\rho_\alpha$ by the segment distance $\Delta t$ when rendering peaks; use a constant $\Delta t = 1$ [S1].
*   **Multiple Peaks**: Handle up to four peaks per segment by storing them in the lookup table and processing sequentially [S1].

## Subtopics

### [Coarse-to-Fine Volume Maximum Search](4657-coarse-to-fine-volume-maximum-search.md)
A two-stage search algorithm to find the approximate location of the maximum value in a volume along a ray. The first stage performs a coarse linear scan, and the second stage refines the position using a divide-by-two approach. It sets itself apart from sibling methods by combining a global linear sweep with a localized binary-search-like refinement, prioritizing robustness in finding the global maximum within a segment over pure convergence speed.

## Operating characteristics
*   **Time Complexity**: Building the lookup table is $O(N^2)$, where $N$ is the number of bins [S1].
*   **Memory Complexity**: Requires a 2D lookup table. A coarser resolution (128 bins) is often sufficient for peak finding, reducing memory usage compared to preintegration [S1].
*   **Performance**:
    *   Building the peak-finding table is roughly twice as fast as building a preintegrated table at the same resolution [S1].
    *   Heptane fire scene (1024x900): Peak finding renders at 7.1 fps, postclassification at 11.6 fps, preintegration at 8.8 fps [S1].
    *   Zebrafish optic tract (1600x512): Peak finding at 2.1 fps, Preintegration at 2.0 fps, Semi-transparent isosurfacing at 4.3 fps [S1].
    *   Finding multiple peaks costs roughly 20% more performance [S1].
*   **Strengths**:
    *   Correctly renders sharp, Dirac-like features in transfer functions [S1].
    *   Avoids Moiré patterns and shading errors associated with preintegration [S1].
    *   Supports different rendering modalities (PEAK_XOR_DVR, PEAK_ONLY, DVR_ONLY) with no performance penalty [S1].
*   **Limitations**:
    *   Introduces bias by assuming the integral can be approximated by discrete impulses [S1].
    *   May miss features if the transfer function is not adequately sampled or if peaks are not local maxima in opacity [S1].
*   **When to Use**: Suited for rendering sharp, high-frequency transfer function features [S1].
*   **When Not to Use**: When smooth, continuous transfer functions are used, standard postclassification or preintegration may be more appropriate [S1].

## Background
*   **Origin**: The method was introduced in the paper "Volume Ray Casting with Peak Finding and Differential Sampling" [S1].
*   **History**: It was developed to address the limitations of postclassification and preintegration in rendering sharp transfer function features. The technique borrows from ray differentials formulations of Igehy [8] for sampling strategy and uses Descartes' rule of signs and secant methods for root solving [S1].
*   **Usage**: Used in interactive high-fidelity volume rendering systems, particularly for scientific visualization of noisy or entropic data [S1].

## Sources
[S1] Volume Ray Casting with Peak Finding and Differential Sampling, https://www.sci.utah.edu/~knolla/avrc.pdf, fetched 2026-10-27
[S2] GitHub - m-pilia/volume-raycasting: Simple visualiser based on GPU ..., https://github.com/m-pilia/volume-raycasting, fetched 2026-10-27 (unofficial)
