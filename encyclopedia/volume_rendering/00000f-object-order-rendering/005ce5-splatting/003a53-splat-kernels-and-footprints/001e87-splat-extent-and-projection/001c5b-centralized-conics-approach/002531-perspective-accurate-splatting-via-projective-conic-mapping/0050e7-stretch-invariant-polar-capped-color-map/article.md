# Stretch-invariant polar capped color map

Also called: Stretch-invariant polar capped map, plane-chart cylindrical projection with azimuthal equidistant polar caps, polar capped color map, polar capped map, spherical representation P

## Overview
The stretch-invariant polar capped color map is a texture mapping technique that encodes point cloud color data onto a 2D image while maintaining uniform sample density across the unit sphere, solving the non-uniform density flaw of standard cylindrical plane-chart projections [S1]. It generates one texture map per scan location to store color information for textured splats while preserving occlusion equivalence with the original point cloud [S1].

## How it works
The method maps texture coordinates $(s, t)$ to a 3D direction vector $(x, y, z)$ on the unit sphere using a hybrid projection: a plane-chart cylindrical projection for the equatorial band and azimuthal equidistant projections for the polar caps [S1]. Texture coordinates $(s, t)$ are assumed to lie in $[0, 1]$ [G]. The geometry (point positions) is stored separately as regular $(x, y, z)$ data to avoid precision loss from encoding depth in image formats like PNG, while the texture map stores only color data [S1].

**Inputs:**
- Forward: Texture coordinates $(s, t)$ [S1].
- Inverse: Normalized direction vector $(\bar{x}, \bar{y}, \bar{z})$ from scan location $s_k$ to point $p$ [S1].

**Outputs:**
- Forward: Unit direction vector $(x, y, z)$ [S1].
- Inverse: Texture coordinates $(s, t)$ [S1].

**Formulas and Steps:**
The sphere is divided into three regions based on the latitude angle $\psi$ [S1].

1. **Bottom Cap ($\psi \in [-\pi/2, -\pi/4)$):** Corresponds to texture coordinate $s \le 1/6$ [S1].
   *Forward Mapping (Texel to Direction):*
   Let $u_b = 6s - 0.5$, $v_b = t - 0.5$, and $r_b = u_b^2 + v_b^2$ [S1].
   $$
   \begin{aligned}
   x &= \frac{u_b}{r_b} \sin\left(\frac{\pi r_b}{2}\right) \\
   y &= \frac{v_b}{r_b} \sin\left(\frac{\pi r_b}{2}\right) \\
   z &= -\cos\left(\frac{\pi r_b}{2}\right)
   \end{aligned}
   $$
   *Inverse Mapping (Direction to Texel):*
   Let $r = 2\pi \arccos(\bar{z})$ [S1].
   $$
   \begin{aligned}
   s &= \frac{\frac{\bar{x} r}{\sin(\pi r / 2)} + 0.5}{6} \\
   t &= \frac{\frac{\bar{y} r}{\sin(\pi r / 2)} + 0.5}{6}
   \end{aligned}
   $$

2. **Top Cap ($\psi \in (\pi/4, \pi/2]$):** Corresponds to texture coordinate $1/6 < s \le 2/6$ [S1].
   *Forward Mapping (Texel to Direction):*
   Let $u_t = 6s - 1.5$, $v_t = t - 0.5$, and $r_t = \sqrt{u_t^2 + v_t^2}$ [S1].
   $$
   \begin{aligned}
   x &= \frac{u_t}{r_t} \sin\left(\frac{\pi r_t}{2}\right) \\
   y &= \frac{v_t}{r_t} \sin\left(\frac{\pi r_t}{2}\right) \\
   z &= \cos\left(\frac{\pi r_t}{2}\right)
   \end{aligned}
   $$
   *Inverse Mapping (Direction to Texel):*
   Let $r = 2\pi \arccos(\bar{z})$ [S1].
   $$
   \begin{aligned}
   s &= \frac{\frac{\bar{x} r}{\sin(\pi r / 2)} + 1.5}{6} \\
   t &= \frac{\frac{\bar{y} r}{\sin(\pi r / 2)} + 0.5}{6}
   \end{aligned}
   $$

