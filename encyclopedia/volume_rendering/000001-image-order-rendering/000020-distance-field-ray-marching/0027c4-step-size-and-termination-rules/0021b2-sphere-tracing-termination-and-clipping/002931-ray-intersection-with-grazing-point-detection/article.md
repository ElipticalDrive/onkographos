# Ray Intersection with Grazing Point Detection


## Overview
A ray intersection method that performs sphere tracing to find the primary intersection while simultaneously tracking the 'grazing point' (the point on the surface where the ray direction is tangent to the surface normal, indicated by a zero directional derivative) [S1, unofficial]. It returns the intersection time, validity, grazing point status, and the grazing point position. It sets itself apart from standard sphere tracing by explicitly identifying and refining grazing contacts, which are often missed or poorly resolved by standard termination rules [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray `ray` (origin `o`, direction `d`, maxt `maxt`), a Signed Distance Field (SDF) evaluator `self.at(p)`, and a directional derivative evaluator `self.dir_deriv(p, d)`.
*   **Output:** Intersection time `t`, validity flag `is_valid`, grazing point position `grz`, and grazing time `grz_t`.

**Data Structures:**
*   `ray`: Struct containing `o` (Point3f), `d` (Vector3f, normalized), `maxt` (float).
*   `self`: Object providing `at(p)` returning float SDF, and `dir_deriv(p, d)` returning float directional derivative.

**Parameters:**
*   `ITX_REFINE`: Half-width of the interval for final intersection refinement. Value from `utils.ITX_REFINE` [S1, unofficial].
*   `ITX_EPS`: Minimum signed distance threshold for valid intersection/grazing candidate. Value from `utils.ITX_EPS` [S1, unofficial].
*   `SDF_EPS`: Threshold for absolute SDF value to confirm grazing point lies on surface. Value from `utils.SDF_EPS` [S1, unofficial].
*   `SDF_DERIV_EPS`: Threshold for absolute directional derivative to confirm tangency. Value from `utils.SDF_DERIV_EPS` [S1, unofficial].
*   `NUM_GRZ`: Maximum iterations for grazing point refinement. Value from `utils.NUM_GRZ` [S1, unofficial].
*   `GRZ_MODE`: String, either `'secant'` or `'bisection'`, determining the refinement method for grazing points [S1, unofficial].

**Steps:**
1.  **Initialization:**
    *   Copy input ray parameters.
    *   Set `t = 0`.
    *   Set flags: `no_itx = True` (no intersection found), `in_range = True` (ray within bounds), `pos_dir = True` (directional derivative positive).
    *   Set `grz_t = 0` (grazing time).
    *   Set `min_sdf = 1` (minimum SDF encountered for grazing candidates).
    *   Check bounding box intersection:
        *   If ray origin is inside the bounding box, set `t = 0`.
        *   Otherwise, set `t = mint + RAY_EPS` (where `mint` is the entry time into the bounding box).
        *   Clamp `ray.maxt` to the bounding box max intersection time.

2.  **Sphere Tracing Loop:**
    *   Enter a loop continuing while `no_itx` is True AND `in_range` is True.
    *   Suspend gradients during this loop [S1, unofficial].
    *   Evaluate point `p = ray(t)`.
    *   Evaluate signed distance `d = self.at(p)`.
    *   Evaluate directional derivative `dir = self.dir_deriv(p, ray.d)`.
    *   **Identify Grazing Candidates:**
        *   Create mask: `idx = (~pos_dir) & (dir > 0) & (d < min_sdf) & (d > ITX_EPS)`.
        *   This identifies points where the ray was previously moving away (`~pos_dir`), is now moving towards the surface (`dir > 0`), has a smaller SDF than previous candidates (`d < min_sdf`), and is not too close to the surface (`d > ITX_EPS`).
        *   Update `grz_t` and `min_sdf` for indices satisfying `idx`.
    *   **Advance Ray:**
        *   `t += d`.
        *   Update `pos_dir = (dir > 0)`.
        *   Update `no_itx = (d > ITX_EPS)`.
        *   Update `in_range = self.in_range(p) & (t < ray.maxt)`.

3.  **Grazing Point Refinement:**
    *   After the loop, call `self.get_grz(ray, grz_t - min_sdf, grz_t + min_sdf)` to refine the grazing point position [S1, unofficial].
    *   The `get_grz` method uses either secant or bisection iteration (see `get_grz_secant` or `get_grz_bisection` in [S1, unofficial]) to find the root of the directional derivative within the interval.
    *   **Verification:**
        *   Verify the grazing point `grz` is valid if:
            *   `abs(self.at(grz)) < SDF_EPS`
            *   `abs(self.dir_deriv(grz, ray.d)) < SDF_DERIV_EPS`
            *   `self.at(grz) > ITX_EPS`

4.  **Intersection Refinement and Finalization:**
    *   Determine ray validity: `is_valid = (~no_itx) & in_range`.
    *   If `is_valid` is False, set `t = infinity`.
    *   If `is_valid` is True, refine the intersection time using `self.refine_intersect(ray, t - ITX_REFINE, t + ITX_REFINE)` [S1, unofficial].
    *   The `refine_intersect` method uses bisection iteration (see [S1, unofficial]) to find the root of the SDF within the interval.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_intersect_grz(ray, sdf_eval, dir_deriv_eval, utils):
    """
    Ray intersection with grazing point detection.
    
    Args:
        ray: Ray object with o, d, maxt.
        sdf_eval: Function self.at(p) returning SDF.
        dir_deriv_eval: Function self.dir_deriv(p, d) returning directional derivative.
        utils: Object containing ITX_REFINE, ITX_EPS, SDF_EPS, SDF_DERIV_EPS, NUM_GRZ, GRZ_MODE.
        
    Returns:
        t: Intersection time.
        is_valid: Boolean indicating valid intersection.
        grz: Grazing point position.
        grz_valid: Boolean indicating valid grazing point.
    """
    t = 0.0
    no_itx = True
    in_range = True
    pos_dir = True
    grz_t = 0.0
    min_sdf = 1.0
    
    # Check bounding box intersection
    if ray.o inside bounding_box:
        t = 0.0
    else:
        t = mint + RAY_EPS # RAY_EPS is a small constant, e.g., 1e-6
        ray.maxt = min(ray.maxt, maxt_box)
        
    # Suspend gradients during loop
    with suspend_grad():
        while no_itx and in_range:
            p = ray.o + ray.d * t
            d = sdf_eval(p)
            dir = dir_deriv_eval(p, ray.d)
            
            # Identify potential grazing points
            # Mask: not previously moving away, now moving towards, smaller SDF, not too close
            if not pos_dir and dir > 0 and d < min_sdf and d > utils.ITX_EPS:
                grz_t = t
                min_sdf = d
                
            # Advance ray
            t += d
            pos_dir = (dir > 0)
            no_itx = (d > utils.ITX_EPS)
            in_range = in_range(p) and (t < ray.maxt)
            
    # Refine grazing point
    grz = get_grz(ray, grz_t - min_sdf, grz_t + min_sdf, sdf_eval, dir_deriv_eval, utils)
    
    # Verify grazing point
    grz_valid = (abs(sdf_eval(grz)) < utils.SDF_EPS) and \
                (abs(dir_deriv_eval(grz, ray.d)) < utils.SDF_DERIV_EPS) and \
                (sdf_eval(grz) > utils.ITX_EPS)
                
    # Determine ray validity
    is_valid = (not no_itx) and in_range
    
    if not is_valid:
        t = float('inf')
    else:
        # Refine intersection
        t = refine_intersect(ray, t - utils.ITX_REFINE, t + utils.ITX_REFINE, sdf_eval, utils)
        
    return t, is_valid, grz, grz_valid

def get_grz(ray, t_min, t_max, sdf_eval, dir_deriv_eval, utils):
    """Refine grazing point using secant or bisection method."""
    if utils.GRZ_MODE == 'secant':
        return get_grz_secant(ray, t_min, t_max, sdf_eval, dir_deriv_eval, utils)
    elif utils.GRZ_MODE == 'bisection':
        return get_grz_bisection(ray, t_min, t_max, sdf_eval, dir_deriv_eval, utils)
    else:
        raise ValueError(f"Unsupported GRZ_MODE: {utils.GRZ_MODE}")

def get_grz_secant(ray, t_min, t_max, sdf_eval, dir_deriv_eval, utils):
    """Secant method for finding the grazing point."""
    d = ray.d
    L = ray(t_min)
    R = ray(t_max)
    cnt = 0
    with suspend_grad():
        while cnt < utils.NUM_GRZ:
            dL = dir_deriv_eval(L, d)
            dR = dir_deriv_eval(R, d)
            # Avoid division by zero
            if abs(dR - dL) < 1e-12:
                break
            M = R - dR * (R - L) / (dR - dL)
            dM = dir_deriv_eval(M, d)
            L[dM < 0] = M
            R[dM > 0] = M
            cnt += 1
    return L

def get_grz_bisection(ray, t_min, t_max, sdf_eval, dir_deriv_eval, utils):
    """Bisection method for finding the grazing point."""
    d = ray.d
    L = ray(t_min)
    R = ray(t_max)
    cnt = 0
    with suspend_grad():
        while cnt < utils.NUM_GRZ:
            M = (L + R) / 2
            dM = dir_deriv_eval(M, d)
            L[dM < 0] = M
            R[dM > 0] = M
            cnt += 1
    return L

def refine_intersect(ray, t_min, t_max, sdf_eval, utils):
    """Bisection method for refining the intersection point."""
    if utils.NUM_REFINE == 0:
        return (t_min + t_max) / 2
        
    L = t_min
    R = t_max
    cnt = 0
    with suspend_grad():
        while cnt < utils.NUM_REFINE:
            dL = sdf_eval(ray(L))
            dR = sdf_eval(ray(R))
            M = (L + R) / 2
            dM = sdf_eval(ray(M))
            L[dM > 0] = M
            R[dM < 0] = M
            cnt += 1
    return (L + R) / 2
```

## Subtopics

### Screen-space aware intersection point selection
A method for selecting an appropriate intersection point when sphere tracing fails to converge (e.g., maximum iterations exceeded), particularly when a ray grazes an object's edge [S2, unofficial]. It selects the point along the ray with the smallest unbounding sphere radius, as measured in screen-space [S2, unofficial]. It terminates the ray early if the unbounding sphere radius is smaller than one half the size of a pixel [S2, unofficial]. Parameters include `pixelRadius` (radius of a pixel at t = 1) and `forceHit` (boolean enforcing the use of the candidate_t value as the result) [S2, unofficial]. It uses an over-relaxation sphere tracing step length calculation with a fallback to bisection-like refinement if the step length decreases significantly [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The main loop runs until intersection or max iterations. The grazing point refinement adds a constant number of iterations (`NUM_GRZ`) per ray. The intersection refinement adds `NUM_REFINE` iterations. Overall, it is comparable to standard sphere tracing but with a small constant overhead for grazing detection and refinement.
*   **Memory Complexity:** O(1) additional memory per ray, storing a few scalar values and flags.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales linearly with the number of rays. The constant overhead of grazing detection may become more significant for scenes with many grazing rays.
*   **Strengths:** Explicitly handles grazing contacts, which are often missed or poorly resolved by standard sphere tracing. Provides a fallback grazing point even if the primary intersection is not found.
*   **Limitations:** Requires evaluation of the directional derivative, which may be more expensive than the SDF itself. The coarse sphere-traced grazing time `grz_t` is insufficiently accurate and requires root-finding refinement [S1, unofficial].
*   **When to use:** When accurate handling of grazing contacts is important, such as in high-quality rendering or when dealing with complex geometry where rays frequently skim surfaces.
*   **When not to use:** When performance is critical and grazing contacts are rare or can be ignored. When the directional derivative is not available or expensive to compute.

## Background
Originated from the implementation in `zichenwang01/relaxed-boundary` [S1, unofficial]. The method is designed to handle the challenges of sphere tracing near object boundaries, particularly grazing contacts. The screen-space aware intersection point selection variant is described in "Enhanced Sphere Tracing" [S2, unofficial].

## Sources
[S1] zichenwang01/relaxed-boundary, https://github.com/zichenwang01/relaxed-boundary, fetched 2026-10-27 (unofficial)
[S2] Enhanced Sphere Tracing, https://kev.town/raymarching-toolkit/media/enhanced_sphere_tracing.pdf, fetched 2026-10-27 (unofficial)
