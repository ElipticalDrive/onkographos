# Volumetric Raymarching with SDF Density
Also called: Raymarching with a twist, Signed distance field (SDF) ray marching, Volumetric Raymarching

## Overview
Volumetric Raymarching with SDF Density is a rendering technique that adapts sphere tracing principles to volumetric Signed Distance Fields (SDFs), encoding both interior density and exterior distance in a single continuous function [S1][S2, unofficial]. It solves the problem of rendering intricate procedural volumes, such as fractals or noise-based densities, without requiring explicit voxelization or precomputed grids [S1]. Unlike standard raymarching which stops at the first surface intersection, this method marches through the object interior using a constant step size, accumulating color and alpha via front-to-back blending based on the SDF's positive interior values [S2, unofficial]. It sets itself apart from [Fixed Step Ray Marching](3136-fixed-step-ray-marching.md) by interpreting the distance field as density rather than geometric proximity, and from [Front-to-Back Volume Rendering](2350-front-to-back-volume-rendering.md) by evaluating the density function procedurally on-the-fly rather than sampling a discrete grid [S1][S2, unofficial].

## How it works
**Inputs:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$ (normalized), and an SDF function $f(\mathbf{p})$ that returns the negative Euclidean distance to the surface for points inside the volume, and zero for points outside [S2, unofficial].
**Outputs:** A `vec4` accumulator containing the final color $\mathbf{C}$ and accumulated alpha $\alpha$ [S2, unofficial].

**Data Structures:**
*   `rayOrigin`: `vec3` representing the ray start point [S2, unofficial].
*   `rayDirection`: `vec3` representing the normalized ray direction [S2, unofficial].
*   `res`: `vec4` accumulator for color and alpha [S2, unofficial].
*   `depth`: `float` tracking distance traveled along the ray [S2, unofficial].
*   `p`: `vec3` current sample position [S2, unofficial].

**Parameters:**
*   `MARCH_SIZE`: Constant step size for traversal. Typical value: $0.08$ [S2, unofficial].
*   `MAX_STEPS`: Maximum number of iterations in the raymarching loop. Typical value: $100$ [S2, unofficial].

**Steps:**
1.  Initialize `depth` to $0.0$ and `res` to $(0.0, 0.0, 0.0, 0.0)$ [S2, unofficial].
2.  Calculate the initial sample position $\mathbf{p} = \mathbf{o} + \text{depth} \cdot \mathbf{d}$ [S2, unofficial].
3.  Iterate $i$ from $0$ to `MAX_STEPS` - 1:
    a. Evaluate the SDF at the current position to obtain density $D = f(\mathbf{p})$ [S2, unofficial].
    b. If $D > 0.0$:
        i. Construct a `vec4` color where the RGB channels are interpolated between white $(1,1,1)$ and black $(0,0,0)$ based on $D$, and the alpha channel is set to $D$ [S2, unofficial].
        ii. Modulate the RGB channels by the alpha channel: $\mathbf{C}_{sample}.\text{rgb} \leftarrow \mathbf{C}_{sample}.\text{rgb} \cdot D$ [S2, unofficial].
        iii. Accumulate the sample into the result using front-to-back blending: $\mathbf{res} \leftarrow \mathbf{res} + \mathbf{C}_{sample} \cdot (1.0 - \mathbf{res}.a)$ [S2, unofficial].
    c. Advance the traversal distance: $\text{depth} \leftarrow \text{depth} + \text{MARCH\_SIZE}$ [S2, unofficial].
    d. Update the sample position: $\mathbf{p} \leftarrow \mathbf{o} + \text{depth} \cdot \mathbf{d}$ [S2, unofficial].
