# Ray-Cylinder Intersection

## Overview
Computes the entry and exit distances for a ray intersecting a cylinder defined by `geomSize`, composed of front/back rectangles, top/bottom faces, and a lateral curved surface [S1, unofficial]. It differs from [Ray-box intersection](776-ray-box-intersection.md) by handling the curved lateral surface via quadratic solving and cylindrical coordinate checks [S1, unofficial].

## How it works
**Inputs and Outputs**
- `rayOrig`: Ray origin $\mathbf{o}$ (float3).
- `rayVect`: Ray direction $\mathbf{d}$ (float3). The source implementation uses the unnormalized direction vector for distance calculations, where the parameter $s$ represents the magnitude along $\mathbf{d}$ [S1, unofficial].
- `geomSize`: Cylinder dimensions (float3). $x$ is the radius $R$, $y$ is the height $H$, $z$ is the width of the rectangular cross-section faces [S1, unofficial].
- `inAbs`: Output entry distance $s_{in}$.
- `outAbs`: Output exit distance $s_{out}$.

**Data Structures**
The geometry is defined by a bounding box-like structure `geomSize` where:
- `geomSize.x`: Radius $R$ of the cylinder.
- `geomSize.y`: Height $H$ of the cylinder (extent along the Y-axis).
- `geomSize.z`: Width $W$ of the front/back rectangular faces [S1, unofficial].

**Coordinate Systems**
The method uses Cartesian coordinates for ray intersection tests and converts to cylindrical coordinates for boundary checks on the top/bottom faces and lateral surface [S1, unofficial].
- Cartesian: $(x, y, z)$
- Cylindrical: $(r, h, z)$ where $r$ is radius, $h$ is height, and $z$ is the width component [G]. The source code `cartToCyl` and `cylToCart` imply a specific mapping where the Cartesian $z$ coordinate is preserved as the third cylindrical component for slab checks [S1, unofficial].

**Numbered Steps**

