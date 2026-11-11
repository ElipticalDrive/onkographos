# Interval Raymarching

Also called: Affine Raymarching, Interval SDF Raymarching

## Overview
Interval Raymarching is a technique for accelerating Signed Distance Field (SDF) raymarching by evaluating the SDF over an interval of the ray using affine or interval arithmetic, rather than at a single point [S1, unofficial]. It solves the problem of slow convergence in standard sphere tracing by enabling reliable linear interpolation (secant method) to find surface intersections, aiming for the $\phi$ degree of convergence (golden ratio) instead of the linear convergence of sphere tracing [S1, unofficial]. Unlike sphere tracing, which guarantees safety via Lipschitz constraints, Interval Raymarching relies on conservative-estimate SDFs to ensure correctness when leaping across potential intersections [S1, unofficial].

## How it works
**Inputs:**
- Ray $r(t) = \mathbf{o} + t\mathbf{d}$, where $\mathbf{o}$ is origin and $\mathbf{d}$ is normalized direction.
- Conservative SDF function $f(\mathbf{p})$ (must be an underestimate of true distance to avoid missing intersections [S1, unofficial]).
- Initial interval $[t_{min}, t_{max}]$ along the ray.

**Outputs:**
- Intersection distance $t^*$ where $f(r(t^*)) \approx 0$.
- Surface normal $\mathbf{n}$ (derived from affine coefficients).

**Data Structures:**
- **Affine Form:** An affine form $\hat{f}$ representing the SDF over an interval is defined as:
  $$ \hat{f} = f_0 + \sum_{i=1}^{n} f_i \epsilon_i $$
  where $f_0$ is the central value, $f_i$ are coefficients, and $\epsilon_i$ are independent noise symbols with $\epsilon_i \in [-1, 1]$ [S1, unofficial].
- **Interval Bounds:** The range of the affine form is $[f_0 - \sum |f_i|, f_0 + \sum |f_i|]$ [S1, unofficial].

**Steps:**
1. **Interval Evaluation:** Evaluate the SDF over the current ray interval $[t_a, t_b]$ using affine arithmetic. This produces an affine form $\hat{f}$ that represents the SDF value as a linear function of position plus an error bound [S1, unofficial].
2. **Zero Check:** Determine if the interval includes zero. If $0 \notin [f_0 - \sum |f_i|, f_0 + \sum |f_i|]$, the ray does not intersect the surface within this interval. If it does, the intersection is possible [S1, unofficial].
3. **Subdivision (Optional):** If multiple zeros are possible or precision is insufficient, subdivide the interval $[t_a, t_b]$ into smaller subintervals. The subintervals containing zeros are typically much smaller than the original interval [S1, unofficial].
4. **Secant Interpolation:** Use the affine form to reliably linearly interpolate to find the zero of the SDF. This effectively applies the method of secants, leveraging the linear nature of the affine approximation within the interval [S1, unofficial].
5. **Termination:** Stop when the interval width is below a threshold $\epsilon$ or the zero is found within the desired precision [S1, unofficial].

**Parameters:**
- $\epsilon$: Epsilon size for gradient approximation, comparable to the projected pixel size to attenuate aliasing of surface bumps [S1, unofficial].

**Pitfalls:**
- **Underestimation:** If the SDF is not a conservative estimate (i.e., it underestimates the true distance), the algorithm may fail to detect intersections [S1, unofficial].
- **Precision:** Interval arithmetic can suffer from overestimation of uncertainty, leading to excessive subdivision. Reduced affine arithmetic helps mitigate this [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def interval_raymarch(ray_origin, ray_dir, sdf_func, t_min, t_max, epsilon):
    """
    Perform interval raymarching using affine arithmetic.
    
    Args:
        ray_origin: 3D vector, ray origin o
        ray_dir: 3D vector, normalized ray direction d
        sdf_func: Function that returns an affine form for the SDF over an interval
        t_min: Float, start of ray interval
        t_max: Float, end of ray interval
        epsilon: Float, convergence threshold
        
    Returns:
        t_hit: Float, distance to intersection, or infinity if no hit
        normal: 3D vector, surface normal at intersection
    """
    t_current = t_min
    t_end = t_max
    
    while t_end - t_current > epsilon:
        # Step 1: Evaluate SDF over current interval [t_current, t_end]
        # Using affine arithmetic to get an affine form
        affine_form = sdf_func.affine_eval(ray_origin, ray_dir, t_current, t_end)
        
        # Extract central value and error bound
        f0 = affine_form.central_value
        error_bound = affine_form.sum_abs_coefficients
        
        # Step 2: Check if zero is in the interval
        interval_min = f0 - error_bound
        interval_max = f0 + error_bound
        
        if interval_max < 0 or interval_min > 0:
            # No zero in this interval, skip or subdivide if necessary
            # For simplicity, assume we move forward if no intersection
            # In practice, this might involve splitting the interval
            t_current = t_end
            break
            
        # Step 3: Use affine form for secant interpolation
        # The affine form is linear: f(t) ≈ f0 + gradient * (t - t_mid)
        # We can estimate the zero by solving f(t) = 0
        # Approximate gradient from affine coefficients
        gradient = affine_form.estimate_gradient(ray_dir)
        
        if abs(gradient) < 1e-10:
            # Gradient too small, subdivide
            t_mid = (t_current + t_end) / 2
            t_end = t_mid
            continue
            
        # Secant step: t_new = t_current - f(t_current) / gradient
        # Using the central value as an approximation of f(t_current)
        t_new = t_current - f0 / gradient
        
        # Clamp t_new to the current interval
        t_new = max(t_current, min(t_new, t_end))
        
        # Update interval
        t_current = t_new
        
    # Step 5: Termination
    if t_current >= t_max:
        return float('inf'), None
        
    # Compute normal from affine coefficients
    normal = affine_form.compute_normal()
    
    return t_current, normal
```

## Operating characteristics
- **Convergence:** Aims for $\phi$ degree of convergence (golden ratio) via secant method, compared to linear convergence in sphere tracing [S1, unofficial].
- **Performance:** Potentially significantly faster than sphere tracing due to larger steps and faster convergence [S1, unofficial].
- **Memory:** Requires storage for affine coefficients, which may be larger than simple scalar values [S1, unofficial].
- **Strengths:** Faster convergence, inherent gradient approximation for lighting [S1, unofficial].
- **Limitations:** Requires conservative SDFs, potential for overestimation in interval arithmetic, complexity of implementation [S1, unofficial].

## Background
Interval Raymarching was proposed by Kragen Javier Sitaker in 2019, drawing on concepts from affine and interval arithmetic [S1, unofficial]. Related work includes Jorge Eliecer Florez Diaz's dissertation on interval arithmetic for raytracing implicit functions and Duff's 1992 work on interval arithmetic for CSG [S1, unofficial].

## Sources
[S1] Interval raymarching ⁑ Dercuano, https://dercuano.github.io/notes/interval-raymarching.html, fetched 2026-10-30 (unofficial)
