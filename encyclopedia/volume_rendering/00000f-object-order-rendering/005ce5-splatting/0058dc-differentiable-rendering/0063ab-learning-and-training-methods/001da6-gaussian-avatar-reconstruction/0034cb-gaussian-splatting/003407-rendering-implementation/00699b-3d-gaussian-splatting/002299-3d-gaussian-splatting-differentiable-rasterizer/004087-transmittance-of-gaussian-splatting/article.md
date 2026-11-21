# Transmittance of Gaussian Splatting

Also called: Proposition 2, Ray-marching equivalence of Gaussian Splatting

## Overview
This method establishes that the image formation model of Gaussian Splatting (alpha-blending) is equivalent to a ray-marching volumetric rendering model under assumptions of statistical independence and non-overlapping Gaussian primitives [S1]. It defines a continuous transmittance function and a view-dependent attenuation coefficient that yield the same color output as the standard discrete alpha-compositing rasterizer [S1].

## How it works
**Inputs:**
- Ray origin $\mathbf{o} \in \mathbb{R}^3$ (in empty space) [S1].
- Ray direction $\mathbf{w} \in \mathbb{S}^2$ [S1].
- Distance parameter $t \in [0, +\infty)$ [S1].
- Set of $N$ 3D Gaussians $\{G_i\}_{i=1}^N$, where each $G_i(\mathbf{x})$ is the Gaussian density function at position $\mathbf{x}$ [S1].
- View-dependent colors $\mathbf{c}_i(\mathbf{w})$ for each Gaussian [S1].

**Outputs:**
- Total transmittance $T_{\mathbf{o}, \mathbf{w}}(t)$ [S1].
- Rendered color $C_{\mathbf{o}, \mathbf{w}}$ [S1].

**Data Structures:**
- The Gaussian primitives are stored as a list or array [S1].
- Each primitive $i$ contains parameters defining $G_i(\mathbf{x})$ (mean, covariance) and color $\mathbf{c}_i$ [S1].
- No explicit volumetric grid is required; the attenuation is computed analytically from the Gaussian parameters [S1].

**Assumptions:**
1. **Statistical Independence:** The presence of one Gaussian does not affect the transmission properties of another [S1]. This follows the RayGauss assumption [S1].
2. **Non-overlapping Primitives:** In the 3D domain, Gaussians do not significantly overlap [S1]. This is an approximation similar to the sorting assumption in alpha blending [S1].

**Formulas:**

1. **Capped Gaussian Value:**
   Let $t_{\mathbf{o}, \mathbf{w}}^{G_i}$ be the distance along the ray where the Gaussian $G_i$ reaches its maximum contribution [S1]. The capped Gaussian value along the ray is:
   $$ \bar{G}_{\mathbf{o}, \mathbf{w}}^{(i)}(t) = G_i(\mathbf{o} + \min(t, t_{\mathbf{o}, \mathbf{w}}^{G_i})\mathbf{w}) $$
   [S1]

2. **Continuous Transmittance Function:**
   The total transmittance from the origin to distance $t$ is the product of the individual transmittances of all $N$ Gaussians:
   $$ T_{\mathbf{o}, \mathbf{w}}(t) = \prod_{i=1}^N \left(1 - \bar{G}_{\mathbf{o}, \mathbf{w}}^{(i)}(t)\right) $$
   [S1]

3. **View-Dependent Attenuation Coefficient:**
   The attenuation coefficient $\sigma(\mathbf{x}, \mathbf{w})$ at position $\mathbf{x}$ for direction $\mathbf{w}$ is derived from the transmittance:
   $$ \sigma(\mathbf{x}, \mathbf{w}) = \sum_{i=1}^N \max(0, -\mathbf{w} \cdot \nabla \log(1 - G_i(\mathbf{x}))) $$
   [S1]

