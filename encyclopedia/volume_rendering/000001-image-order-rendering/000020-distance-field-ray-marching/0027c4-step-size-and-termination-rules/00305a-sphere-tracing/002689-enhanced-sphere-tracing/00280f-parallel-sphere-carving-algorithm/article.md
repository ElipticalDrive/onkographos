# Parallel Sphere Carving Algorithm


## Overview
The Parallel Sphere Carving Algorithm is a GPU-friendly implementation of the Sphere Carving method for extracting surfaces from conservative signed distance fields (SDFs) [S1]. It solves the problem of efficiently generating a set of empty spheres that bound a surface defined by an SDF $f$, specifically optimizing for parallel execution on graphics hardware [S1]. Unlike sequential sphere carving, which processes spheres one by one, this method uses four distinct parallel kernels to gather intersecting pairs, compute intersection triplets, filter interior points, and augment the sphere set simultaneously [S1].

## How it works
The algorithm performs a single iteration of sphere carving, taking a conservative SDF $f$ and a current array of empty spheres $S$ as input, and returning an updated array of empty spheres $S$ and an array of points $P$ whose convex hull bounds the surface [S1].

### Data Structures
All sets are implemented as pre-allocated arrays with sufficient memory [S1]. Parallel updates to these arrays require one atomic counter per array to push values [S1]. Each sphere $S_i$ is defined by its center $\mathbf{c}_i$ and radius $r_i = f(\mathbf{c}_i)$ [S1]. The algorithm uses four arrays: $S$ (empty spheres), $P$ (valid surface points), $P'$ (intermediate intersection points), and $pairs$ (array of sphere index pairs) [S1].

### Steps
1. **Gather Intersecting Pairs**: Initialize $pairs \leftarrow \emptyset$ [S1]. Parallel for each sphere $S_a \in S$, parallel for each sphere $S_b \in S$ with index $b > a$, if $S_a \cap S_b \neq \emptyset$ then $pairs \leftarrow pairs \cup (a, b)$ [S1].
2. **Compute Intersection Points**: Initialize $P, P' \leftarrow \emptyset$ [S1]. Parallel for each pair $(a, b) \in pairs$, parallel for each sphere $S_c \in S$ with index $c > b$, $P' \leftarrow P' \cup \text{intersections}(S_a, S_b, S_c)$ [S1].
3. **Filter Interior Points**: Parallel for each point $p \in P'$, parallel for each sphere $S \in S$, if $p \in \text{interior}(S)$ then discard $p$ [S1]. Otherwise, $P \leftarrow P \cup p$ [S1].
4. **Augment Sphere Set**: Parallel for each point $p \in P$, $S \leftarrow S \cup \text{sphere}(p, f(p))$ [S1].

### Intersection Computation
To compute the intersection points of three spheres $S_a, S_b, S_c$ with centers $\mathbf{c}_a, \mathbf{c}_b, \mathbf{c}_c$ and radii $r_a, r_b, r_c$:
1. Define $\mathbf{e}_x = \frac{\mathbf{c}_b - \mathbf{c}_a}{\|\mathbf{c}_b - \mathbf{c}_a\|}$ [S1].
2. Define $x = \frac{r_a^2 - r_b^2 + \|\mathbf{c}_b - \mathbf{c}_a\|^2}{2 \|\mathbf{c}_b - \mathbf{c}_a\|}$ [S1].
3. Define $\mathbf{e}_y = \frac{(\mathbf{c}_c - \mathbf{c}_a) - ((\mathbf{c}_c - \mathbf{c}_a) \cdot \mathbf{e}_x)\mathbf{e}_x}{\|(\mathbf{c}_c - \mathbf{c}_a) - ((\mathbf{c}_c - \mathbf{c}_a) \cdot \mathbf{e}_x)\mathbf{e}_x\|}$ [S1].
4. Define $y = \frac{r_a^2 - r_c^2 + \|\mathbf{c}_c - \mathbf{c}_a\|^2 - 2x((\mathbf{c}_c - \mathbf{c}_a) \cdot \mathbf{e}_x)}{2 \|(\mathbf{c}_c - \mathbf{c}_a) - ((\mathbf{c}_c - \mathbf{c}_a) \cdot \mathbf{e}_x)\mathbf{e}_x\|}$ [S1].
5. Define $z = \sqrt{r_a^2 - x^2 - y^2}$ [S1]. If the term under the square root is negative, there is no intersection [S1].
6. The base point is $\mathbf{p}_0 = \mathbf{c}_a + x\mathbf{e}_x + y\mathbf{e}_y$ and the intersection points are $\mathbf{p} = \mathbf{p}_0 \pm z\mathbf{e}_z$, where $\mathbf{e}_z = \mathbf{e}_x \times \mathbf{e}_y$ [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
from typing import List, Tuple

def dot(a: Tuple[float, float, float], b: Tuple[float, float, float]) -> float:
    return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]

def norm(a: Tuple[float, float, float]) -> float:
    return dot(a, a) ** 0.5

def cross(a: Tuple[float, float, float], b: Tuple[float, float, float]) -> Tuple[float, float, float]:
    return (a[1]*b[2] - a[2]*b[1], a[2]*b[0] - a[0]*b[2], a[0]*b[1] - a[1]*b[0])

