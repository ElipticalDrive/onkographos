# Raymarch Blending

Also called: Alpha Blending Raymarch

## Overview
Raymarch Blending combines a ray-marched volumetric result with an existing background scene texture using standard alpha compositing [S1, unofficial]. It solves the problem of integrating semi-transparent volumetric data into a pre-rendered scene containing opaque geometry, ensuring correct occlusion where the volume interacts with the solid meshes [S1, unofficial]. Unlike [Alpha Compositing Ray Casting](922-alpha-compositing-ray-casting.md) which typically handles volume-to-volume or volume-to-black blending, this method explicitly blends against a pre-existing framebuffer color [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A pre-rendered scene texture containing the background color and opaque geometry; ray origin (`ro`) and ray direction (`rd`) for the current fragment [S1, unofficial].
*   **Output:** A final fragment color that blends the raymarched volume over the background [S1, unofficial].

**Data Structures:**
*   Scene texture: A 2D texture sampling the scene color prior to this shader pass [S1, unofficial].
*   `raymarch(ro, rd)`: A function returning a 4D vector where the first three components are the accumulated color and the fourth component is the accumulated opacity (alpha) of the volume along the ray [S1, unofficial].

**Steps:**
1.  **Sample Background:** Retrieve the existing scene color from the background texture at the current UV coordinates [S1, unofficial].
    $$ C_{bg} = \text{tex2D}(\text{scene\_texture}, uv) $$
2.  **Raymarch Volume:** Compute the volumetric contribution by marching along the ray [S1, unofficial].
    $$ C_{vol}, \alpha_{vol} = \text{raymarch}(ro, rd) $$
3.  **Blend:** Combine the background color and the volumetric result using the standard alpha blending equation [S1, unofficial].
    $$ C_{final} = C_{bg} \cdot (1.0 - \alpha_{vol}) + C_{vol} \cdot \alpha_{vol} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def normalize(v):
    length = (v[0]**2 + v[1]**2 + v[2]**2)**0.5
    return [v[0]/length, v[1]/length, v[2]/length]

def sample_texture(tex, uv):
    # Returns a 3D list [r, g, b]
    return tex.sample(uv)

def raymarch(ro, rd):
    # Returns a 4D list [r, g, b, a]
    # Implementation details are in the raymarching method [G]
    return [0.0, 0.0, 0.0, 0.0]

def frag_shader(i, camera_ws, main_tex):
    # 1. Calculate ray parameters
    rd = normalize(i.ray)
    ro = camera_ws

    # 2. Sample the existing scene color from the background texture
    col_bg = sample_texture(main_tex, i.uv)

    # 3. Perform raymarching to get volumetric color and alpha
    add = raymarch(ro, rd)

    # 4. Blend the volumetric result with the background
    # Formula: C_bg * (1 - a) + C_vol * a
    final_rgb = [
        col_bg[0] * (1.0 - add[3]) + add[0] * add[3],
        col_bg[1] * (1.0 - add[3]) + add[1] * add[3],
        col_bg[2] * (1.0 - add[3]) + add[2] * add[3]
    ]

    # Return 4D color with alpha = 1.0
    return final_rgb + [1.0]
```

**Parameters:**
*   `maxstep`: The maximum number of samples for the raymarch [S1, unofficial]. The source notes that 64 samples were used for a simple torus example, but suggests tweaking this value based on performance and accuracy needs [S1, unofficial].
*   `raymarch` parameters: Step size, density thresholds, and other marching logic are handled internally by the `raymarch` function and are not specified in the blending context [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Performance:** Increasing `maxstep` increases accuracy but incurs a proportional performance cost [S1, unofficial].
*   **Pipeline Dependency:** This method assumes the background texture already contains the correct opaque geometry from previous rendering passes [S1, unofficial].

## Subtopics

### [Back-to-Front Alpha Blending with 3D Geometry](6566-back-to-front-alpha-blending-with-3d-geometry.md)
Composites a ray-marched volume over pre-rendered opaque 3D polygonal geometry by leveraging standard graphics pipeline alpha blending. It solves the occlusion problem where volumetric data must correctly interact with solid meshes, ensuring that the volume is blended against the background color already present in the framebuffer from previous rendering passes. Unlike standard ray casting which only evaluates volume density, this approach ensures correct occlusion of the volume by solid geometry, solving the problem of rendering transparent volumes over opaque meshes in a single pass or specific pipeline configuration.

## Operating characteristics
*   **Time Complexity:** Dominated by the `raymarch` function, which is $O(N)$ where $N$ is the number of samples (`maxstep`) [S1, unofficial].
*   **Memory Complexity:** $O(1)$ additional memory per fragment for the blending calculation, assuming the background texture is already in memory [S1, unofficial].
*   **Strengths:** Simple to implement; allows for easy integration of volumes into existing scenes with opaque geometry; leverages standard alpha blending hardware [S1, unofficial].
*   **Limitations:** Performance depends heavily on the `raymarch` implementation [S1, unofficial].
*   **When to use:** When rendering semi-transparent volumes over opaque 3D models in a deferred or multi-pass pipeline [S1, unofficial].
*   **When not to use:** Not given in the sources.

## Background
This method is described in an unofficial tutorial on raymarching distance fields [S1, unofficial]. It is presented as a practical technique for integrating raymarched volumes into a standard graphics pipeline, specifically addressing the common scenario of adding volumetric effects to scenes with pre-existing geometry [S1, unofficial].

## Sources
[S1] Raymarching Distance Fields: Concepts and Implementation ..., https://adrianb.io/2016/10/01/raymarching.html, fetched 2026-10-27 (unofficial)