4.  Return `res` [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vec3(x, y, z): return (x, y, z)
def vec4(x, y, z, w): return (x, y, z, w)
def mix(a, b, t): return tuple(a[i] + t * (b[i] - a[i]) for i in range(len(a)))

def raymarch(ray_origin, ray_direction, f):
    MARCH_SIZE = 0.08
    MAX_STEPS = 100
    
    depth = 0.0
    p = vec3(ray_origin[0] + depth * ray_direction[0],
             ray_origin[1] + depth * ray_direction[1],
             ray_origin[2] + depth * ray_direction[2])
    res = vec4(0.0, 0.0, 0.0, 0.0)
    
    for i in range(MAX_STEPS):
        density = f(p)
        
        if density > 0.0:
            color = vec4(*mix(vec3(1.0, 1.0, 1.0), vec3(0.0, 0.0, 0.0), density), density)
            color = vec4(color[0] * color[3], color[1] * color[3], color[2] * color[3], color[3])
            res = vec4(res[0] + color[0] * (1.0 - res[3]),
                       res[1] + color[1] * (1.0 - res[3]),
                       res[2] + color[2] * (1.0 - res[3]),
                       res[3] + color[3] * (1.0 - res[3]))
            
        depth = depth + MARCH_SIZE
        p = vec3(ray_origin[0] + depth * ray_direction[0],
                 ray_origin[1] + depth * ray_direction[1],
                 ray_origin[2] + depth * ray_direction[2])
        
    return res
```

**Edge Cases and Pitfalls:**
*   **Step Size:** A constant step size must be used for traversal; the magnitude returned by the SDF represents density, not geometric clearance, and must never be used to scale the step [S2, unofficial].
*   **SDF Sign:** The SDF function must invert the standard distance convention by returning $-d$ inside the volume so that density values are strictly positive within the object and zero outside, enabling the $D > 0.0$ occupancy test [S2, unofficial].
*   **Boundary Precision:** Fixed step sizes may undersample thin volumetric features or require higher `MAX_STEPS` to avoid aliasing near density gradients [S1].

## Subtopics

### [Volumetric Light Shafts (Cone Raymarching)](4565-volumetric-light-shafts-cone-raymarching.md)
A post-processing technique using cone-shaped SDFs for directional light shafts in screen space. Chosen when rendering volumetric lighting effects from specific light sources rather than general volume density.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is `MAX_STEPS`, as each iteration performs a constant number of SDF evaluations and arithmetic operations [S1].
*   **Memory Complexity:** $O(1)$ auxiliary memory, since the method evaluates the SDF procedurally without storing voxel grids or acceleration structures [S1].
*   **Performance:** Highly efficient for real-time procedural volumes on GPUs due to the absence of precomputation and memory bandwidth bottlenecks [S1]. Frame rate scales linearly with `MAX_STEPS` and inversely with `MARCH_SIZE`, as smaller steps increase traversal iterations [S2, unofficial].
*   **Strengths:** Provides watertight traversal without precomputation; enables compact representation of complex procedural volumes; supports differentiable optimization for neural scene fitting [S1].
*   **Limitations:** Fixed step size cannot adapt to local density gradients, potentially missing thin features or requiring high iteration counts for accuracy; the base formulation does not model multiple scattering or complex light transport [S1][S2, unofficial].
*   **When to use:** Real-time rendering of procedural clouds, smoke, or terrain where explicit voxelization is infeasible, or when differentiable volume fitting is required [S1].

## Background
The method originates from adapting sphere tracing principles to volumetric SDFs, popularized in real-time graphics communities for rendering procedural clouds and smoke [S1][S2, unofficial]. It enables compact representation of intricate procedural volumes by evaluating the SDF on-the-fly, supporting applications like real-time procedural terrain or cloud volumes [S1]. The technique leverages the continuous nature of SDFs to encode interior density, allowing for differentiable optimization in neural reconstruction contexts [S1].

## Sources
[S1] Volume ray casting, https://grokipedia.com/page/Volume_ray_casting, fetched 2026-10-26
[S2] Real-time dreamy Cloudscapes with Volumetric Raymarching, https://blog.maximeheckel.com/posts/real-time-cloudscapes-with-volumetric-raymarching/, fetched 2026-10-28 (unofficial)
