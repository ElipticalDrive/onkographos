# Shading and Frame Buffer Filling

Also called: ISORender, Lambertian shading with no shadows, Packet Ray Casting with Lambertian Shading, Simple Lambertian shading with no shadows

## Overview
This method performs fast, shadow-less Lambertian shading for isosurfaces extracted from thresholded volumes [S3, unofficial]. It distinguishes itself from sibling methods by explicitly avoiding deferred normal computation to prevent prohibitive memory and computational costs associated with per-ray cell storage or repeated neighbor-finding [S2][S3, unofficial]. While the underlying ray traversal remains single-ray based, the shading phase aggregates rays into packets to exploit parallelism [S1, unofficial][S2][S3, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** `uClosest` texture containing the closest hit position and distance for each pixel [S1, unofficial].
- **Input:** `uLight` uniform `vec3` representing the light direction [S1, unofficial].
- **Input:** `uDiffuse` uniform `vec3` representing the diffuse color/material property [S1, unofficial].
- **Output:** `oColor` fragment color [S1, unofficial].

**Data Structures:**
- `uClosest`: A 2D texture where each texel stores a `vec4`. The components are interpreted as the 3D hit position $(x, y, z)$ and the distance $w$ along the ray [S1, unofficial]. If $w \le 0$, no intersection was found [S1, unofficial].

**Parameters:**
- Gradient step size $h = 0.005$ [S1, unofficial].

**Steps:**
1. **Sample Hit Position:** For each pixel, sample the closest hit position from the `uClosest` texture [S1, unofficial].
   $$ \text{closest} = \text{texture}(uClosest, vPosition) $$
   where $vPosition$ is the fragment coordinate [S1, unofficial].

2. **Check Intersection:** If $\text{closest}.w > 0$, an isosurface intersection exists [S1, unofficial]. Proceed to normal estimation and shading [S1, unofficial]. If $\text{closest}.w \le 0$, output white and terminate [S1, unofficial].
   $$ \text{if } \text{closest}.w \le 0: \quad oColor = (1.0, 1.0, 1.0, 1.0) $$

3. **Normal Estimation:** Compute the surface normal using gradient-based normal estimation [S1, unofficial]. The gradient is approximated using central differences on the volume data at the hit position [S1, unofficial]. The step size $h$ is fixed at $0.005$ [S1, unofficial].
   $$ \mathbf{n} = \nabla V(\mathbf{p}) \approx \left( \frac{V(\mathbf{p} + h\mathbf{e}_x) - V(\mathbf{p} - h\mathbf{e}_x)}{2h}, \frac{V(\mathbf{p} + h\mathbf{e}_y) - V(\mathbf{p} - h\mathbf{e}_y)}{2h}, \frac{V(\mathbf{p} + h\mathbf{e}_z) - V(\mathbf{p} - h\mathbf{e}_z)}{2h} \right) $$
   Normalize $\mathbf{n}$ to unit length [G].

4. **Lambertian Shading:** Compute the diffuse lighting factor using the dot product of the normal $\mathbf{n}$ and the light direction $\mathbf{l}$ (from `uLight`) [S1, unofficial].
   $$ \text{lambert} = \max(\mathbf{n} \cdot \mathbf{l}, 0.0) $$

5. **Output Color:** Multiply the material diffuse color by the lighting factor [S1, unofficial].
   $$ oColor = \text{uDiffuse} \times \text{lambert} $$
   The alpha component is set to $1.0$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Vec3:
    def __init__(self, x, y, z): self.x, self.y, self.z = x, y, z
    def __add__(self, other): return Vec3(self.x + other.x, self.y + other.y, self.z + other.z)
    def __sub__(self, other): return Vec3(self.x - other.x, self.y - other.y, self.z - other.z)
class Vec4:
    def __init__(self, x, y, z, w): self.x, self.y, self.z, self.w = x, y, z, w
def dot(a, b): return a.x*b.x + a.y*b.y + a.z*b.z
def normalize(v):
    mag = (v.x**2 + v.y**2 + v.z**2)**0.5
    return Vec3(v.x/mag, v.y/mag, v.z/mag)
def max_val(a, b): return a if a > b else b
def texture_2d(tex, uv): return tex[uv]
def texture_3d(tex, p): return tex[p]

def shading_and_frame_buffer_filling(uClosest, uLight, uDiffuse, uVolume, vPosition):
    # Step 1: Sample closest hit position and distance
    closest = texture_2d(uClosest, vPosition)
    
    # Step 2: Check intersection
    if closest.w <= 0.0:
        return Vec4(1.0, 1.0, 1.0, 1.0)
    
    # Step 3: Compute normal via gradient-based estimation
    h = 0.005
    p = Vec3(closest.x, closest.y, closest.z)
    
    # Central differences for gradient [S1, unofficial]
    gx = (texture_3d(uVolume, p + Vec3(h, 0.0, 0.0)).x - texture_3d(uVolume, p - Vec3(h, 0.0, 0.0)).x) / (2.0 * h)
    gy = (texture_3d(uVolume, p + Vec3(0.0, h, 0.0)).x - texture_3d(uVolume, p - Vec3(0.0, h, 0.0)).x) / (2.0 * h)
    gz = (texture_3d(uVolume, p + Vec3(0.0, 0.0, h)).x - texture_3d(uVolume, p - Vec3(0.0, 0.0, h)).x) / (2.0 * h)
    normal = normalize(Vec3(gx, gy, gz))
    
    # Step 4: Compute Lambertian diffuse factor
    light_dir = normalize(uLight)
    lambert = max_val(dot(normal, light_dir), 0.0)
    
    # Step 5: Output color
    oColor = Vec4(uDiffuse.x * lambert, uDiffuse.y * lambert, uDiffuse.z * lambert, 1.0)
    return oColor
```

**Edge Cases and Pitfalls:**
- **No Intersection:** Pixels with no hit output white [S1, unofficial].
- **Normal Computation Cost:** Deferring normal computation is avoided due to the prohibitive cost of storing each cell per ray or repeating neighbor-finding [S2][S3, unofficial].
- **Precision:** The method yields single-precision, numerically stable results [S2][S3, unofficial].

## Subtopics

### [Lambertian Shading with Packet Architecture](2241-lambertian-shading-with-packet-architecture.md)
This method performs fast, shadow-less Lambertian shading for isosurfaces extracted from thresholded volumes by batching ray generation and shading operations using a packet architecture. It is distinguished from sibling methods by explicitly avoiding deferred normal computation to prevent prohibitive memory and computational costs associated with per-ray cell storage or repeated neighbor-finding. While the underlying ray traversal remains single-ray based, the shading phase aggregates rays into packets to exploit parallelism.

## Operating characteristics
- **Time Complexity:** The shading pass is $O(1)$ per pixel, dominated by texture sampling and arithmetic operations [G]. The packet architecture allows diffuse shading to be performed in batch, likely delivering a speedup over a more conventional single-ray tracer [S2][S3, unofficial].
- **Memory Complexity:** Not given in the sources.
- **Performance:** Suitable for fast ray casting of isosurfaces where shadowing is not required [S2][S3, unofficial].
- **Strengths:** Simple implementation, low memory footprint, fast shading via packet batching.
- **Limitations:** No shadows, limited to Lambertian shading, normal estimation depends on gradient accuracy.

## Background
This method is described in the context of interactive isosurface ray tracing of large octree volumes [S2][S3, unofficial].

## Sources
[S1] rokcej/volumetric-ray-casting, https://github.com/rokcej/volumetric-ray-casting, fetched 2026-10-26 (unofficial)
[S2] PDF Interactive Isosurface Ray Tracing of Large Octree Volumes, https://collections.lib.utah.edu/dl_files/1d/58/1d589a03a725c9d894a3866ad32f1386f0613f9d.pdf, fetched 2026-10-28
[S3] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.researchgate.net/publication/261082346_Interactive_Isosurface_Ray Tracing of Large Octree Volumes, fetched 2026-10-29 (unofficial)
