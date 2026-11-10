# Ray Intersection with Grazing Point Detection


## Overview
A ray intersection method that performs sphere tracing to find the primary intersection while simultaneously tracking the 'grazing point' (the point on the surface where the ray direction is tangent to the surface normal, indicated by a zero directional derivative) [S1, unofficial]. It returns the intersection time, validity, grazing point status, and the grazing point position [S1, unofficial]. It sets itself apart from standard sphere tracing by explicitly identifying and refining grazing contacts, which are often missed or poorly resolved by standard termination rules [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray `ray` (origin `o`, direction `d`, maxt `maxt`), a Signed Distance Field (SDF) evaluator `self.at(p)`, and a directional derivative evaluator `self.dir_deriv(p, d)` [S1, unofficial].
*   **Output:** Intersection time `t`, validity flag `is_valid`, grazing point position `grz`, and grazing time `grz_t` [S1, unofficial].

**Data Structures:**
*   `ray`: Struct containing `o` (Point3f), `d` (Vector3f, normalized), `maxt` (float) [S1, unofficial].
*   `self`: Object providing `at(p)` returning float SDF, and `dir_deriv(p, d)` returning float directional derivative [S1, unofficial].

**Parameters:**
*   `ITX_REFINE`: Half-width of the interval for final intersection refinement. Value from `utils.ITX_REFINE` [S1, unofficial].
*   `ITX_EPS`: Minimum signed distance threshold for valid intersection/grazing candidate. Value from `utils.ITX_EPS` [S1, unofficial].
*   `SDF_EPS`: Threshold for absolute SDF value to confirm grazing point lies on surface. Value from `utils.SDF_EPS` [S1, unofficial].
*   `SDF_DERIV_EPS`: Threshold for absolute directional derivative to confirm tangency. Value from `utils.SDF_DERIV_EPS` [S1, unofficial].
*   `NUM_GRZ`: Maximum iterations for grazing point refinement. Value from `utils.NUM_GRZ` [S1, unofficial].
*   `GRZ_MODE`: String, either `'secant'` or `'bisection'`, determining the refinement method for grazing points [S1, unofficial].

**Steps:**
1.  **Initialization:**
    *   Copy input ray parameters [S1, unofficial].
    *   Set `t = 0` [S1, unofficial].
    *   Set flags: `no_itx = True` (no intersection found), `in_range = True` (ray within bounds), `pos_dir = True` (directional derivative positive) [S1, unofficial].
    *   Set `grz_t = 0` (grazing time) [S1, unofficial].
    *   Set `min_sdf = 1` (minimum SDF encountered for grazing candidates) [S1, unofficial].
    *   Check bounding box intersection: If ray origin is inside the bounding box, set `t = 0`; otherwise, set `t = mint + RAY_EPS` (where `mint` is the entry time into the bounding box) [S1, unofficial].
    *   Clamp `ray.maxt` to the bounding box max intersection time [S1, unofficial].

2.  **Sphere Tracing Loop:**
    *   Enter a loop continuing while `no_itx` is True AND `in_range` is True [S1, unofficial].
    *   Suspend gradients during this loop [S1, unofficial].
    *   Evaluate point `p = ray(t)` [S1, unofficial].
    *   Evaluate signed distance `d = self.at(p)` [S1, unofficial].
    *   Evaluate directional derivative `dir = self.dir_deriv(p, ray.d)` [S1, unofficial].
    *   **Identify Grazing Candidates:**
        *   Create mask: `idx = (~pos_dir) & (dir > 0) & (d < min_sdf) & (d > ITX_EPS)` [S1, unofficial].
        *   This identifies points where the ray was previously moving away (`~pos_dir`), is now moving towards the surface (`dir > 0`), has a smaller SDF than previous candidates (`d < min_sdf`), and is not too close to the surface (`d > ITX_EPS`) [S1, unofficial].
        *   Update `grz_t` and `min_sdf` for indices satisfying `idx` [S1, unofficial].
    *   **Advance Ray:**
        *   `t += d` [S1, unofficial].
        *   Update `pos_dir = (dir > 0)` [S1, unofficial].
        *   Update `no_itx = (d > ITX_EPS)` [S1, unofficial].
        *   Update `in_range = self.in_range(p) & (t < ray.maxt)` [S1, unofficial].

3.  **Grazing Point Refinement:**
    *   After the loop, call `self.get_grz(ray, grz_t - min_sdf, grz_t + min_sdf)` to refine the grazing point position [S1, unofficial].
    *   The `get_grz` method uses either secant or bisection iteration to find the root of the directional derivative within the interval [S1, unofficial].
    *   **Verification:**
        *   Verify the grazing point `grz` is valid if: `abs(self.at(grz)) < SDF_EPS`, `abs(self.dir_deriv(grz, ray.d)) < SDF_DERIV_EPS`, and `self.at(grz) > ITX_EPS` [S1, unofficial].

4.  **Intersection Refinement and Finalization:**
    *   Determine ray validity: `is_valid = (~no_itx) & in_range` [S1, unofficial].
    *   If `is_valid` is False, set `t = infinity` [S1, unofficial].
    *   If `is_valid` is True, refine the intersection time using `self.refine_intersect(ray, t - ITX_REFINE, t + ITX_REFINE)` [S1, unofficial].
    *   The `refine_intersect` method uses bisection iteration to find the root of the SDF within the interval [S1, unofficial].

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
        utils: Object containing ITX_REFINE, ITX_EPS, SDF_EPS, SDF_DERIV_EPS, NUM_GRZ, GRZ_MODE, RAY_EPS.
        
    Returns:
        t: Intersection time.
        is_valid: Boolean indicating valid intersection.
        grz: Grazing point position.
        grz_valid: Boolean indicating valid grazing point.
    """
    t = 0.0
    no_itx = True
    in_range_flag = True
    pos_dir = True
    grz_t = 0.0
    min_sdf = 1.0
    
    # Check bounding box intersection
    if is_inside_bounding_box(ray.o):
        t = 0.0
    else:
        t = mint + utils.RAY_EPS
        ray.maxt = min(ray.maxt, maxt_box)
        
    # Suspend gradients during loop
    with suspend_grad():
        while no_itx and in_range_flag:
            p = ray.o + ray.d * t
            d = sdf_eval(p)
            dir = dir_deriv_eval(p, ray.d)
            
            # Identify potential grazing points
            if (not pos_dir) and (dir > 0) and (d < min_sdf) and (d > utils.ITX_EPS):
                grz_t = t
                min_sdf = d
                
            # Advance ray
            t += d
            pos_dir = (dir > 0)
            no_itx = (d > utils.ITX_EPS)
            in_range_flag = check_in_range(p) and (t < ray.maxt)
            
    # Refine grazing point
    grz = get_grz(ray, grz_t - min_sdf, grz_t + min_sdf, sdf_eval, dir_deriv_eval, utils)
    
    # Verify grazing point
    grz_valid = (abs(sdf_eval(grz)) < utils.SDF_EPS) and \
                (abs(dir_deriv_eval(grz, ray.d)) < utils.SDF_DERIV_EPS) and \
                (sdf_eval(grz) > utils.ITX_EPS)
                
    # Determine ray validity
    is_valid = (not no_itx) and in_range_flag
    
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
            if abs(dR - dL) < 1e-12:
                break
            M = R - dR * (R - L) / (dR - dL)
            dM = dir_deriv_eval(M, d)
            if dM < 0:
                L = M
            if dM > 0:
                R = M
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
            if dM < 0:
                L = M
            if dM > 0:
                R = M
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
            if dM > 0:
                L = M
            if dM < 0:
                R = M
            cnt += 1
    return (L + R) / 2

def is_inside_bounding_box(p):
    # Placeholder for bounding box check implementation [G]
    return True

def check_in_range(p):
    # Placeholder for in-range check implementation [G]
    return True

def suspend_grad():
    # Placeholder for gradient suspension context manager [G]
    class DummyContext:
        def __enter__(self): return self
        def __exit__(self, *args): pass
    return DummyContext()
```

## Subtopics

### Screen-space aware intersection point selection
A method for selecting an appropriate intersection point when sphere tracing fails to converge (e.g., maximum iterations exceeded), particularly when a ray grazes an object's edge [S2, unofficial]. It selects the point along the ray with the smallest unbounding sphere radius, as measured in screen-space [S2, unofficial]. It terminates the ray early if the unbounding sphere radius is smaller than one half the size of a pixel [S2, unofficial]. Parameters include `pixelRadius` (radius of a pixel at t = 1) and `forceHit` (boolean enforcing the use of the candidate_t value as the result) [S2, unofficial]. It uses an over-relaxation sphere tracing step length calculation with a fallback to bisection-like refinement if the step length decreases significantly [S2, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Screen-space aware intersection point selection implementation [S2, unofficial]
omega = 1.2
t = t_min
candidate_error = float('inf')
candidate_t = t_min
previousRadius = 0.0
stepLength = 0.0
functionSign = -1 if f(o) < 0 else 1.0

for i in range(MAX_ITERATIONS):
    signedRadius = functionSign * f(d * t + o)
    radius = abs(signedRadius)
    sorFail = omega > 1 and (radius + previousRadius) < stepLength
    
    if sorFail:
        stepLength -= omega * stepLength
        omega = 1.0
    else:
        stepLength = signedRadius * omega
        previousRadius = radius
        error = radius / t
        
        if not sorFail and error < candidate_error:
            candidate_t = t
            candidate_error = error
            if not sorFail and (error < pixelRadius or t > t_max):
                break
                
        t += stepLength
        if (t > t_max or candidate_error > pixelRadius) and not forceHit:
            return float('inf')
            
return candidate_t
```

## Operating characteristics
*   **Time Complexity:** The main loop runs until intersection or max iterations [S1, unofficial]. The grazing point refinement adds a constant number of iterations (`NUM_GRZ`) per ray [S1, unofficial]. The intersection refinement adds `NUM_REFINE` iterations [S1, unofficial]. Overall, it is comparable to standard sphere tracing but with a small constant overhead for grazing detection and refinement [S1, unofficial].
*   **Memory Complexity:** O(1) additional memory per ray, storing a few scalar values and flags [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales linearly with the number of rays. The constant overhead of grazing detection may become more significant for scenes with many grazing rays [S1, unofficial].
*   **Strengths:** Explicitly handles grazing contacts, which are often missed or poorly resolved by standard sphere tracing [S1, unofficial]. Provides a fallback grazing point even if the primary intersection is not found [S1, unofficial].
*   **Limitations:** Requires evaluation of the directional derivative, which may be more expensive than the SDF itself [S1, unofficial]. The coarse sphere-traced grazing time `grz_t` is insufficiently accurate and requires root-finding refinement [S1, unofficial]. Implementation of the relaxation technique combined with this method can yield diminished rendering performance due to diverging threads [S2, unofficial].
*   **When to use:** When accurate handling of grazing contacts is important, such as in high-quality rendering or when dealing with complex geometry where rays frequently skim surfaces [S1, unofficial].
*   **When not to use:** When performance is critical and grazing contacts are rare or can be ignored [S1, unofficial]. When the directional derivative is not available or expensive to compute [S1, unofficial].

## Background
Originated from the implementation in `zichenwang01/relaxed-boundary` [S1, unofficial]. The method is designed to handle the challenges of sphere tracing near object boundaries, particularly grazing contacts [S1, unofficial]. The screen-space aware intersection point selection variant is described in "Enhanced Sphere Tracing" [S2, unofficial].

## Sources
[S1] zichenwang01/relaxed-boundary, https://github.com/zichenwang01/relaxed-boundary, fetched 2026-10-27 (unofficial)
[S2] Enhanced Sphere Tracing, https://kev.town/raymarching-toolkit/media/enhanced_sphere_tracing.pdf, fetched 2026-10-27 (unofficial)
