# Volumetric Raymarching with SDF Density
Also called: Raymarching with a twist, Signed distance field (SDF) ray marching, Volumetric Raymarching

## Overview
Volumetric Raymarching with SDF Density is a rendering technique that adapts sphere tracing principles to volumetric Signed Distance Fields (SDFs), encoding both interior density and exterior distance in a single continuous function [S1][S2, unofficial]. It solves the problem of rendering intricate procedural volumes, such as fractals or noise-based densities, without requiring explicit voxelization or precomputed grids [S1]. Unlike standard raymarching which stops at the first surface intersection, this method marches through the object interior using a constant step size, accumulating color and alpha via front-to-back blending based on the SDF's positive interior values [S2, unofficial]. It sets itself apart from [Fixed Step Ray Marching](3136-fixed-step-ray-marching.md) by interpreting the distance field as density rather than geometric proximity, and from [Tiled Ray Marching with Grid Sampling](2586-tiled-ray-marching-with-grid-sampling.md) by relying on procedural evaluation rather than grid lookups [S1][S2, unofficial].

## How it works
**Inputs:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$ (normalized), SDF function $f(\mathbf{p})$ that returns $-d$ (negative of Euclidean distance to surface) for points inside the volume, and $0$ or positive for points outside.
**Outputs:** Final color $\mathbf{C}$ and accumulated alpha $\alpha$.

**Data Structures:**
*   `rayOrigin`: `vec3`
*   `rayDirection`: `vec3`
*   `res`: `vec4` (accumulator for color and alpha)
*   `depth`: `float` (distance along ray)
*   `p`: `vec3` (current sample position)

**Parameters:**
*   `MARCH_SIZE`: Constant step size. Typical value: $0.08$ [S2, unofficial].
*   `MAX_STEPS`: Maximum iterations. Typical value: $100$ [S2, unofficial].

**Steps:**
1.  Initialize `depth` to $0.0$ and `res` to $(0.0, 0.0, 0.0, 0.0)$.
2.  Calculate initial position $\mathbf{p} = \mathbf{o} + \text{depth} \cdot \mathbf{d}$.
3.  Loop for $i$ from $0$ to `MAX_STEPS` - 1:
    a. Evaluate density $D = f(\mathbf{p})$.
    b. If $D > 0.0$:
        i. Compute sample color $\mathbf{C}_{sample}$. A simple mapping mixes white $(1,1,1)$ and black $(0,0,0)$ based on density: $\mathbf{C}_{sample} = \text{mix}((1,1,1), (0,0,0), D)$.
        ii. Set sample alpha $\alpha_{sample} = D$.
        iii. Apply alpha to color: $\mathbf{C}_{sample} \leftarrow \mathbf{C}_{sample} \cdot \alpha_{sample}$.
        iv. Accumulate using front-to-back blending:
            $$ \mathbf{res} \leftarrow \mathbf{res} + \mathbf{C}_{sample} \cdot (1.0 - \mathbf{res}.a) $$
    c. Advance depth: $\text{depth} \leftarrow \text{depth} + \text{MARCH\_SIZE}$.
    d. Update position: $\mathbf{p} \leftarrow \mathbf{o} + \text{depth} \cdot \mathbf{d}$.
4.  Return `res`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def raymarch(ray_origin, ray_direction):
    # Constants
    MARCH_SIZE = 0.08
    MAX_STEPS = 100
    
    depth = 0.0
    p = ray_origin + depth * ray_direction
    res = vec4(0.0, 0.0, 0.0, 0.0)
    
    for i in range(MAX_STEPS):
        # Evaluate SDF; returns -distance inside, >= 0 outside
        density = scene(p)
        
        if density > 0.0:
            # Map density to color (white to black)
            # mix(a, b, t) returns a + t * (b - a)
            sample_color = mix(vec3(1.0, 1.0, 1.0), vec3(0.0, 0.0, 0.0), density)
            sample_alpha = density
            
            # Apply alpha to color
            sample_color = sample_color * sample_alpha
            
            # Front-to-back blending
            res = res + sample_color * (1.0 - res.a)
            
        depth = depth + MARCH_SIZE
        p = ray_origin + depth * ray_direction
        
    return res
```

**Edge Cases and Pitfalls:**
*   **Step Size:** Using a constant step size is critical. The distance returned by the SDF must NOT be used for stepping, as it represents density magnitude, not geometric clearance [S2, unofficial].
*   **SDF Sign:** The SDF function must return the negative of the distance ($-d$) so that values are positive inside the object and zero outside. Standard SDFs return positive inside; this inversion is required for the density check $D > 0.0$ [S2, unofficial].
*   **Precision:** For complex, non-intersecting surfaces like fractals, fixed-step methods may require more iterations than sphere tracing to converge, potentially leading to aliasing in thin features if the step size is too large [S1].

## Subtopics

### [Volumetric Light Shafts (Cone Raymarching)](4565-volumetric-light-shafts-cone-raymarching.md)
A post-processing technique using cone-shaped SDFs for directional light shafts in screen space. Chosen when rendering volumetric lighting effects from specific light sources rather than general volume density.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is `MAX_STEPS`. Independent of scene complexity as the SDF is evaluated procedurally [S1].
*   **Memory Complexity:** $O(1)$ auxiliary memory, as no voxel grid or acceleration structure is stored [S1].
*   **Performance:** Efficient for real-time procedural volumes (e.g., clouds, smoke) on GPUs due to lack of precomputation [S1]. Performance scales with `MAX_STEPS` and `MARCH_SIZE`; smaller steps increase accuracy but reduce frame rate.
*   **Strengths:** Watertight traversal without precomputation; compact representation; avoids aliasing in thin features by refining steps near boundaries (if step size is adaptive, though this method uses fixed steps) [S1].
*   **Limitations:** Fixed step size may miss thin features or require high `MAX_STEPS` for accuracy; no explicit handling of scattering or complex lighting beyond simple color mapping in the base method [S1][S2, unofficial].
*   **When to use:** Procedural volumes where explicit voxelization is infeasible, real-time rendering of clouds/smoke, differentiable optimization for scene fitting [S1].
*   **When not to use:** When high-fidelity light transport (multiple scattering) is required without additional shaders; when exact geometric boundaries are needed without density interpretation.

## Background
Originating from the adaptation of sphere tracing principles to volumetric SDFs, this method was popularized in real-time graphics communities for rendering procedural clouds and smoke [S1][S2, unofficial]. It enables compact representation of intricate procedural volumes by evaluating the SDF on-the-fly, supporting applications like real-time procedural terrain or cloud volumes [S1]. The technique leverages the continuous nature of SDFs to encode interior density, allowing for differentiable optimization in neural reconstruction contexts [S1].

## Sources
[S1] Volume ray casting, https://grokipedia.com/page/Volume_ray_casting, fetched 2026-10-26
[S2] Real-time dreamy Cloudscapes with Volumetric Raymarching, https://blog.maximeheckel.com/posts/real-time-cloudscapes-with-volumetric-raymarching/, fetched 2026-10-28 (unofficial)