1. **Initialization**: Set `inAbsVal = 0`, `outAbsVal = 0`. Set `inAbsFound = false`, `outAbsFound = false` [S1, unofficial].
2. **Intersect Front Rectangle**: Define $P_0=(0,0,0)$, $P_1=(\text{geomSize}.x, 0, 0)$, $P_2=(0, \text{geomSize}.z, 0)$. Compute normal $\mathbf{n} = (P_1 - P_0) \times (P_2 - P_0)$. Compute dot $d = \mathbf{n} \cdot \mathbf{d}$. If $d \neq 0$, compute $s = \mathbf{n} \cdot (P_0 - \mathbf{o}) / d$. Compute intersection point $P = \mathbf{o} + s\mathbf{d}$. Check if $P.x \ge 0$ and $P.x \le \text{geomSize}.x$ and $P.y \ge 0$ and $P.y \le \text{geomSize}.z$. Update `inAbsVal` or `outAbsVal` based on the sign of $d$ ($d > 0 \rightarrow$ exit, $d < 0 \rightarrow$ entry) [S1, unofficial].
3. **Intersect Back Rectangle**: Define $P_0=(\text{lastPoint}.x, 0, \text{lastPoint}.z)$, $P_1=(0,0,0)$, $P_2=(0, \text{geomSize}.z, 0)$, where `lastPoint = cylToCart(geomSize)`. Compute normal $\mathbf{n} = (P_1 - P_0) \times (P_2 - P_0)$. Compute dot $d = \mathbf{n} \cdot \mathbf{d}$. If $d \neq 0$, compute $s = \mathbf{n} \cdot (P_0 - \mathbf{o}) / d$. Compute $P = \mathbf{o} + s\mathbf{d}$. Check bounds using projected coordinate $x = \mathbf{P} \cdot (P_0 - P_1) / \|P_0 - P_1\|$. If $0 \le x \le \text{geomSize}.x$ and $0 \le P.y \le \text{geomSize}.z$, update `inAbsVal`/`outAbsVal` [S1, unofficial].
4. **Intersect Top Face**: Define $P_0=(0, \text{geomSize}.z, 0)$, $\mathbf{n}=(0,1,0)$. Compute $d = \mathbf{n} \cdot \mathbf{d}$. If $d \neq 0$, compute $s = \mathbf{n} \cdot (P_0 - \mathbf{o}) / d$. Compute $P = \mathbf{o} + s\mathbf{d}$. Convert intersection point to cylindrical coordinates $P_{\text{cyl}} = \text{cartToCyl}(P)$. Check if $P_{\text{cyl}}.x \le \text{geomSize}.x$ and $P_{\text{cyl}}.y \le \text{geomSize}.y$. Update `inAbsVal`/`outAbsVal` [S1, unofficial].
5. **Intersect Bottom Face**: Define $P_0=(0,0,0)$, $\mathbf{n}=(0,-1,0)$. Compute $d = \mathbf{n} \cdot \mathbf{d}$. If $d \neq 0$, compute $s = \mathbf{n} \cdot (P_0 - \mathbf{o}) / d$. Compute $P = \mathbf{o} + s\mathbf{d}$. Convert to cylindrical. Check bounds. Update `inAbsVal`/`outAbsVal` [S1, unofficial].
6. **Intersect Lateral Face**: Solve quadratic equation for cylinder radius $\text{geomSize}.x$. Compute $a = \mathbf{u} \cdot \mathbf{u}$, $b = 2 \mathbf{u} \cdot \mathbf{v}$, $c = \mathbf{v} \cdot \mathbf{v} - (\text{geomSize}.x)^2$ where $\mathbf{u}=(\mathbf{d}.x, \mathbf{d}.z)$ and $\mathbf{v}=(\mathbf{o}.x, \mathbf{o}.z)$. If discriminant $\Delta \ge 0$, compute roots $s_0, s_1$. For each root, compute $P = \mathbf{o} + s_i\mathbf{d}$. Convert to cylindrical $P_{\text{cyl}}$. Check if $P_{\text{cyl}}.y \le \text{geomSize}.y$ and $0 \le P_{\text{cyl}}.z \le \text{geomSize}.z$. Update `inAbsVal`/`outAbsVal` based on the dot product of the surface normal and $\mathbf{d}$ [S1, unofficial].
7. **Handle Degenerate Cases**: If only entry found, set `outAbsVal = inAbsVal + 2 * geomSize.x / length(rayVect)`. If only exit found, set `inAbsVal = outAbsVal - 2 * geomSize.x / length(rayVect)` [S1, unofficial].
8. **Return**: Return `true` if `outAbsVal >= inAbsVal` and both `outAbsFound` and `inAbsFound` are true [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_intersect_cyl(ray_orig, ray_vect, geom_size):
    in_abs_val = 0.0
    out_abs_val = 0.0
    in_abs_found = False
    out_abs_found = False

    def update(s, is_exit):
        nonlocal in_abs_val, out_abs_val, in_abs_found, out_abs_found
        if is_exit:
            out_abs_val = s if not out_abs_found else max(out_abs_val, s)
            out_abs_found = True
        else:
            in_abs_val = s if not in_abs_found else min(in_abs_val, s)
            in_abs_found = True

    def cart_to_cyl(p):
        return (sqrt(p[0]**2 + p[2]**2), p[1], p[2])

    # Front Rectangle
    P0 = (0.0, 0.0, 0.0)
    P1 = (geom_size.x, 0.0, 0.0)
    P2 = (0.0, geom_size.z, 0.0)
    n = cross(P1 - P0, P2 - P0)
    d = dot(n, ray_vect)
    if d != 0:
        s = dot(n, (P0[0]-ray_orig[0], P0[1]-ray_orig[1], P0[2]-ray_orig[2])) / d
        P = (ray_orig[0] + s*ray_vect[0], ray_orig[1] + s*ray_vect[1], ray_orig[2] + s*ray_vect[2])
        if 0 <= P[0] <= geom_size.x and 0 <= P[1] <= geom_size.z:
            update(s, d > 0)

    # Back Rectangle
    last_point = (geom_size.x, geom_size.y, geom_size.z)
    P0 = (last_point[0], 0.0, last_point[2])
    P1 = (0.0, 0.0, 0.0)
    P2 = (0.0, geom_size.z, 0.0)
    n = cross(P1 - P0, P2 - P0)
    d = dot(n, ray_vect)
    if d != 0:
        s = dot(n, (P0[0]-ray_orig[0], P0[1]-ray_orig[1], P0[2]-ray_orig[2])) / d
        P = (ray_orig[0] + s*ray_vect[0], ray_orig[1] + s*ray_vect[1], ray_orig[2] + s*ray_vect[2])
        x_proj = dot(P, (P0[0]-P1[0], P0[1]-P1[1], P0[2]-P1[2])) / length(P0 - P1)
        if 0 <= x_proj <= geom_size.x and 0 <= P[1] <= geom_size.z:
            update(s, d > 0)

    # Top Face
    P0 = (0.0, geom_size.z, 0.0)
    n = (0.0, 1.0, 0.0)
    d = dot(n, ray_vect)
    if d != 0:
        s = dot(n, (P0[0]-ray_orig[0], P0[1]-ray_orig[1], P0[2]-ray_orig[2])) / d
        P = (ray_orig[0] + s*ray_vect[0], ray_orig[1] + s*ray_vect[1], ray_orig[2] + s*ray_vect[2])
        P_cyl = cart_to_cyl(P)
        if P_cyl[0] <= geom_size.x and P_cyl[1] <= geom_size.y:
            update(s, d > 0)

    # Bottom Face
    P0 = (0.0, 0.0, 0.0)
    n = (0.0, -1.0, 0.0)
    d = dot(n, ray_vect)
    if d != 0:
        s = dot(n, (P0[0]-ray_orig[0], P0[1]-ray_orig[1], P0[2]-ray_orig[2])) / d
        P = (ray_orig[0] + s*ray_vect[0], ray_orig[1] + s*ray_vect[1], ray_orig[2] + s*ray_vect[2])
        P_cyl = cart_to_cyl(P)
        if P_cyl[0] <= geom_size.x and P_cyl[1] <= geom_size.y:
            update(s, d > 0)

    # Lateral Face
    u = (ray_vect.x, ray_vect.z)
    v = (ray_orig.x, ray_orig.z)
    a = dot(u, u)
    b = 2 * dot(u, v)
    c = dot(v, v) - (geom_size.x * geom_size.x)
    delta = b*b - 4*a*c
    if delta >= 0:
        sqrt_delta = sqrt(delta)
        s0 = (-b - sqrt_delta) / (2*a)
        s1 = (-b + sqrt_delta) / (2*a)
        for s in (s0, s1):
            P = (ray_orig[0] + s*ray_vect[0], ray_orig[1] + s*ray_vect[1], ray_orig[2] + s*ray_vect[2])
            P_cyl = cart_to_cyl(P)
            if P_cyl[1] <= geom_size.y and 0 <= P_cyl[2] <= geom_size.z:
                normal = (P.x, 0.0, P.z)
                if dot(normal, ray_vect) > 0:
                    update(s, True)
                else:
                    update(s, False)

    # Degenerate cases
    if in_abs_found and not out_abs_found:
        out_abs_val = in_abs_val + 2 * geom_size.x / length(ray_vect)
    if out_abs_found and not in_abs_found:
        in_abs_val = out_abs_val - 2 * geom_size.x / length(ray_vect)

    return out_abs_val >= in_abs_val and out_abs_found and in_abs_found, in_abs_val, out_abs_val
```

**Parameters**
- `geomType`: Character indicating geometry type. If `geomType == 'c'`, this method is used [S1, unofficial].

**Edge Cases and Pitfalls**
- **Precision**: Division by zero occurs if the ray is parallel to a face ($d=0$) [S1, unofficial].
- **Degenerate Input**: If the ray passes exactly through an edge or vertex, multiple faces may report the same $s$. The min/max logic handles this, but floating point errors may cause `inAbsVal > outAbsVal` [S1, unofficial].
- **Cylindrical Conversion**: The source uses `cartToCyl` and `cylToCart` which are not defined in the snippet. The implementation assumes a specific mapping where the Z-coordinate is preserved for the slab check [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Constant time $O(1)$ as it performs a fixed number of intersection tests (5 surfaces) [S1, unofficial].
- **Memory Complexity**: $O(1)$ auxiliary space [S1, unofficial].
- **Strengths**: Handles the curved surface exactly via quadratic formula. Handles clipping planes efficiently [S1, unofficial].
- **Limitations**: The geometry is a complex segment (cylinder + slab), not a pure infinite cylinder. The "degenerate" fallback assumes a straight path through the center, which may be inaccurate for oblique rays [S1, unofficial].

## Background
This method is derived from a CUDA volume rendering implementation [S1, unofficial]. It is used in the context of ray casting through volume data where the geometry is defined by a combination of cylindrical and planar primitives [S1, unofficial].

## Sources
[S1] ojaisnielsen/dtu-2010-cuda-volume-rendering, https://github.com/ojaisnielsen/dtu-2010-cuda-volume-rendering, fetched 2026-10-27 (unofficial)
