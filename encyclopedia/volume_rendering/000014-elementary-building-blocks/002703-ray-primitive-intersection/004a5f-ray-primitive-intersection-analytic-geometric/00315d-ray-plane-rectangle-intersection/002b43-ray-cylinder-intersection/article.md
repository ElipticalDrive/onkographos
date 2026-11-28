# Ray-Cylinder Intersection


## Overview
Computes the entry and exit distances for a ray intersecting a finite cylinder aligned along the Y-axis, defined by radius, top Y, and bottom Y [S1, unofficial]. It decomposes the cylinder into front/back rectangles, top/bottom faces, and a lateral curved surface to test intersections [S2, unofficial]. Unlike [Ray-Sphere Intersection](3539-ray-sphere-intersection.md) which relies on a single quadratic solution, this method requires intersection tests against multiple geometric primitives [S2, unofficial].

## How it works
**Inputs and Outputs**
- **Inputs**: Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$ (not necessarily normalized), cylinder geometry size `geomSize` (a vector where `geomSize.x` is the radius, `geomSize.y` is the height along the Y-axis, and `geomSize.z` is the depth/width along the Z-axis) [S2, unofficial].
- **Outputs**: Entry distance `inAbs`, exit distance `outAbs`. Returns `true` if an intersection exists, `false` otherwise [S2, unofficial].

**Data Structures**
- `geomSize`: `float3` containing $(r, h, w)$.
- `rayOrig`: `float3` ray origin.
- `rayVect`: `float3` ray direction.

**Algorithm Steps**
1. Initialize `inAbsVal = 0`, `outAbsVal = 0`, `inAbsFound = false`, `outAbsFound = false` [S2, unofficial].
2. **Intersect Front Rectangle**: Define points $P_0 = (0, 0, 0)$, $P_1 = (geomSize.x, 0, 0)$, $P_2 = (0, geomSize.z, 0)$ [S2, unofficial]. Compute normal $\mathbf{n} = (P_1 - P_0) \times (P_2 - P_0)$ [S2, unofficial]. Compute dot product $d = \mathbf{n} \cdot \mathbf{d}$ [S2, unofficial]. If $d \neq 0$, compute $s = \mathbf{n} \cdot (P_0 - \mathbf{o}) / d$ [S2, unofficial]. Compute intersection point $P = \mathbf{o} + s\mathbf{d}$ [S2, unofficial]. Check bounds $P.x \ge 0$, $P.x \le geomSize.x$, $P.y \ge 0$, $P.y \le geomSize.z$ [S2, unofficial]. If $d > 0$, update `outAbsVal` with $s$ (maximize); if $d < 0$, update `inAbsVal` with $s$ (minimize) [S2, unofficial].
3. **Intersect Back Rectangle**: Define points $P_0 = (lastPoint.x, 0, lastPoint.z)$, $P_1 = (0, 0, 0)$, $P_2 = (0, geomSize.z, 0)$, where `lastPoint` is derived from `cylToCart(geomSize)` [S2, unofficial]. Compute normal $\mathbf{n}$, dot $d$, and $s$ [S2, unofficial]. Check bounds using projected coordinate $x = \mathbf{P} \cdot (P_0 - P_1) / \|P_0 - P_1\|$ [S2, unofficial]. Check bounds $x \ge 0$, $x \le geomSize.x$, $P.y \ge 0$, $P.y \le geomSize.z$ [S2, unofficial]. Update `inAbsVal`/`outAbsVal` based on sign of $d$ [S2, unofficial].
4. **Intersect Top Face**: Define point $P_0 = (0, geomSize.z, 0)$ and normal $\mathbf{n} = (0, 1, 0)$ [S2, unofficial]. Compute $s$ [S2, unofficial]. Convert intersection point to cylindrical coordinates $P = \text{cartToCyl}(\mathbf{o} + s\mathbf{d})$ [S2, unofficial]. Check if $P.x \le geomSize.x$ and $P.y \le geomSize.y$ [S2, unofficial]. Update `inAbsVal`/`outAbsVal` based on sign of $d$ [S2, unofficial].
5. **Intersect Bottom Face**: Define point $P_0 = (0, 0, 0)$ and normal $\mathbf{n} = (0, -1, 0)$ [S2, unofficial]. Compute $s$ [S2, unofficial]. Convert to cylindrical coordinates [S2, unofficial]. Check bounds [S2, unofficial]. Update `inAbsVal`/`outAbsVal` based on sign of $d$ [S2, unofficial].
6. **Intersect Lateral Face**: Solve quadratic equation for cylinder radius `geomSize.x` [S2, unofficial]. Compute $a = \mathbf{u} \cdot \mathbf{u}$, $b = 2 \mathbf{u} \cdot \mathbf{v}$, $c = \mathbf{v} \cdot \mathbf{v} - (\text{geomSize.x})^2$, where $\mathbf{u} = (\mathbf{d}.x, \mathbf{d}.z)$ and $\mathbf{v} = (\mathbf{o}.x, \mathbf{o}.z)$ [S2, unofficial]. If discriminant $\Delta \ge 0$, compute roots $s_0, s_1$ [S2, unofficial]. Convert points to cylindrical $P_{\text{Cyl}}$ [S2, unofficial]. Check if $P_{\text{Cyl}}.y \le \text{geomSize.y}$ and $P_{\text{Cyl}}.z \ge 0$ and $P_{\text{Cyl}}.z \le \text{geomSize.z}$ [S2, unofficial]. Update `inAbsVal`/`outAbsVal` based on dot product $d[i]$ [S2, unofficial].
7. **Handle Degenerate Cases**: If only entry found, set `outAbsVal = inAbsVal + 2 * geomSize.x / \| \mathbf{d} \|` [S2, unofficial]. If only exit found, set `inAbsVal = outAbsVal - 2 * geomSize.x / \| \mathbf{d} \|` [S2, unofficial].
8. **Return**: `true` if `outAbsVal >= inAbsVal` and both `outAbsFound` and `inAbsFound` are true [S2, unofficial].