4. **Rendered Color:**
   Order $M$ Gaussians with positive projection along the ray by increasing distance from $\mathbf{o}$ [S1]. Define intervals $t_0=0 < t_1 < \dots < t_M=+\infty$ such that within $[t_{i-1}, t_i]$, only Gaussian $i$ contributes significantly and previous Gaussians are fully attenuated [S1]. The color is:
   $$ C_{\mathbf{o}, \mathbf{w}} = \sum_{i=1}^M \mathbf{c}_i(\mathbf{w}) \bar{G}_{\mathbf{o}, \mathbf{w}}^{(i)}(t_{\mathbf{o}, \mathbf{w}}^{G_i}) \prod_{j=1}^{i-1} \left( 1 - \bar{G}_{\mathbf{o}, \mathbf{w}}^{(j)}(t_{\mathbf{o}, \mathbf{w}}^{G_j}) \right) $$
   [S1]
   This matches the standard alpha-blending formula [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dot(a, b):
    return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]

def render_gaussian_splatting_transmittance(ray_origin, ray_direction, gaussians):
    """
    Computes the transmittance and color for a ray using the 
    Transmittance of Gaussian Splatting equivalence.
    
    Args:
        ray_origin: 3D vector (o)
        ray_direction: 3D vector (w), normalized
        gaussians: List of Gaussian objects, each with:
            - center: 3D vector
            - evaluate(x): returns G_i(x)
            - max_contribution_dist(o, w): returns t_{o,w}^{G_i}
            - color(w): returns c_i(w)
    
    Returns:
        transmittance: float T(o, w, infinity)
        color: 3D vector C(o, w)
    """
    # 1. Filter and sort Gaussians by positive projection along the ray
    valid_gaussians = []
    for g in gaussians:
        vec_to_center = [g.center[0]-ray_origin[0], g.center[1]-ray_origin[1], g.center[2]-ray_origin[2]]
        proj_dist = dot(vec_to_center, ray_direction)
        if proj_dist > 0:
            valid_gaussians.append(g)
            
    valid_gaussians.sort(key=lambda g: dot([g.center[0]-ray_origin[0], g.center[1]-ray_origin[1], g.center[2]-ray_origin[2]], ray_direction))
    M = len(valid_gaussians)
    
    accumulated_transmittance = 1.0
    color = [0.0, 0.0, 0.0]
    
    for i in range(M):
        g = valid_gaussians[i]
        t_max = g.max_contribution_dist(ray_origin, ray_direction)
        
        point = [ray_origin[0]+ray_direction[0]*t_max, ray_origin[1]+ray_direction[1]*t_max, ray_origin[2]+ray_direction[2]*t_max]
        g_val = g.evaluate(point)
        
        c_i = g.color(ray_direction)
        color[0] += c_i[0] * g_val * accumulated_transmittance
        color[1] += c_i[1] * g_val * accumulated_transmittance
        color[2] += c_i[2] * g_val * accumulated_transmittance
        
        accumulated_transmittance *= (1.0 - g_val)
        
    return accumulated_transmittance, color
```

**Edge Cases and Pitfalls:**
- **Non-Reciprocity:** The attenuation coefficient is not reciprocal ($\sigma(\mathbf{x}, \mathbf{w}) \neq \sigma(\mathbf{x}, -\mathbf{w})$) because each Gaussian contributes to attenuation only before the ray reaches its point of maximum contribution [S1].
- **Overlap Assumption:** The equivalence relies on non-overlapping Gaussians; in practice, this is an approximation similar to alpha blending sorting [S1].
- **Numerical Stability:** If $G_i(\mathbf{x}) \approx 1$, the term $\log(1 - G_i(\mathbf{x}))$ approaches $-\infty$, requiring clamping or careful handling [G].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Strengths and Limitations:** Not given in the sources.
- **When to Use:** Not given in the sources.

## Background
This result is presented as "Proposition 2" in the paper "From Blobs to Spokes: High-Fidelity Surface Reconstruction via Oriented Gaussians" [S1]. It builds on the observation that alpha blending approximates volumetric rendering by sorting and compositing [S1]. The statistical independence assumption follows from RayGauss [S1]. The non-overlapping assumption is noted as an approximation similar to alpha blending sorting [S1].

## Sources
[S1] From Blobs to Spokes: High-Fidelity Surface Reconstruction via Oriented Gaussians, https://doi.org/10.1007/978-3-032-37095-2_4, fetched 2026-10-29
