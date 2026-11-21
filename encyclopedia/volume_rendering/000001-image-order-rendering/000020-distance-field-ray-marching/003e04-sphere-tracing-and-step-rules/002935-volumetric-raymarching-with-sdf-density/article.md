# Volumetric Raymarching with SDF Density

Also called: Raymarching with a twist, Signed distance field (SDF) ray marching, Volumetric Raymarching

## Overview
Volumetric Raymarching with SDF Density adapts sphere tracing principles to render volumetric objects by treating the Signed Distance Function (SDF) as a density field rather than a surface boundary [S1][S2, unofficial]. Unlike standard sphere tracing which terminates upon surface intersection, this method marches through the object's interior using a constant step size, accumulating color and alpha based on the SDF's positive values [S2, unofficial]. It enables the compact, procedural representation of intricate volumes like clouds or smoke without explicit voxelization [S1].

## How it works
**Inputs and Outputs:**
- **Inputs:** Ray origin $\mathbf{o}$, normalized ray direction $\mathbf{d}$, a density function $f(\mathbf{p})$ derived from an SDF, maximum step count $N_{\text{max}}$, constant step size $s$, and convergence threshold $\epsilon$ [S1][S2, unofficial].
- **Outputs:** Accumulated color $C_{\text{out}}$ and alpha $\alpha_{\text{out}}$ [S2, unofficial].

**Data Structures:**
- **Density Function $f(\mathbf{p})$:** A scalar field where $f(\mathbf{p}) > 0$ inside the volume, $f(\mathbf{p}) = 0$ on the boundary, and $f(\mathbf{p}) < 0$ outside [S2, unofficial]. Note that this is the negation of the standard SDF definition ($f_{\text{std}}(\mathbf{p}) = -f(\mathbf{p})$) [S2, unofficial].

**Parameters:**
- **MARCH_SIZE ($s$):** Constant step size. Typical value: $0.08$ [S2, unofficial].
- **MAX_STEPS ($N_{\text{max}}$):** Maximum iterations. Typical value: $100$ [S2, unofficial].
- **$\epsilon$:** Small tolerance for convergence checks if used, though constant stepping often bypasses dynamic step sizing [S1].

**Algorithm Steps:**
1.  **Initialize:** Set accumulated distance $t = 0$, current position $\mathbf{p} = \mathbf{o}$, and result color/alpha $C_{\text{res}} = (0,0,0,0)$ [S2, unofficial].
2.  **Loop:** For $i$ from $0$ to $N_{\text{max}}-1$:
    a.  **Sample Density:** Evaluate $d = f(\mathbf{p})$ [S2, unofficial].
    b.  **Check Density:** If $d > 0$:
        i.  Compute sample color $C_{\text{sample}}$ and alpha $\alpha_{\text{sample}}$. A simple mapping uses density directly for alpha: $\alpha_{\text{sample}} = d$ [S2, unofficial]. Color is interpolated between white and black based on density: $C_{\text{sample}} = \text{mix}(\text{white}, \text{black}, d)$ [S2, unofficial].
        ii. Apply lighting/attenuation if applicable (e.g., $C_{\text{sample}}.\text{rgb} *= C_{\text{sample}}.\text{a}$) [S2, unofficial].
        iii. **Front-to-Back Blending:** Update result:
            $$ C_{\text{res}} = C_{\text{res}} + C_{\text{sample}} \cdot (1 - \alpha_{\text{res}}) $$
            $$ \alpha_{\text{res}} = \alpha_{\text{res}} + \alpha_{\text{sample}} \cdot (1 - \alpha_{\text{res}}) $$
            [S2, unofficial]
    c.  **Step:** Advance ray: $t \leftarrow t + s$, $\mathbf{p} \leftarrow \mathbf{o} + t \cdot \mathbf{d}$ [S2, unofficial].