def parallel_sphere_carving_iteration(S: List[Tuple[Tuple[float, float, float], float]], f) -> Tuple[List[Tuple[Tuple[float, float, float], float]], List[Tuple[float, float, float]]]:
    # S is a pre-allocated array of (center, radius) tuples.
    # Parallel updates use one atomic counter per array.
    
    pairs = []
    # Kernel 1: Gather intersecting sphere pairs
    for a in range(len(S)):
        ca, ra = S[a]
        for b in range(a + 1, len(S)):
            cb, rb = S[b]
            if norm((cb[0]-ca[0], cb[1]-ca[1], cb[2]-ca[2])) < ra + rb:
                pairs.append((a, b))
                
    P_prime = []
    # Kernel 2: Compute intersection points for triplets
    for (a, b) in pairs:
        ca, ra = S[a]
        cb, rb = S[b]
        dx_ab = cb[0] - ca[0]
        dy_ab = cb[1] - ca[1]
        dz_ab = cb[2] - ca[2]
        dist_ab = norm((dx_ab, dy_ab, dz_ab))
        ex = (dx_ab/dist_ab, dy_ab/dist_ab, dz_ab/dist_ab)
        x = (ra*ra - rb*rb + dist_ab*dist_ab) / (2.0 * dist_ab)
        
        for c in range(b + 1, len(S)):
            cc, rc = S[c]
            dx_ac = cc[0] - ca[0]
            dy_ac = cc[1] - ca[1]
            dz_ac = cc[2] - ca[2]
            v = (dx_ac, dy_ac, dz_ac)
            proj = dot(v, ex)
            ey_vec = (v[0] - proj*ex[0], v[1] - proj*ex[1], v[2] - proj*ex[2])
            ey_len = norm(ey_vec)
            if ey_len < 1e-9: continue
            ey = (ey_vec[0]/ey_len, ey_vec[1]/ey_len, ey_vec[2]/ey_len)
            y = (ra*ra - rc*rc + dot(v, v) - 2*x*proj) / (2.0 * ey_len)
            rad = ra*ra - x*x - y*y
            if rad < 0: continue
            z = rad ** 0.5
            p0 = (ca[0] + x*ex[0] + y*ey[0], ca[1] + x*ex[1] + y*ey[1], ca[2] + x*ex[2] + y*ey[2])
            ez = cross(ex, ey)
            p1 = (p0[0] + z*ez[0], p0[1] + z*ez[1], p0[2] + z*ez[2])
            p2 = (p0[0] - z*ez[0], p0[1] - z*ez[1], p0[2] - z*ez[2])
            P_prime.append(p1)
            P_prime.append(p2)
            
    P = []
    # Kernel 3: Filter points inside the volume
    for p in P_prime:
        inside = False
        for s in S:
            sc, sr = s
            if norm((p[0]-sc[0], p[1]-sc[1], p[2]-sc[2])) < sr:
                inside = True
                break
        if not inside:
            P.append(p)
            
    # Kernel 4: Augment sphere set
    for p in P:
        r = f(p)
        # In parallel implementation, use atomic counter to push to S
        S.append((p, r))
        
    return S, P
```

### Parameters and Edge Cases
*   **Atomic Counters**: Parallel array updates require one atomic counter per array to push values [S1].
*   **Precision**: Filtering points accounts for floating-point precision by discarding a point if its distance to any sphere center is less than the sphere radius [S1].
*   **Pre-allocation**: All sets are implemented as arrays assumed to be pre-allocated with sufficient memory [S1].

## Subtopics

### Icosphere Initialization
A strategy for initializing the Sphere Carving algorithm using a recursively subdivided icosahedron (icosphere) as the initial bounding volume $V_0$ [S1]. The vertices of the icosphere are used as the initial sample points $P$ on the surface of $V_0$ [S1]. Coverage verification is performed per triangle $\Delta(a, b, c)$ of the icosphere, ensuring that the triangle is covered by the union of the spheres centered at its vertices $a, b, c$ with radii $f(a), f(b), f(c)$ [S1]. The condition is $\forall \Delta \in V_0, \Delta \subset S(a) \cup S(b) \cup S(c)$ [S1]. If the condition is not met, the subdivision level is increased [S1]. This ensures a valid initial sphere set $S_0$ that covers the entire surface of $V_0$ [S1].

## Operating characteristics
*   **Time Complexity**: Not given in the sources.
*   **Memory Complexity**: Not given in the sources.
*   **Strengths**: Not given in the sources.
*   **Limitations**: Not given in the sources.
*   **When to use**: Not given in the sources.
*   **When not to use**: Not given in the sources.

## Background
The Parallel Sphere Carving Algorithm was introduced in "Sphere Carving: Bounding Volumes for Signed Distance Fields" [S1]. It provides a parallel-friendly implementation for a single Sphere Carving iteration, requiring four simple kernels [S1]. The method relies on a conservative signed distance field $f$ and an initial set of empty spheres $S$ [S1]. The algorithm was designed to address the need for efficient surface extraction from SDFs in parallel environments [S1].

## Sources
[S1] Sphere Carving: Bounding Volumes for Signed Distance Fields, https://doi.org/10.1145/3730845, fetched 2026-10-27
