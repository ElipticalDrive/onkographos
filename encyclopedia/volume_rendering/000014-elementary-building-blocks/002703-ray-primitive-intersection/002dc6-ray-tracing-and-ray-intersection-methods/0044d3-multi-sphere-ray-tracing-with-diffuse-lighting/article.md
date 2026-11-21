# Multi-Sphere Ray Tracing with Diffuse Lighting

Also called: Primary Ray Tracing Loop, sphIntrAll

## Overview
This method casts a primary ray against a fixed set of spheres to find the closest intersection and computes unshadowed diffuse lighting from multiple point lights [S1, unofficial]. It omits recursive shadow rays and reflections, distinguishing it from full ray tracing implementations [G].

## How it works
**Inputs:** Ray origin `org`, ray direction `dir`, array of spheres `S` with count `sphCnt`, array of lights `L` with count `lightCnt` [S1, unofficial].
**Outputs:** Final color `color` (vec4) [S1, unofficial].

**Data Structures:**
*   `struct intr`: `bool is` (hit status), `float t` (distance to hit), `vec3 N` (surface normal at hit) [S1, unofficial].
*   `struct material`: `vec3 kd` (diffuse color coefficient), `float l` (unused) [S1, unofficial].
*   `struct light`: `vec3 p` (position), `float intensiv` (intensity) [S1, unofficial].
*   `struct sph`: `float r` (radius), `vec3 c` (center), `material m` (material) [S1, unofficial].

**Steps:**
1. Initialize closest hit `mint` with `is=false`, `t=1000`, `N=vec3(0)` [S1, unofficial].
2. Loop over all spheres `i` from `0` to `sphCnt-1` [S1, unofficial].
3. Compute intersection of ray `(org, dir)` with sphere `S[i]` using `sphInt` [S1, unofficial].
4. If `intersect.is` is true and `intersect.t < mint.t`, update `mint = intersect`, store material `mm = S[i].m`, and set `color = vec4(mm.kd, 1)` [S1, unofficial].
5. If `mint.is` is true, initialize `difInt = 0` [S1, unofficial].
6. Loop over all lights `i` from `0` to `lightCnt-1` [S1, unofficial].
7. Compute hit point `P = org + dir * mint.t` [S1, unofficial].
8. Compute light direction `ld = normalize(L[i].p - P)` [S1, unofficial]. (Corrected from source code where normalization was incorrectly applied to `P` instead of the difference vector)
9. Accumulate diffuse contribution: `difInt += L[i].intensiv * max(0., dot(ld, mint.N))` [S1, unofficial].
10. Set final color: `color = vec4(mm.kd * difInt, 1)` [S1, unofficial].
11. Return `color` [S1, unofficial].

**Formula:**
The final color $C$ is calculated as:
$$ C = \mathbf{k}_d \sum_{j=1}^{L} I_j \max(0, \mathbf{d}_j \cdot \mathbf{N}) $$
Where $\mathbf{k}_d$ is the diffuse color coefficient, $L$ is the number of lights, $I_j$ is the intensity of light $j$, $\mathbf{d}_j$ is the normalized direction from the hit point to light $j$, and $\mathbf{N}$ is the surface normal at the hit point [S1, unofficial].

**Parameters:**
*   `sphCnt`: Number of spheres. Typical value: 3 [S1, unofficial].
*   `lightCnt`: Number of lights processed. Typical value: 1 (array contains 2) [S1, unofficial].
*   `mint.t`: Initial maximum ray distance. Typical value: 1000 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_sphere_diffuse(org, dir, S, sphCnt, L, lightCnt):
    mint_is = False
    mint_t = 1000.0
    mint_N = (0.0, 0.0, 0.0)
    mm_kd = (0.0, 0.0, 0.0)

    for i in range(sphCnt):
        intersect = sphInt(S[i], org, dir)
        if intersect[0] and intersect[1] < mint_t:
            mint_is = True
            mint_t = intersect[1]
            mint_N = intersect[2]
            mm_kd = S[i].m.kd

    if mint_is:
        difInt = 0.0
        hit_point = (org[0] + dir[0] * mint_t, org[1] + dir[1] * mint_t, org[2] + dir[2] * mint_t)
        for j in range(lightCnt):
            ld = normalize((L[j].p[0] - hit_point[0], L[j].p[1] - hit_point[1], L[j].p[2] - hit_point[2]))
            difInt += L[j].intensiv * max(0.0, dot(ld, mint_N))
        color = (mm_kd[0] * difInt, mm_kd[1] * difInt, mm_kd[2] * difInt, 1.0)
    else:
        color = (0.0, 0.0, 0.0, 1.0)
    return color

def sphInt(sph, org, dir):
    oc = (org[0] - sph.c[0], org[1] - sph.c[1], org[2] - sph.c[2])
    a = dot(dir, dir)
    b = 2.0 * dot(oc, dir)
    c = dot(oc, oc) - sph.r * sph.r
    disc = b * b - 4 * a * c
    if disc < 0:
        return (False, 1000.0, (0.0, 0.0, 0.0))
    sqrt_disc = sqrt(max(0.0, disc))
    t1 = (-b - sqrt_disc) / (2 * a)
    t2 = (-b + sqrt_disc) / (2 * a)
    t = t1 if t1 > 0.0001 else t2
    if t < 0.0001:
        return (False, 1000.0, (0.0, 0.0, 0.0))
    hit_pt = (org[0] + dir[0] * t, org[1] + dir[1] * t, org[2] + dir[2] * t)
    N = normalize((hit_pt[0] - sph.c[0], hit_pt[1] - sph.c[1], hit_pt[2] - sph.c[2]))
    return (True, t, N)

def normalize(v):
    l = sqrt(v[0]**2 + v[1]**2 + v[2]**2)
    return (v[0]/l, v[1]/l, v[2]/l)

def dot(v1, v2):
    return v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]

def sqrt(x):
    return x**0.5
```

**Edge Cases:**
*   **No Hit:** If no sphere is intersected, the method returns a default color (implementation dependent, often black or background) [G].
*   **Precision:** Using a large initial `mint.t` (1000) may cause precision issues if the scene is scaled differently [G].
*   **Light Direction:** The light direction is computed from the hit point to the light position, not from the light to the hit point, to ensure correct dot product sign for diffuse shading [S1, unofficial].

## Subtopics

### Sphere Intersection
The core primitive test. See [Ray Tracing and Ray Intersection Methods](5617-ray-tracing-and-ray-intersection-methods.md).

### Diffuse Lighting Model
The shading model used. See [Ray Tracing](8334-ray-tracing.md).

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Simple to implement, no recursion, suitable for static scenes with few primitives [G].
*   **Limitations:** No shadows, reflections, or refractions [G].
*   **When to Use:** Educational purposes, simple demos, or scenes with very few objects and lights [G].
*   **When Not to Use:** Complex scenes, real-time rendering, or when accurate lighting (shadows, reflections) is required [G].

## Background
This method is a basic ray tracing technique often used in introductory computer graphics courses and simple renderers [G]. It demonstrates the fundamental concepts of ray-primitive intersection and diffuse shading without the complexity of recursive ray tracing [G].

## Sources
[S1] trimesh33/sdf, https://github.com/trimesh33/sdf, fetched 2026-10-27 (unofficial)
