# Anti-Aliasing for Splatting

Also called: Anti-Aliased Splatting, Anti-Aliasing Technique for Splatting, Perspective Anti-Aliased Splatting

## Overview
Anti-Aliasing for Splatting is a method designed to prevent sample-rate aliasing artifacts (pre-aliasing) that arise during the resampling phase of volume rendering when the integration grid sampling frequency is insufficient relative to the volume raster's frequency content [S1, unofficial][S2, unofficial]. Unlike standard splatting, which uses fixed kernel sizes in eye space and suffers from aliasing when splats become smaller than the integration grid sampling rate, this method scales the splat size and attenuates energy for distant objects to maintain constant energy per unit area on the view plane [S2, unofficial]. It specifically addresses the divergence of rays in perspective projections where the integration grid becomes coarser than the volume raster [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Volume raster defined by grid spacing $s$, volume raster sampling rate $p = 1/s$, and bandlimit frequency $\omega$ [S2, unofficial]. Camera parameters: distance from eye to screen $D$, pixel extent $p_{pixel}$, and volume raster sample spacing $v$ [S2, unofficial].
*   **Output:** Rendered image with anti-aliased splats.

**Data Structures:**
*   **Splat Primitive:** Defined by position $\mathbf{p}_k$, density $\rho_k$, and a reconstruction kernel. The kernel is typically pre-computed (e.g., optimal cubic spline) and stored in a texture (e.g., $256 \times 256$ table) [S2, unofficial].
*   **Polygon Representation:** Splats are rendered as polygons in world space, rotated to be perpendicular to the eye-to-voxel ray [S2, unofficial].

**Parameters:**
*   $s$: Volume raster grid spacing [S1, unofficial].
*   $p$: Volume raster sampling rate, $p = 1/s$ [S1, unofficial].
*   $\omega$: Bandlimit frequency of the volume raster [S1, unofficial].
*   $D$: Distance from the eye point to the screen [S2, unofficial].
*   $v$: Sample spacing of the volume raster [S2, unofficial].
*   $p_{pixel}$: Extent of a pixel [S2, unofficial].
*   $k$: Critical distance where volume raster and integration grid sampling rates are equal [S1, unofficial].

**Steps:**

1.  **Calculate Critical Distance $k$:**
    Determine the distance along the eye-space z-axis ($z_e$) where the sampling rates of the volume raster and the integration grid are identical. This divides the scene into two regions: Region 1 ($d < k$) and Region 2 ($d \ge k$) [S1, unofficial][S2, unofficial].
    $$ k = \frac{v}{p_{pixel}} \cdot D $$
    Where $v$ is the volume raster sample spacing, $p_{pixel}$ is the pixel extent, and $D$ is the eye-to-screen distance [S2, unofficial].

2.  **Check Volume Raster Condition:**
    Ensure the volume raster itself is sampled above the Nyquist limit to avoid inherent aliasing:
    $$ p > 2\omega $$
    If this condition is not met, aliasing exists in the source data regardless of the splatting method [S1, unofficial][S2, unofficial].

3.  **Iterate Over Splats:**
    For each splat at distance $d$ from the eye:

    *   **Case 1: $d < k$ (Region 1)**
        The integration grid sampling frequency $\phi(d)$ is greater than the volume raster sampling rate $p$ [S1, unofficial]. Since $\phi(d) > p > 2\omega$, no additional filtering is required [S1, unofficial]. Render the splat using standard splatting parameters [S1, unofficial][S2, unofficial].

    *   **Case 2: $d \ge k$ (Region 2)**
        The integration grid sampling frequency $\phi(d)$ is less than $p$ [S1, unofficial]. To avoid aliasing, the splat must be low-pass filtered such that the new highest frequency $\omega(d)$ satisfies:
        $$ \phi(d) > 2\omega(d) $$
        This is achieved by scaling the splat and attenuating its energy [S2, unofficial].

        1.  **Scale Splat Radius:**
            Scale the splat radius $r$ by a factor proportional to the distance [S2, unofficial]. If $r_1$ is the radius at distance $k$, the radius $r_2$ at distance $d$ is:
            $$ r_2 = r_1 \cdot \frac{d}{k} $$
            This scaling narrows the frequency components by a factor of $k/d$ [S2, unofficial].

        2.  **Attenuate Splat Energy:**
            Attenuate the splat's energy (e.g., alpha channel) to maintain constant energy per unit area on the view plane [S2, unofficial]. If $u_1$ is the energy at distance $k$, the energy $u_2$ at distance $d$ is:
            $$ u_2 = u_1 \cdot \left(\frac{k}{d}\right)^2 $$
            This ensures that the integral of the splat function remains consistent with the physical density despite the spatial expansion [S2, unofficial].

        3.  **Render:**
            Draw the scaled polygon with the attenuated alpha channel [S2, unofficial].