3. **Equatorial Band ($\psi \in [-\pi/4, \pi/4]$):** Corresponds to texture coordinate $s > 2/6$ [S1].
   *Forward Mapping (Texel to Direction):*
   Let $\psi = t \frac{\pi}{2} - \frac{\pi}{4}$ and $\theta = (1.5s - 0.5)2\pi$ [S1].
   $$
   \begin{aligned}
   x &= \cos(\psi) \cos(\theta) \\
   y &= \cos(\psi) \sin(\theta) \\
   z &= \sin(\psi)
   \end{aligned}
   $$
   *Inverse Mapping (Direction to Texel):*
   Let $\psi = \arcsin(\bar{z})$ and $\theta = \operatorname{atan2}(\bar{y}, \bar{x})$ [S1].
   $$
   \begin{aligned}
   s &= \frac{\frac{\theta}{2\pi} + 0.5}{1.5} \\
   t &= \frac{\psi + \frac{\pi}{4}}{2\pi}
   \end{aligned}
   $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def forward_map(s: float, t: float) -> tuple:
    """Maps texture coordinates (s, t) to a 3D direction vector (x, y, z) [S1]."""
    if s <= 1.0 / 6.0:
        # Bottom Cap [S1]
        ub = 6.0 * s - 0.5
        vb = t - 0.5
        rb = ub * ub + vb * vb
        
        # [G] Guard against division by zero at exact pole center
        if rb == 0.0:
            return (0.0, 0.0, -1.0)
            
        sin_val = math.sin(math.pi * rb / 2.0)
        x = (ub / rb) * sin_val
        y = (vb / rb) * sin_val
        z = -math.cos(math.pi * rb / 2.0)
        return (x, y, z)
        
    elif s <= 2.0 / 6.0:
        # Top Cap [S1]
        ut = 6.0 * s - 1.5
        vt = t - 0.5
        rt = math.sqrt(ut * ut + vt * vt)
        
        # [G] Guard against division by zero at exact pole center
        if rt == 0.0:
            return (0.0, 0.0, 1.0)
            
        sin_val = math.sin(math.pi * rt / 2.0)
        x = (ut / rt) * sin_val
        y = (vt / rt) * sin_val
        z = math.cos(math.pi * rt / 2.0)
        return (x, y, z)
        
    else:
        # Equatorial Band [S1]
        psi = t * (math.pi / 2.0) - (math.pi / 4.0)
        theta = (1.5 * s - 0.5) * 2.0 * math.pi
        
        x = math.cos(psi) * math.cos(theta)
        y = math.cos(psi) * math.sin(theta)
        z = math.sin(psi)
        return (x, y, z)

def inverse_map(x_bar: float, y_bar: float, z_bar: float) -> tuple:
    """Maps a normalized direction vector (x_bar, y_bar, z_bar) to texture coordinates (s, t) [S1]."""
    # [G] Normalize input to handle unnormalized vectors
    length = math.sqrt(x_bar**2 + y_bar**2 + z_bar**2)
    if length == 0.0:
        return (0.0, 0.0)
    x_bar /= length
    y_bar /= length
    z_bar /= length
    
    psi = math.asin(z_bar)
    
    if psi <= -math.pi / 4.0:
        # Bottom Cap [S1]
        r = 2.0 * math.pi * math.acos(z_bar)
        sin_val = math.sin(math.pi * r / 2.0)
        
        # [G] Guard against division by zero at exact poles
        if sin_val == 0.0:
            return (0.5 / 6.0, 0.5 / 6.0)
            
        s = ((x_bar * r / sin_val) + 0.5) / 6.0
        t = ((y_bar * r / sin_val) + 0.5) / 6.0
        return (s, t)
        
    elif psi > math.pi / 4.0:
        # Top Cap [S1]
        r = 2.0 * math.pi * math.acos(z_bar)
        sin_val = math.sin(math.pi * r / 2.0)
        
        if sin_val == 0.0:
            return (1.5 / 6.0, 0.5 / 6.0)
            
        s = ((x_bar * r / sin_val) + 1.5) / 6.0
        t = ((y_bar * r / sin_val) + 0.5) / 6.0
        return (s, t)
        
    else:
        # Equatorial Band [S1]
        theta = math.atan2(y_bar, x_bar)
        s = (theta / (2.0 * math.pi) + 0.5) / 1.5
        t = (psi + math.pi / 4.0) / (2.0 * math.pi)
        return (s, t)
```

**Parameters:**
- $s, t$: Texture coordinates, typically in $[0, 1]$ [G].
- $\psi$: Latitude angle, range $[-\pi/2, \pi/2]$ [S1].
- $\theta$: Longitude angle, range $[-\pi, \pi]$ [G].

**Edge Cases and Pitfalls:**
- Division by zero occurs in the cap formulas when $r_b=0$ or $r_t=0$ (exact pole centers) or when $\sin(\pi r / 2) = 0$ (poles) [G]. The pseudocode handles these centers explicitly.
- Precision loss is avoided by storing geometry as $(x,y,z)$ rather than encoding depth in the texture, as noted in the source [S1].
- The equatorial band formula for $s$ involves division by $1.5$; numerical error may push $s$ slightly outside $[2/6, 1]$, requiring standard clamping or wrapping in practice [G].

## Operating characteristics
- **Time Complexity:** The mapping operations are $O(1)$ per point/texel, involving basic trigonometric functions and arithmetic [G].
- **Memory Complexity:** Requires one texture map per scan location $s_k$ [S1]. Using texture maps for color data greatly reduces disk memory footprint compared to storing all color data in point structures, and allows for lossy image compression [S1].
- **Strengths:** Minimizes distortion across every region of the unit sphere and achieves more uniform sample density compared to standard cylindrical plane-chart projections [S1]. Maintains occlusion equivalence with the original point cloud [S1].
- **When to use:** When rendering point clouds with textured splats and uniform sampling density across the sphere is critical for visual quality or algorithmic consistency [S1].
- **When not to use:** Not given in the sources.

## Background
The method was proposed by M. Comino, A. Chica, and C. Andujar in 2019 [S1]. It addresses a flaw in the cylindrical plane-chart projection used by Comino et al. in 2017, which resulted in non-uniform sample density across the unit sphere [S1]. The approach follows Snyder et al. for the mathematical formulation of the stretch-invariant projection [S1]. It is used in the context of view-dependent hierarchical rendering of massive point clouds through textured splats [S1].

## Sources
[S1] View-dependent Hierarchical Rendering of Massive Point Clouds through Textured Splats, https://doi.org/10.2312/ceig.20191203, fetched 2026-10-26
