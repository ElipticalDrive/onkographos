# General Multidimensional Transfer Function Peak Finding

Also called: Dynamic Peak Finding in Transfer Space, Multidimensional Peak Finding, Peak finding in higher dimensions, ρ-space Peak Sampling

## Overview
General Multidimensional Transfer Function Peak Finding is a direct volume rendering technique that samples directly at local maxima along the image of the ray in multidimensional transfer function space, rather than at the maxima of the transfer function itself [S1]. It solves for the spatial location of isosurfaces corresponding to these peaks to accurately render high-frequency features and noisy data at lower sampling rates [S1]. Unlike 1D peak finding, which finds maxima of the transfer function, this method finds local maxima along the approximated image of the ray, which usually do not coincide with the maxima of the N-dimensional function [S1]. It sets itself apart from its siblings by using ray marching to reduce the search in transfer space to 1D regardless of the dimension of classification, allowing accurate rendering of noisy data and precise transfer functions at lower sampling costs [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A multidimensional transfer function $\rho(\mathbf{f})$ where $\mathbf{f} = (f_0, \dots, f_{M-1})$ is the vector of scalar data values at a point, and $M > 1$ is the dimensionality [S1]. A ray defined by origin $\mathbf{O}$ and direction $\mathbf{D}$ [G]. World-space sampling rate $\Delta t$ [S1]. Transfer function space sampling rate $\Delta s$ [S1].
*   **Output:** The color and opacity for each pixel, derived from integration along the ray with samples taken at peak locations [G].

**Data Structures:**
*   **Transfer Function:** Evaluated as a function $\rho: \mathbb{R}^M \to \mathbb{R}$ [S1]. For higher dimensions, it may be constructed by convolving lower-dimensional functions, e.g., $\rho(\mathbf{f}) = \nu(f_0, f_1) * \mu(f_2, f_3)$ [S1].
*   **Ray Segment:** Defined by world-space endpoints $\mathbf{f}_0 = \mathbf{f}(t_0)$ and $\mathbf{f}_1 = \mathbf{f}(t_1)$ [S1].

**Steps:**
1.  **World-Space Sampling:** Sample the scalar data $\mathbf{f}(t)$ along the ray at intervals determined by $\Delta t$. Let the samples be $\mathbf{f}_i = \mathbf{f}(t_i)$ [S1].
2.  **Transfer Space Approximation:** For each segment between world-space samples $[\mathbf{f}_0, \mathbf{f}_1]$, parameterize an approximation of the segment in transfer space [S1]. The sources propose using chord parameterization (linear interpolation between $\mathbf{f}_0$ and $\mathbf{f}_1$ in transfer space) or spline parameterization [S1]. The chord approximation is defined by sampling $\rho$ along the linear interpolation of $\mathbf{f}$ in world space, but evaluating $\rho$ at those points [S1].
3.  **Transfer Space Sampling:** Sample the transfer function $\rho$ along the approximated segment in transfer space at intervals determined by $\Delta s$ [S1]. The sampling rate $\Delta s$ controls behavior and acceleration [S1]. The requirement is that $\rho$ is monotonic on each sub-segment $[\mathbf{f}_0, \mathbf{f}_1]$; this is a less stringent requirement than the Nyquist limit in $\rho$-space [S1].
4.  **Peak Detection:** Query the transfer function along these segments to identify a local maximum [S1]. Let this local maximum be $\tilde{\mathbf{p}}$ [S1].
5.  **Isosurface Solving:** Use the identified peak to solve for the isosurface root $t_\rho$ along the world-space ray [S1]. The value $t_\rho$ is the root of the isosurface along the world-space ray corresponding to the peak value [S1].
6.  **Shading:** Use $\mathbf{R}(t_\rho)$ as the position at which to shade [S1]. For multifield data, choose a gradient for shading: Option 1 is shading multiple data gradients $\nabla f_0, \nabla f_1, \dots$ separately using multiple central-difference neighbor stencils [S1]. Option 2 is computing the gradient $\nabla \rho(\mathbf{f})$ of the transfer function, classifying $\mathbf{f}$ at each point of a single central-difference stencil [S1]. The first approach is preferred because $\nabla \mathbf{f}$ tends to exhibit higher frequency than separate individual gradients [S1].
7.  **Integration:** Integrate along the ray using the samples at $t_\rho$ and standard samples, similar to 1D peak finding but adapted for the multidimensional context [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_multidimensional_peak_finding(ray_origin, ray_direction, ray_length, transfer_function, delta_t, delta_s):
    # Helper: vector operations
    def vec_add(a, b): return tuple(a[i] + b[i] for i in range(len(a)))
    def vec_scale(v, s): return tuple(v[i] * s for i in range(len(v)))
    def lerp(a, b, t): return tuple(a[i] + (b[i] - a[i]) * t for i in range(len(a)))
    def dist_sq(a, b): return sum((a[i] - b[i])**2 for i in range(len(a)))
    def sqrt(x): return x ** 0.5

    def evaluate_data(pos):
        # Standard volume texture lookup [G]
        return (0.5, 0.5, 0.5, 0.5)

    def find_local_max(points):
        # Linear scan for local maximum [G]
        best_alpha, best_rho = points[0]
        for alpha, rho in points:
            if rho > best_rho:
                best_alpha, best_rho = alpha, rho
        return best_alpha, best_rho

    def compute_gradient(f, neighbor):
        # Central difference approximation [G]
        return tuple((f[i] - neighbor[i]) for i in range(len(f)))

    def integrate_sample(color, t_rho, rho, grad0, grad1):
        # Standard DVR accumulation [G]
        return tuple(color[i] + rho * grad0[i] * grad1[i] for i in range(4))

    neighbor0 = (1.0, 0.0, 0.0)
    neighbor1 = (0.0, 1.0, 0.0)

    t = 0.0
    samples = []
    while t < ray_length:
        f = evaluate_data(vec_add(ray_origin, vec_scale(ray_direction, t)))
        samples.append((t, f))
        t += delta_t

    final_color = (0.0, 0.0, 0.0, 0.0)

    for i in range(len(samples) - 1):
        t0, f0 = samples[i]
        t1, f1 = samples[i+1]

        seg_len = sqrt(dist_sq(f0, f1))
        num_samples = max(2, int(seg_len / delta_s) + 1)

        rho_values = []
        for k in range(num_samples + 1):
            alpha = k / num_samples
            f_interp = lerp(f0, f1, alpha)
            rho_val = transfer_function.evaluate(f_interp)
            rho_values.append((alpha, rho_val))

        max_alpha, max_rho = find_local_max(rho_values)
        t_rho = lerp(t0, t1, max_alpha)

        f_at_peak = evaluate_data(vec_add(ray_origin, vec_scale(ray_direction, t_rho)))
        grad0 = compute_gradient(f0, neighbor0)
        grad1 = compute_gradient(f1, neighbor1)

        final_color = integrate_sample(final_color, t_rho, max_rho, grad0, grad1)

    return final_color
```

**Parameters:**
*   $\Delta t$: World-space sampling rate. Typical values: $\Delta t = 4$ for DDA slabs, $\Delta t = 0.5$ for postclassified examples [S1].
*   $\Delta s$: Transfer function space sampling rate. Typical value: $\Delta s = 2$ [S1].

**Edge Cases and Pitfalls:**
*   **Monotonicity:** Multidimensional peak finding is expensive and unnecessary wherever the transfer function $\mathbf{f}$ is monotonic, as it must sample transfer space between every world-space segment [S1].
*   **Shading Ambiguities:** Using bisection for multiple dimensions can yield shading ambiguities when peaks are close together or when the assumption of one peak per segment fails [S1].
*   **High Frequencies:** High frequencies occur more easily in multidimensional space than in 1D scalar fields, even with relatively low-frequency transfer functions due to convolution [S1].
*   **Peak Existence:** Unlike 1D, a peak must be looked for on each segment because it is not known in advance if one exists in $[\mathbf{f}_0, \mathbf{f}_1]$ [S1].

## Subtopics

### [Chord Parameterization](14542-chord-parameterization.md)
Chord Parameterization is a method for finding local maxima in multidimensional transfer functions by approximating the path between two world-space samples as a straight line (chord) in transfer function space and sampling along it. It sets itself apart from its parent [General Multidimensional Transfer Function Peak Finding](14527-general-multidimensional-transfer-function-peak-finding.md) by providing a specific, efficient geometric approximation for the search path, and from [Spline Parameterization](14543-spline-parameterization.md) by using linear interpolation instead of curved spline fits.

### Separable Transfer Function Peak Finding
A technique for multidimensional transfer functions that are separable into 1D functions, using 1D peak finding with modifications [S1]. It assumes one peak per ray segment and employs one isosurface solving routine instead of separate solvers for separate dimensions [S1]. It uses bisection to determine whether a root for any dimension exists in the left or right half of a bracket, instead of the secant method [S1]. Shading ambiguities may occur when roots exist in both subfields of $\mathbf{f}$, but in practice the quality is good and performance matches 1D peak finding [S1]. This trivial extension only works for a small subset of multidimensional classifications [S1].

### Integration and Shading for Multifield DVR
This subtopic addresses how to handle shading when using multidimensional peak finding [S1]. It uses $t_\rho$ as the root of the isosurface along the world-space ray and $\mathbf{R}(t_\rho)$ as the position at which to shade [S1]. Two principal options exist for choosing the gradient of a multifield volume $\mathbf{f}$ when shading:
1.  Shading multiple data gradients $\nabla f_0, \nabla f_1, \dots$ separately, using multiple central-difference neighbor stencils [S1].
2.  Computing the gradient $\nabla \rho(\mathbf{f})$ of the transfer function, classifying $\mathbf{f}$ at each point of a single central-difference stencil [S1].
Both approaches are expensive, and are responsible for a significant share of the cost of multifield DVR regardless of whether peak finding is used or not [S1]. The first approach is preferred because $\nabla \mathbf{f}$ tends to exhibit higher frequency than separate individual gradients [S1].

## Operating characteristics
*   **Time Complexity:** Linear per ray segment in the worst case, but average-case is constant time per segment assuming $\rho$ is Lipschitz [S1].
*   **Memory Complexity:** No precomputation or storage requirements are needed; it is an alternative to preintegration which requires lookup tables [S1].
*   **Measured Performance:** In a 4-channel combustion simulation visualization at 1200 × 600 resolution, postclassified renderings with $\Delta t = 0.5$ rendered at 0.8 and 0.75 fps, while peak finding with chordal ray marching ($\Delta t = 1, \Delta s = 2$) rendered at 1.6 fps [S1].
*   **Strengths:** Produces comparable image quality with order-of-magnitude better performance than standard classification, and can reproduce features omitted entirely by standard classification [S1]. It ensures scale-invariance [S1].
*   **Limitations:** Multidimensional peak finding is expensive and unnecessary wherever the transfer function $\mathbf{f}$ is monotonic [S1]. High frequencies occur more easily in multidimensional space [S1].
*   **When to Use:** When rendering noisy volume data or specifying more precise transfer functions in multidimensional space [S1].
*   **When Not to Use:** Scale-invariant volume rendering with peak finding may not be the best modality when isosurfaces cause occlusion [S1].

## Background
The method was introduced in the paper "Volume Rendering with Multidimensional Peak Finding" [S1]. It extends the 1D peak finding technique to higher-dimensional transfer functions (e.g., 2D, 4D) [S1].

## Sources
[S1] Volume Rendering with Multidimensional Peak Finding, https://www.sci.utah.edu/~knolla/multipeak_pv.pdf, fetched 2026-10-29