3.  **Return:** $C_{\text{res}}$ and $\alpha_{\text{res}}$ [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volumetric_raymarch_sdf_density(ray_origin, ray_direction, density_func, max_steps=100, march_size=0.08):
    """
    Renders a volume using SDF density with constant step marching.
    
    Args:
        ray_origin: vec3, camera origin
        ray_direction: vec3, normalized ray direction
        density_func: callable, returns positive value inside volume, 0 on boundary, negative outside
        max_steps: int, maximum number of march steps
        march_size: float, constant step size (default 0.08)
        
    Returns:
        vec4: Accumulated color (RGB) and alpha (A)
    """
    t = 0.0
    p = ray_origin + t * ray_direction
    res_color = [0.0, 0.0, 0.0]
    res_alpha = 0.0
    
    for _ in range(max_steps):
        # Sample density from the SDF-derived function
        # Note: The function must return positive values inside the volume
        density = density_func(p)
        
        if density > 0.0:
            # Map density to color and alpha
            # Simple example: White to Black based on density
            # Alpha is directly proportional to density
            alpha_sample = density
            
            # Interpolate color: mix(white, black, density)
            # In Python-like logic: color = white * (1 - density) + black * density
            # Assuming density is normalized or scaled appropriately
            r = 1.0 * (1.0 - density) + 0.0 * density
            g = 1.0 * (1.0 - density) + 0.0 * density
            b = 1.0 * (1.0 - density) + 0.0 * density
            
            sample_color = [r, g, b]
            
            # Attenuate color by alpha (as per source [S2, unofficial])
            sample_color[0] *= alpha_sample
            sample_color[1] *= alpha_sample
            sample_color[2] *= alpha_sample
            
            # Front-to-back blending
            # C_res = C_res + C_sample * (1 - alpha_res)
            # alpha_res = alpha_res + alpha_sample * (1 - alpha_res)
            
            weight = 1.0 - res_alpha
            res_color[0] += sample_color[0] * weight
            res_color[1] += sample_color[1] * weight
            res_color[2] += sample_color[2] * weight
            res_alpha += alpha_sample * weight
            
        # Constant step advancement
        t += march_size
        p = ray_origin + t * ray_direction
        
    return res_color + [res_alpha]

```

**Edge Cases and Pitfalls:**
- **Density Scaling:** The SDF must be transformed to return positive values inside the volume. Standard SDFs return negative values inside [S2, unofficial].
- **Step Size:** Using a constant step size ignores the Lipschitz property of the SDF for stepping, potentially leading to aliasing or missed thin features if $s$ is too large [S1][S2, unofficial].

## Subtopics

### [Volumetric Light Shafts (Cone Raymarching)](4565-volumetric-light-shafts-cone-raymarching.md)
Uses a cone-shaped volume and SDFs for light shafts, differing by operating in screen space on pre-computed textures and tapering from light sources.

## Operating characteristics
- **Time Complexity:** $O(N_{\text{max}})$ per ray, where $N_{\text{max}}$ is the maximum number of steps [S2, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary memory, as it is procedural and does not store voxel grids [S1].
- **Performance:** Efficient for real-time rendering of procedural volumes like clouds or smoke [S1][S2, unofficial]. Performance depends heavily on $N_{\text{max}}$ and $s$.
- **Strengths:** No precomputation or voxelization required; watertight traversal; compact representation [S1].
- **Limitations:** Constant step size may cause aliasing in thin features; less efficient than adaptive sphere tracing for surface rendering [S1][S2, unofficial].

## Background
Originating from the adaptation of sphere tracing to volumetric rendering, this method leverages the continuous nature of SDFs to encode density [S1]. It gained popularity through real-time shader communities, notably Shadertoy scenes by Inigo Quilez and others, for rendering procedural clouds and smoke [S2, unofficial].

## Sources
[S1] Volume ray casting — Grokipedia, https://grokipedia.com/page/Volume_ray_casting, fetched 2026-10-26
[S2] Real-time dreamy Cloudscapes with Volumetric Raymarching, https://blog.maximeheckel.com/posts/real-time-cloudscapes-with-volumetric-raymarching/, fetched 2026-10-28 (unofficial)
