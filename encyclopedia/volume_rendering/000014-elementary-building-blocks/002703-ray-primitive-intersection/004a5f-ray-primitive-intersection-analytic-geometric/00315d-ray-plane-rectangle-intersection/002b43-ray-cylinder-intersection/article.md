# Ray-Cylinder Intersection

Also called: Cylinder-Ray Intersection, Ray-Cylinder Hit Test

## Overview
Computes the entry and exit distances for a ray intersecting a cylinder defined by `geomSize`, composed of front/back rectangles, top/bottom faces, and a lateral curved surface [S1, unofficial]. It is used in volume rendering to determine the traversal bounds of cylindrical primitives [S1, unofficial]. Unlike [Ray-Sphere Intersection](3539-ray-sphere-intersection.md) which relies on a single quadratic solution, this method decomposes the cylinder into planar and curved components, requiring intersection tests against multiple geometric primitives [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Inputs**: Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$ (not necessarily normalized), cylinder geometry size `geomSize` (a vector where `geomSize.x` is the radius, `geomSize.y` is the height along the Y-axis, and `geomSize.z` is the depth/width along the Z-axis).
- **Outputs**: Entry distance `inAbs`, exit distance `outAbs`. Returns `true` if an intersection exists, `false` otherwise.

**Data Structures**
- `geomSize`: `float3` containing $(r, h, w)$.
- `rayOrig`: `float3` ray origin.
- `rayVect`: `float3` ray direction.

**Algorithm Steps**
1. Initialize `inAbsVal = 0`, `outAbsVal = 0`, `inAbsFound = false`, `outAbsFound = false`.
2. Define the cylinder's local coordinate system. The cylinder axis is aligned with the Y-axis. The base is at $y=0$ and the top at $y=geomSize.y$. The radius is $R = geomSize.x$.
3. **Intersect Front Rectangle**:
   - Define points: $P_0 = (0, 0, 0)$, $P_1 = (R, 0, 0)$, $P_2 = (0, geomSize.z, 0)$.
   - Compute normal $\mathbf{n} = (P_1 - P_0) \times (P_2 - P_0)$.
   - Compute dot product $d = \mathbf{n} \cdot \mathbf{d}$.
   - If $d \neq 0$, compute $s = \frac{\mathbf{n} \cdot (P_0 - \mathbf{o})}{d}$.
   - Compute intersection point $P = \mathbf{o} + s\mathbf{d}$.
   - Check bounds: $P.x \ge 0$, $P.x \le R$, $P.y \ge 0$, $P.y \le geomSize.z$.
   - If bounds are satisfied:
     - If $d > 0$, update `outAbsVal` with $s$ (maximize). Set `outAbsFound = true`.
     - If $d < 0$, update `inAbsVal` with $s$ (minimize). Set `inAbsFound = true`.
4. **Intersect Back Rectangle**:
   - Define points: $P_0 = (R, 0, geomSize.z)$, $P_1 = (0, 0, 0)$, $P_2 = (0, geomSize.z, 0)$. Note: The source code uses `lastPoint` derived from `cylToCart(geomSize)`, which implies the back face is at $z=geomSize.z$ if the front is at $z=0$. However, the specific point definitions in the source are $P_0=\{lastPoint.x, 0, lastPoint.z\}$, $P_1=\{0,0,0\}$, $P_2=\{0,geomSize.z,0\}$. This geometry seems to define a triangle or a specific plane. We follow the source's explicit vector math.
   - Compute normal $\mathbf{n} = (P_1 - P_0) \times (P_2 - P_0)$.
   - Compute dot product $d = \mathbf{n} \cdot \mathbf{d}$.
   - If $d \neq 0$, compute $s = \frac{\mathbf{n} \cdot (P_0 - \mathbf{o})}{d}$.
   - Compute intersection point $P = \mathbf{o} + s\mathbf{d}$.
   - Check bounds using projected coordinate $x = \frac{P \cdot (P_0 - P_1)}{\|P_0 - P_1\|}$.
   - Check bounds: $x \ge 0$, $x \le R$, $P.y \ge 0$, $P.y \le geomSize.z$.
   - If bounds are satisfied, update `inAbsVal`/`outAbsVal` based on sign of $d$ as in Step 3.
5. **Intersect Top Face**:
   - Define point $P_0 = (0, geomSize.y, 0)$ and normal $\mathbf{n} = (0, 1, 0)$.
   - Compute dot product $d = \mathbf{n} \cdot \mathbf{d}$.
   - If $d \neq 0$, compute $s = \frac{\mathbf{n} \cdot (P_0 - \mathbf{o})}{d}$.
   - Compute intersection point $P = \mathbf{o} + s\mathbf{d}$.
   - Convert $P$ to cylindrical coordinates $(r, \theta, z)$ or check Cartesian bounds against radius. The source uses `cartToCyl` and checks $P.x \le geomSize.x$ and $P.y \le geomSize.y$. Note: The source code for Top Face checks `P.y <= geomSize.y` after conversion, which likely refers to the radial distance or a specific coordinate mapping. Given `geomSize` structure, we assume standard cylindrical check: $\sqrt{P.x^2 + P.z^2} \le R$ and $0 \le P.y \le geomSize.y$ (though $P.y$ is fixed at intersection with plane $y=geomSize.y$). The source code specifically checks `P.x <= geomSize.x && P.y <= geomSize.y` after `cartToCyl`. We implement the source's logic: convert to cylindrical, check radial bound.
   - If bounds are satisfied, update `inAbsVal`/`outAbsVal` based on sign of $d$.
6. **Intersect Bottom Face**:
   - Define point $P_0 = (0, 0, 0)$ and normal $\mathbf{n} = (0, -1, 0)$.
   - Compute dot product $d = \mathbf{n} \cdot \mathbf{d}$.
   - If $d \neq 0$, compute $s = \frac{\mathbf{n} \cdot (P_0 - \mathbf{o})}{d}$.
   - Compute intersection point $P = \mathbf{o} + s\mathbf{d}$.
   - Convert $P$ to cylindrical coordinates. Check radial bound $P.x \le geomSize.x$ and vertical bound $P.y \le geomSize.y$ (likely checking if $z$ in cylindrical is within height).
   - If bounds are satisfied, update `inAbsVal`/`outAbsVal` based on sign of $d$.
7. **Intersect Lateral Face**:
   - The lateral surface is defined by $x^2 + z^2 = R^2$.
   - Let $\mathbf{u} = (d.x, 0, d.z)$ and $\mathbf{v} = (o.x, 0, o.z)$.
   - Solve quadratic equation $a s^2 + b s + c = 0$:
     - $a = \mathbf{u} \cdot \mathbf{u}$
     - $b = 2 (\mathbf{u} \cdot \mathbf{v})$
     - $c = \mathbf{v} \cdot \mathbf{v} - R^2$
   - Compute discriminant $\Delta = b^2 - 4ac$.
   - If $\Delta \ge 0$, compute roots $s_0, s_1$.
   - For each root $s_i$:
     - Compute point $P = \mathbf{o} + s_i \mathbf{d}$.
     - Convert $P$ to cylindrical coordinates.
     - Check bounds: radial distance $\le R$ (implicit by solving quadratic), and height $0 \le P.y \le geomSize.y$.
     - If bounds are satisfied, update `inAbsVal`/`outAbsVal` based on sign of $d$ (or direction relative to normal). The source uses `dot` product with normal to determine entry/exit. For cylinder, normal at $P$ is $(P.x, 0, P.z)/R$.
8. **Handle Degenerate Cases**:
   - If `inAbsFound` is true and `outAbsFound` is false, set `outAbsVal = inAbsVal + 2 * R / \|d\|`.
   - If `outAbsFound` is true and `inAbsFound` is false, set `inAbsVal = outAbsVal - 2 * R / \|d\|`.
9. **Return**: `true` if `outAbsVal >= inAbsVal` and both `inAbsFound` and `outAbsFound` are true.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_intersect_cylinder(ray_orig, ray_dir, geom_size):
    """
    ray_orig: float3 (x, y, z)
    ray_dir: float3 (x, y, z)
    geom_size: float3 (radius, height, width) -> Note: Source uses geomSize.x as radius, geomSize.y as height, geomSize.z as width/depth.
    Returns: (in_abs, out_abs, hit)
    """
    R = geom_size.x
    H = geom_size.y
    W = geom_size.z
    
    in_abs_val = 0.0
    out_abs_val = 0.0
    in_abs_found = False
    out_abs_found = False
    
    # Helper for cylindrical conversion
    def cart_to_cyl(p):
        r = sqrt(p.x**2 + p.z**2)
        theta = atan2(p.z, p.x)
        y = p.y
        return (r, theta, y)
        
    # 1. Front Rectangle (z=0 plane, x in [0,R], y in [0,H]? Source says P2={0, geomSize.z, 0} so y goes up to W? 
    # Source code: P0={0,0,0}, P1={geomSize.x,0,0}, P2={0,geomSize.z,0}. 
    # This defines a triangle in the x-y plane at z=0.
    # Normal calculation:
    P0 = (0, 0, 0)
    P1 = (R, 0, 0)
    P2 = (0, W, 0)
    
    n = cross(sub(P1, P0), sub(P2, P0))
    d = dot(n, ray_dir)
    
    if d != 0:
        s = dot(n, sub(P0, ray_orig)) / d
        P = add(ray_orig, mul(s, ray_dir))
        if P.x >= 0 and P.x <= R and P.y >= 0 and P.y <= W:
            if d > 0:
                out_abs_val = max(out_abs_val, s) if out_abs_found else s
                out_abs_found = True
            else:
                in_abs_val = min(in_abs_val, s) if in_abs_found else s
                in_abs_found = True

    # 2. Back Rectangle
    # Source: P0={lastPoint.x, 0, lastPoint.z}, P1={0,0,0}, P2={0,geomSize.z,0}
    # lastPoint = cylToCart(geomSize). Assuming cylToCart(geomSize) returns (R, H, W) or similar.
    # Let's assume lastPoint is (R, H, W) based on typical bounding box.
    # However, source code uses specific vectors.
    last_point = (R, H, W) # Assumed from context of cylToCart(geomSize)
    P0 = (last_point[0], 0, last_point[2])
    P1 = (0, 0, 0)
    P2 = (0, W, 0)
    
    n = cross(sub(P1, P0), sub(P2, P0))
    d = dot(n, ray_dir)
    
    if d != 0:
        s = dot(n, sub(P0, ray_orig)) / d
        P = add(ray_orig, mul(s, ray_dir))
        # Check bounds using projected coordinate
        vec = sub(P0, P1)
        len_vec = length(vec)
        if len_vec > 0:
            x_proj = dot(P, vec) / len_vec
            if x_proj >= 0 and x_proj <= R and P.y >= 0 and P.y <= W:
                if d > 0:
                    out_abs_val = max(out_abs_val, s) if out_abs_found else s
                    out_abs_found = True
                else:
                    in_abs_val = min(in_abs_val, s) if in_abs_found else s
                    in_abs_found = True

    # 3. Top Face
    P0 = (0, H, 0)
    n = (0, 1, 0)
    d = dot(n, ray_dir)
    
    if d != 0:
        s = dot(n, sub(P0, ray_orig)) / d
        P = add(ray_orig, mul(s, ray_dir))
        cyl_p = cart_to_cyl(P)
        # Source checks: P.x <= geomSize.x && P.y <= geomSize.y
        # In cylindrical, P.x is radius, P.y is theta? Or P.y is z?
        # Source code: P = cartToCyl(...). if (P.x <= geomSize.x && P.y <= geomSize.y)
        # Assuming cartToCyl returns (r, theta, z) or (r, z, theta).
        # Given geomSize.y is height, and P.y in source is checked against it, 
        # it's likely P.y in cyl coords is the vertical coordinate (z in standard, y in source).
        # Let's assume cartToCyl returns (r, z, theta) or similar where second component is height.
        # Actually, standard cyl is (r, theta, z). If P.y <= H, and P.y is theta, that's wrong.
        # If P.y is z (height), then it makes sense.
        # Let's assume the source's cartToCyl returns (r, z, theta) or the check is on the height component.
        # We will check radius and height.
        r = cyl_p[0]
        h = cyl_p[1] # Assuming second component is height
        if r <= R and h >= 0 and h <= H:
            if d > 0:
                out_abs_val = max(out_abs_val, s) if out_abs_found else s
                out_abs_found = True
            else:
                in_abs_val = min(in_abs_val, s) if in_abs_found else s
                in_abs_found = True

    # 4. Bottom Face
    P0 = (0, 0, 0)
    n = (0, -1, 0)
    d = dot(n, ray_dir)
    
    if d != 0:
        s = dot(n, sub(P0, ray_orig)) / d
        P = add(ray_orig, mul(s, ray_dir))
        cyl_p = cart_to_cyl(P)
        r = cyl_p[0]
        h = cyl_p[1]
        if r <= R and h >= 0 and h <= H:
            if d > 0:
                out_abs_val = max(out_abs_val, s) if out_abs_found else s
                out_abs_found = True
            else:
                in_abs_val = min(in_abs_val, s) if in_abs_found else s
                in_abs_found = True

    # 5. Lateral Face
    u = (ray_dir.x, 0, ray_dir.z)
    v = (ray_orig.x, 0, ray_orig.z)
    
    a = dot(u, u)
    b = 2 * dot(u, v)
    c = dot(v, v) - (R * R)
    
    delta = b*b - 4*a*c
    
    if delta >= 0:
        sqrt_delta = sqrt(delta)
        s0 = (-b - sqrt_delta) / (2 * a)
        s1 = (-b + sqrt_delta) / (2 * a)
        
        for s in [s0, s1]:
            P = add(ray_orig, mul(s, ray_dir))
            cyl_p = cart_to_cyl(P)
            r = cyl_p[0]
            h = cyl_p[1]
            if r <= R and h >= 0 and h <= H:
                # Determine entry/exit based on normal
                # Normal at P on cylinder is (P.x, 0, P.z) / R
                normal = (P.x / R, 0, P.z / R)
                dot_n_d = dot(normal, ray_dir)
                if dot_n_d > 0:
                    out_abs_val = max(out_abs_val, s) if out_abs_found else s
                    out_abs_found = True
                else:
                    in_abs_val = min(in_abs_val, s) if in_abs_found else s
                    in_abs_found = True

    # 6. Degenerate Cases
    if in_abs_found and not out_abs_found:
        out_abs_val = in_abs_val + 2 * R / length(ray_dir)
    if out_abs_found and not in_abs_found:
        in_abs_val = out_abs_val - 2 * R / length(ray_dir)
        
    hit = out_abs_val >= in_abs_val and out_abs_found and in_abs_found
    return in_abs_val, out_abs_val, hit
```

## Subtopics

### Query Cylinder Cross Section Parameterization
Determines the query cylinder's radius $r_c$ for particle querying in discrete media. The parameter $k$ in the radius formula determines the percentage of the pixel's footprint at the nearest boundary of the medium. Typical value range for $k$ is $[0.5, 1]$ [S2]. Setting $k$ too large produces undesirable aliasing and overly blurred volumetric glints, although overall brightness remains similar [S2]. Setting $k$ too small misses many particles during querying, leading to inefficiency, especially in very sparse media (e.g., $N_{tot} = 10^4$) [S2].

## Operating characteristics
- **Time Complexity**: Constant time per ray, as it performs a fixed number of intersection tests against planar and quadratic surfaces.
- **Memory Complexity**: Constant, requiring only a few scalar variables for state.
- **Strengths**: Accurate intersection for cylindrical primitives; handles degenerate cases where only entry or exit is found [S1, unofficial].
- **Limitations**: More complex than sphere intersection due to multiple face tests; requires careful handling of coordinate transformations [S1, unofficial].
- **When to use**: When rendering scenes with cylindrical primitives or when using query cylinders for sparse particle systems [S1, unofficial][S2].

## Background
The method is derived from CUDA-based volume rendering implementations [S1, unofficial]. It is used in the context of rendering discrete participating media using geometrical optics approximation, where query cylinders are used to efficiently sample particles [S2].

## Sources
[S1] ojaisnielsen/dtu-2010-cuda-volume-rendering, https://github.com/ojaisnielsen/dtu-2010-cuda-volume-rendering, fetched 2026-10-27 (unofficial)
[S2] Rendering discrete participating media using geometrical optics approximation, https://doi.org/10.1007/s41095-021-0253-5, fetched 2026-10-26