4.  **Composite:**
    Composite the semi-transparent splat polygon into the screen buffer [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_anti_aliased_splats(splats, camera_params, r1):
    """
    splats: List of dicts with 'pos' (x,y,z) and 'alpha'.
    camera_params: Dict with 'D' (eye-to-screen distance), 'pixel_extent', 'v' (volume raster spacing).
    r1: Standard splat radius at distance k.
    """
    # 1. Calculate critical distance k
    k = (camera_params['v'] / camera_params['pixel_extent']) * camera_params['D']
    
    for splat in splats:
        x, y, z = splat['pos']
        # 2. Calculate distance d from eye (assumed at origin)
        d = (x**2 + y**2 + z**2)**0.5
        
        if d < k:
            # Case 1: Standard splatting
            radius = r1
            energy = splat['alpha']
        else:
            # Case 2: Anti-aliased splatting
            # Scale radius by d/k
            radius = r1 * (d / k)
            # Attenuate energy by (k/d)^2
            energy = splat['alpha'] * ((k / d) ** 2)
            
        # Compute normal vector for polygon rotation (perpendicular to eye-ray)
        ray_dir = (x/d, y/d, z/d)
        ref = (1, 0, 0) if abs(ray_dir[0]) < 0.9 else (0, 1, 0)
        normal = (ray_dir[1]*ref[2] - ray_dir[2]*ref[1], 
                  ray_dir[2]*ref[0] - ray_dir[0]*ref[2], 
                  ray_dir[0]*ref[1] - ray_dir[1]*ref[0])
        n_len = (normal[0]**2 + normal[1]**2 + normal[2]**2)**0.5
        normal = (normal[0]/n_len, normal[1]/n_len, normal[2]/n_len)
        
        # 3. Render splat as polygon
        draw_polygon(splat['pos'], radius, energy, normal)

def draw_polygon(center, radius, alpha, normal):
    # Hardware rasterization step: draw a polygon centered at center,
    # oriented perpendicular to normal, textured with the pre-computed kernel,
    # and composited into the screen buffer with alpha = alpha.
    pass
```

**Edge Cases and Pitfalls:**
*   **Off-Axis Error:** The calculation of splat size using similar triangles (Equation 4 in source) underestimates the required low-pass filtering for splats not on the +Z axis [S2, unofficial]. At the view cone boundary (e.g., 30 degrees), the kernel is ~1.15 times larger than strictly necessary [S2, unofficial]. However, this results in *more* filtering than required, satisfying the Nyquist condition and avoiding aliasing [S2, unofficial].
*   **Temporal Aliasing:** This method addresses spatial pre-aliasing but does not inherently solve temporal aliasing (flickering/strobing) during animation, although it significantly reduces it by smoothing the spatial signal [S2, unofficial].
*   **Orthographic Projection:** For orthographic projections, $\phi(d)$ is constant [S1, unofficial]. If $p > 2\omega$ holds, no anti-aliasing is needed [S1, unofficial]. If not, the method can be adapted, but the divergence-based scaling is specific to perspective [S1, unofficial][S2, unofficial].

## Subtopics

### Magnification-based Anti-Aliasing
A variant used in holographic wave field rendering where anti-aliasing is achieved by magnifying the wave field by a factor of two in both dimensions and applying a fill factor of 0.93 [S3]. This approach differs from the geometric scaling of splats by operating in the frequency domain via grid interpolation [S3].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of splats [G].
*   **Memory Complexity:** $O(N)$ for storing splat data [G]. Requires a pre-computed kernel texture (e.g., $256 \times 256$) [S2, unofficial].
*   **Performance:** In tests, a 498x398x1 volume (198K splats) rendered into a 270x157 image showed strong aliasing in standard splatting, which was smoothed out by this method [S2, unofficial]. A 512x512x103 terrain volume (386K splats) showed elimination of flickering and strobing [S2, unofficial]. A 420x468x62 microtubule volume (103K splats) rendered into a 270x270 image also demonstrated the technique's applicability [S2, unofficial].
*   **Strengths:** Effectively eliminates pre-aliasing artifacts in perspective views where splats become smaller than pixels [S2, unofficial]. Maintains energy conservation.
*   **Limitations:** Introduces increased blur for distant objects due to the larger kernel size. Does not address post-aliasing from non-ideal reconstruction filters or temporal aliasing directly [S1, unofficial][S2, unofficial].
*   **When to use:** Perspective volume rendering where high-frequency details in the volume raster exceed the integration grid's sampling capacity at distance [S1, unofficial][S2, unofficial].
*   **When not to use:** Orthographic rendering where grid sampling is uniform and sufficient, or when real-time performance is critical and blur is unacceptable [S1, unofficial][S2, unofficial].

## Background
The method was introduced in the paper "An Anti-Aliasing Technique for Splatting" by Westover, presented at IEEE Visualization 1997, where it was awarded Best Paper [S2, unofficial]. It addresses a fundamental limitation of splatting algorithms, which perform reconstruction in perspective space, unlike ray-casting algorithms that perform reconstruction in eye space and can naturally adjust kernel sizes as rays diverge [S1, unofficial][S2, unofficial]. The technique was implemented using hardware acceleration, rendering splats as textured polygons [S2, unofficial].

## Sources
[S1] DTIC ADA363073: Perspective Splatting, https://archive.org/details/DTIC_ADA363073, fetched 2026-10-26 (unofficial)
[S2] DTIC ADA606342: An Anti-Aliasing Technique for Splatting, https://archive.org/details/DTIC_ADA606342, fetched 2026-10-26 (unofficial)
[S3] Mesh-represented and learning-empowered hologram synthesis for full 3D holographic displays, https://www.nature.com/articles/s41467-026-74557-0, fetched 2026-10-30