**Formulas**
Quadratic coefficients for the lateral face: $a = d_x^2 + d_z^2$, $b = 2(d_x o_x + d_z o_z)$, $c = o_x^2 + o_z^2 - r^2$. Discriminant $\Delta = b^2 - 4ac$ [S1, unofficial].

**Parameters**
`geomType`: Character indicating geometry type; if `geomType == 'c'`, use this method [S2, unofficial].

**Pitfalls**
If $y <$ bottom or $y >$ top, the intersection is outside the finite cylinder bounds; return $-1.0$ [S1, unofficial]. Precision issues may occur near boundaries; degenerate cases where only entry or exit is found require fallback calculations [S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_intersect_cylinder(ray_orig, ray_dir, geom_size):
    """
    ray_orig: tuple (x, y, z)
    ray_dir: tuple (x, y, z)
    geom_size: tuple (radius, height, width)
    Returns: (in_abs, out_abs, hit)
    """
    import math
    
    R = geom_size[0]
    H = geom_size[1]
    W = geom_size[2]
    
    in_abs_val = 0.0
    out_abs_val = 0.0
    in_abs_found = False
    out_abs_found = False
    
    def cart_to_cyl(p):
        r = (p[0]**2 + p[2]**2)**0.5
        theta = math.atan2(p[2], p[0])
        y = p[1]
        return (r, theta, y)
        
    # 1. Front Rectangle
    P0 = (0, 0, 0)
    P1 = (R, 0, 0)
    P2 = (0, W, 0)
    n = (P1[1]*P2[2] - P1[2]*P2[1], P1[2]*P2[0] - P1[0]*P2[2], P1[0]*P2[1] - P1[1]*P2[0])
    d = n[0]*ray_dir[0] + n[1]*ray_dir[1] + n[2]*ray_dir[2]
    
    if d != 0:
        s = (n[0]*(P0[0]-ray_orig[0]) + n[1]*(P0[1]-ray_orig[1]) + n[2]*(P0[2]-ray_orig[2])) / d
        P = (ray_orig[0] + s*ray_dir[0], ray_orig[1] + s*ray_dir[1], ray_orig[2] + s*ray_dir[2])
        if P[0] >= 0 and P[0] <= R and P[1] >= 0 and P[1] <= W:
            if d > 0:
                out_abs_val = max(out_abs_val, s) if out_abs_found else s
                out_abs_found = True
            else:
                in_abs_val = min(in_abs_val, s) if in_abs_found else s
                in_abs_found = True

    # 2. Back Rectangle
    last_point = (R, H, W) # Derived from cylToCart(geomSize)
    P0 = (last_point[0], 0, last_point[2])
    P1 = (0, 0, 0)
    P2 = (0, W, 0)
    n = (P1[1]*P2[2] - P1[2]*P2[1], P1[2]*P2[0] - P1[0]*P2[2], P1[0]*P2[1] - P1[1]*P2[0])
    d = n[0]*ray_dir[0] + n[1]*ray_dir[1] + n[2]*ray_dir[2]
    
    if d != 0:
        s = (n[0]*(P0[0]-ray_orig[0]) + n[1]*(P0[1]-ray_orig[1]) + n[2]*(P0[2]-ray_orig[2])) / d
        P = (ray_orig[0] + s*ray_dir[0], ray_orig[1] + s*ray_dir[1], ray_orig[2] + s*ray_dir[2])
        vec = (P0[0]-P1[0], P0[1]-P1[1], P0[2]-P1[2])
        len_vec = (vec[0]**2 + vec[1]**2 + vec[2]**2)**0.5
        if len_vec > 0:
            x_proj = (P[0]*vec[0] + P[1]*vec[1] + P[2]*vec[2]) / len_vec
            if x_proj >= 0 and x_proj <= R and P[1] >= 0 and P[1] <= W:
                if d > 0:
                    out_abs_val = max(out_abs_val, s) if out_abs_found else s
                    out_abs_found = True
                else:
                    in_abs_val = min(in_abs_val, s) if in_abs_found else s
                    in_abs_found = True

    # 3. Top Face
    P0 = (0, H, 0)
    n = (0, 1, 0)
    d = n[0]*ray_dir[0] + n[1]*ray_dir[1] + n[2]*ray_dir[2]
    
    if d != 0:
        s = (n[0]*(P0[0]-ray_orig[0]) + n[1]*(P0[1]-ray_orig[1]) + n[2]*(P0[2]-ray_orig[2])) / d
        P = (ray_orig[0] + s*ray_dir[0], ray_orig[1] + s*ray_dir[1], ray_orig[2] + s*ray_dir[2])
        cyl_p = cart_to_cyl(P)
        if cyl_p[0] <= R and cyl_p[2] >= 0 and cyl_p[2] <= H:
            if d > 0:
                out_abs_val = max(out_abs_val, s) if out_abs_found else s
                out_abs_found = True
            else:
                in_abs_val = min(in_abs_val, s) if in_abs_found else s
                in_abs_found = True

    # 4. Bottom Face
    P0 = (0, 0, 0)
    n = (0, -1, 0)
    d = n[0]*ray_dir[0] + n[1]*ray_dir[1] + n[2]*ray_dir[2]
    
    if d != 0:
        s = (n[0]*(P0[0]-ray_orig[0]) + n[1]*(P0[1]-ray_orig[1]) + n[2]*(P0[2]-ray_orig[2])) / d
        P = (ray_orig[0] + s*ray_dir[0], ray_orig[1] + s*ray_dir[1], ray_orig[2] + s*ray_dir[2])
        cyl_p = cart_to_cyl(P)
        if cyl_p[0] <= R and cyl_p[2] >= 0 and cyl_p[2] <= H:
            if d > 0:
                out_abs_val = max(out_abs_val, s) if out_abs_found else s
                out_abs_found = True
            else:
                in_abs_val = min(in_abs_val, s) if in_abs_found else s
                in_abs_found = True

    # 5. Lateral Face
    u = (ray_dir[0], ray_dir[2])
    v = (ray_orig[0], ray_orig[2])
    
    a = u[0]**2 + u[1]**2
    b = 2 * (u[0]*v[0] + u[1]*v[1])
    c = v[0]**2 + v[1]**2 - (R * R)
    
    delta = b*b - 4*a*c
    
    if delta >= 0:
        sqrt_delta = delta**0.5
        s0 = (-b - sqrt_delta) / (2 * a)
        s1 = (-b + sqrt_delta) / (2 * a)
        
        for s in [s0, s1]:
            P = (ray_orig[0] + s*ray_dir[0], ray_orig[1] + s*ray_dir[1], ray_orig[2] + s*ray_dir[2])
            cyl_p = cart_to_cyl(P)
            if cyl_p[0] <= R and cyl_p[2] >= 0 and cyl_p[2] <= H:
                normal = (P[0] / R, 0, P[2] / R)
                dot_n_d = normal[0]*ray_dir[0] + normal[1]*ray_dir[1] + normal[2]*ray_dir[2]
                if dot_n_d > 0:
                    out_abs_val = max(out_abs_val, s) if out_abs_found else s
                    out_abs_found = True
                else:
                    in_abs_val = min(in_abs_val, s) if in_abs_found else s
                    in_abs_found = True

    # 6. Degenerate Cases
    if in_abs_found and not out_abs_found:
        out_abs_val = in_abs_val + 2 * R / (ray_dir[0]**2 + ray_dir[1]**2 + ray_dir[2]**2)**0.5
    if out_abs_found and not in_abs_found:
        in_abs_val = out_abs_val - 2 * R / (ray_dir[0]**2 + ray_dir[1]**2 + ray_dir[2]**2)**0.5
        
    hit = out_abs_val >= in_abs_val and out_abs_found and in_abs_found
    return in_abs_val, out_abs_val, hit
```

## Subtopics

### Query Cylinder Cross Section Parameterization
Determines the query cylinder's radius $r_c$ for particle querying in discrete media. The parameter $k$ in the radius formula determines the percentage of the pixel's footprint at the nearest boundary of the medium. Typical value range for $k$ is $[0.5, 1]$ [S2, unofficial]. Setting $k$ too large produces undesirable aliasing and overly blurred volumetric glints, although overall brightness remains similar [S2, unofficial]. Setting $k$ too small misses many particles during querying, leading to inefficiency, especially in very sparse media (e.g., $N_{tot} = 10^4$) [S2, unofficial].

## Operating characteristics
- **Time Complexity**: Constant time per ray, as it performs a fixed number of intersection tests against planar and quadratic surfaces.
- **Memory Complexity**: Constant, requiring only a few scalar variables for state.
- **Strengths**: Accurate intersection for cylindrical primitives; handles degenerate cases where only entry or exit is found [S1, unofficial].
- **Limitations**: More complex than sphere intersection due to multiple face tests; requires careful handling of coordinate transformations [S1, unofficial].
- **When to use**: When rendering scenes with cylindrical primitives or when using query cylinders for sparse particle systems [S1, unofficial][S2, unofficial].

## Background
The method is derived from CUDA-based volume rendering implementations [S1, unofficial]. It is used in the context of rendering discrete participating media using geometrical optics approximation, where query cylinders are used to efficiently sample particles [S2, unofficial].

## Sources
[S1] GameDevTecnico/cubos, https://github.com/GameDevTecnico/cubos, fetched 2026-10-27 (unofficial)
[S2] ojaisnielsen/dtu-2010-cuda-volume-rendering, https://github.com/ojaisnielsen/dtu-2010-cuda-volume-rendering, fetched 2026-10-27 (unofficial)
[S3] Rendering discrete participating media using geometrical optics approximation, https://doi.org/10.1007/s41095-021-0253-5, fetched 2026-10-26
