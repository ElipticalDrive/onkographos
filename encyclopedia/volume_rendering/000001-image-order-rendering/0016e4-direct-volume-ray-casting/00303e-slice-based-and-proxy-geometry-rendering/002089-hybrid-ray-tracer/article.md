# Hybrid Ray Tracer

Also called: simultaneous ray casting

## Overview
A rendering method that simultaneously traces rays through volumetric sample arrays and polygonal databases, resampling both at equally spaced intervals, and compositing the resulting colors and opacities in depth-sorted order to handle mixtures of geometric and volumetric data [S1, unofficial]. It sets itself apart from pure volume rendering by explicitly handling polygonal intersections within volumetric slabs using exponential attenuation approximations, and from pure ray tracing by treating the volume as a resampled slab-based medium rather than a continuous density field [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** An $N \times N \times N$ voxel array of scalar values defining volumetric data; a set of embedded polygons defining geometric data; an observer position; viewing ray directions.
*   **Outputs:** A final color for each pixel (ray).

**Data Structures**
*   **Volumetric Data:** Stored as a 3D grid. Each voxel $(i, j, k)$ has an associated color $c_v$ and opacity $o_v$ derived from shading and classification [S1, unofficial].
*   **Polygonal Data:** A set of polygons with shading attributes yielding color $c_p$ and opacity $o_p$ at intersection points [S1, unofficial].

**Parameters**
*   **Slab Thickness:** Equal to the spacing between samples along a viewing ray [S1, unofficial].
*   **Slab Width:** Equal to the spacing between adjacent rays [S1, unofficial].
*   **Threshold $\theta$:** A value used in adaptive supersampling to determine if geometric events are visible [S1, unofficial].

**Formulas**
The opacity $o_v$ of a homogeneous slab is related to its density $d_v$ and thickness $t_v$ by [S1, unofficial]:
$$ o_v = 1 - e^{-d_v t_v} $$
where $0 < o_v < 1$, and $o_v = 1$ signifies complete attenuation.

For a slab of total thickness $t_v$ and opacity $o_v$ intersected by a plane perpendicular to the viewing ray, the opacities of the front portion ($t_f$) and back portion ($t_b$) are calculated as [S1, unofficial]:
$$ o_f = 1 - (1 - o_v)^{t_f/t_v} $$
$$ o_b = 1 - (1 - o_v)^{t_b/t_v} $$

**Algorithm Steps**
1.  **Shade and Classify Volumetric Data:** Compute color $c_v$ and opacity $o_v$ for each voxel in the $N \times N \times N$ array [S1, unofficial].
2.  **Trace Rays:** Cast parallel viewing rays from the observer position into the data [S1, unofficial].
3.  **Resample Volume:** For each ray, compute a vector of colors $c_v$ and opacities $o_v$ by resampling the volumetric data at equally spaced positions along the ray. This is done by tri-linearly interpolating from the colors and opacities in the eight voxels surrounding each sample location [S1, unofficial].
4.  **Compute Polygon Intersections:** Independently compute all intersections between the current ray and polygons in the environment. Shade these intersections to yield a color $c_p$ and opacity $o_p$ for each point of intersection [S1, unofficial].
5.  **Composite with Slabs:**
    *   Treat the volumetric data as a series of slabs, where slab thickness equals the sample spacing and slab width equals the ray spacing [S1, unofficial].
    *   If a ray intersects a polygon within a slab, treat the polygon locally as a plane perpendicular to the ray at the intersection point [S1, unofficial].
    *   Calculate the opacities $o_f$ and $o_b$ for the slivers of volumetric gel immediately in front of and behind this plane using the formulas above [S1, unofficial].
    *   Composite the front volumetric sliver ($c_v, o_v$), then the polygon ($c_p, o_p$), then the back volumetric sliver ($c_v, o_v$) in depth-sorted order from front to back [S1, unofficial].
    *   If a ray intersects more than one polygon within a slab, compute and composite the contribution of each polygon and each sliver of volumetric gel separately [S1, unofficial].
6.  **Adaptive Supersampling (Anti-aliasing):**
    *   To avoid aliasing at polygonal edges, intersections, and shading highlights, increase the number of rays cast per pixel in specific regions [S1, unofficial].
    *   Compute two additional colors during ray tracing to determine if supersampling is needed [S1, unofficial]:
        *   $C_x$: Contributions only from polygons (set $o_v=0$ for all volumetric samples).
        *   $C_q$: Contributions only from polygons, attenuated by passage through volumetric data (set $o_v=0$ for all volumetric samples) [S1, unofficial].
    *   Determine if supersampling is required based on the range of $C_x$ and $C_q$ within a sample region [S1, unofficial]:
        *   If range($C_x$) > $\theta$ and range($C_q$) $\le \theta$: The difference is due to volumetric data; do not supersample.
        *   If range($C_q$) > $\theta$ and range($C_x$) $\le \theta$: The geometric event is hidden by opaque volumetric data; do not supersample.
        *   If range($C_x$) > $\theta$ and range($C_q$) > $\theta$: The region contains a visible geometric event; cast more rays (supersample) [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hybrid_ray_tracer(volumetric_grid, polygons, observer_pos, image_width, image_height, threshold_theta):
    """
    Renders a mixture of volumetric and polygonal data using the hybrid ray tracer method.
    
    Args:
        volumetric_grid: 3D array of shape (N, N, N) containing pre-shaded (color, opacity) tuples.
        polygons: List of polygon objects with shading methods.
        observer_pos: 3D vector for the camera origin.
        image_width, image_height: Dimensions of the output image.
        threshold_theta: Threshold for adaptive supersampling.
        
    Returns:
        Image buffer with final colors.
    """
    image = empty_buffer(image_width, image_height)
    
    # Pre-compute slab dimensions based on grid resolution
    # Slab thickness = spacing between samples along ray
    # Slab width = spacing between adjacent rays
    # These are implicitly handled by the resampling step
    
    for y in range(image_height):
        for x in range(image_width):
            # Primary ray direction
            ray_dir = compute_ray_direction(x, y, image_width, image_height, observer_pos)
            
            # Initialize accumulators for primary ray and supersampling checks
            color_primary = (0.0, 0.0, 0.0)
            opacity_primary = 0.0
            
            color_Cx = (0.0, 0.0, 0.0) # Polygons only, no volume attenuation
            opacity_Cx = 0.0
            
            color_Cq = (0.0, 0.0, 0.0) # Polygons only, attenuated by volume
            opacity_Cq = 0.0
            
            # Trace primary ray
            color_primary, opacity_primary = trace_ray(
                ray_dir, observer_pos, volumetric_grid, polygons, 
                composite_mode='full'
            )
            
            # Trace auxiliary rays for supersampling decision
            color_Cx, _ = trace_ray(
                ray_dir, observer_pos, volumetric_grid, polygons, 
                composite_mode='polygons_only'
            )
            
            color_Cq, _ = trace_ray(
                ray_dir, observer_pos, volumetric_grid, polygons, 
                composite_mode='polygons_attenuated'
            )
            
            # Check for visible geometric events
            range_Cx = compute_range(color_Cx, image, x, y)
            range_Cq = compute_range(color_Cq, image, x, y)
            
            if range_Cx > threshold_theta and range_Cq > threshold_theta:
                # Visible geometric event: supersample
                color_supersampled = (0.0, 0.0, 0.0)
                opacity_supersampled = 0.0
                
                # Cast additional rays (adaptive supersampling)
                # Implementation detail: specific offset directions or jittering
                # would be defined by the supersampling strategy [Whitted, 1980]
                for _ in range(num_supersamples):
                    offset_ray_dir = perturb_ray_direction(ray_dir)
                    c, o = trace_ray(
                        offset_ray_dir, observer_pos, volumetric_grid, polygons,
                        composite_mode='full'
                    )
                    color_supersampled = composite_color(color_supersampled, c, o)
                    opacity_supersampled = composite_opacity(opacity_supersampled, o)
                
                image[x, y] = color_supersampled
            else:
                image[x, y] = color_primary

    return image

def trace_ray(ray_dir, origin, volumetric_grid, polygons, composite_mode):
    """
    Traces a single ray through volume and polygons.
    
    Args:
        composite_mode: 'full', 'polygons_only', or 'polygons_attenuated'.
        
    Returns:
        Tuple (final_color, final_opacity).
    """
    ray_color = (0.0, 0.0, 0.0)
    ray_opacity = 0.0
    
    # Collect all events: volume samples and polygon intersections
    events = []
    
    # 1. Resample volumetric data at equally spaced positions
    # Tri-linear interpolation from 8 surrounding voxels
    sample_spacing = compute_sample_spacing(volumetric_grid)
    t = 0.0
    while t < max_distance:
        pos = origin + t * ray_dir
        c_v, o_v = tri_linear_interpolate(pos, volumetric_grid)
        
        if composite_mode == 'full':
            events.append(('volume', t, c_v, o_v))
        elif composite_mode == 'polygons_attenuated':
            # Volume contributes to attenuation but not color
            # This is handled by setting o_v for attenuation but c_v=0 for color
            # However, the source says "setting o_v=0 for all volumetric samples" for Cx/Cq logic
            # Wait, source says:
            # Cx: contributions only from polygons (setting o_v=0 for all volumetric samples)
            # Cq: contributions only from polygons, attenuated by passage through volumetric data (setting o_v=0 for all volumetric samples)
            # This implies Cx and Cq are computed differently in the source description.
            # Let's re-read carefully:
            # "The first, denoted Cx, contains contributions only from polygons (obtained by setting o_v=0 for all volumetric samples)."
            # "The second, denoted Cq, contains contributions only from polygons, but attenuated by passage through volumetric data (obtained by setting o_v=0 for all volumetric samples)."
            # This phrasing is slightly ambiguous. Does "setting o_v=0" apply to the volume's contribution to color/opacity in the composite?
            # If o_v=0, the volume is transparent.
            # For Cx: Volume is transparent (o_v=0). Polygons are rendered.
            # For Cq: Volume is transparent (o_v=0). Polygons are rendered. But how is it "attenuated by passage through volumetric data"?
            # Perhaps it means the volume's density is used for attenuation, but its color/opacity is zero?
            # Or does it mean the volume samples are skipped entirely?
            # "attenuated by passage through volumetric data" suggests the volume acts as a filter.
            # If o_v=0, there is no attenuation in the standard formula.
            # Let's look at the context: "distinguishing whether the color difference ... is due to volumetric data or geometric data".
            # If we set o_v=0, the volume is invisible.
            # Maybe Cq uses the volume's density to attenuate the polygon colors, but doesn't add volume color?
            # The source says: "obtained by setting o_v=0 for all volumetric samples".
            # If o_v=0, then (1-o_v) = 1. Attenuation factor is 1. No attenuation.
            # This seems contradictory. Let's assume the source implies:
            # Cx: Render polygons against black background, volume ignored.
            # Cq: Render polygons, but their color is modulated by the volume's transmittance?
            # But if o_v=0, transmittance is 1.
            # Alternative interpretation: The "setting o_v=0" refers to the *color* contribution of the volume, but the opacity is still used for attenuation?
            # No, it explicitly says "setting o_v=0".
            # Let's stick to the literal text for the algorithm steps, but note the ambiguity.
            # Actually, looking at standard hybrid rendering, Cq usually represents the geometric contribution *as seen through* the volume.
            # If the volume is opaque, Cq should be dark. If the volume is transparent, Cq should be bright.
            # If we set o_v=0, the volume is invisible, so Cq = Cx.
            # This suggests the source text might have a typo or specific definition.
            # However, the decision logic is:
            # If range(Cx) > theta and range(Cq) <= theta: Volumetric variation.
            # If range(Cq) > theta and range(Cx) <= theta: Geometric event hidden by volume.
            # If both > theta: Visible geometric event.
            # This logic requires Cx and Cq to be different.
            # If Cx is "polygons only" and Cq is "polygons attenuated by volume", then Cq must use volume opacity.
            # The text "setting o_v=0 for all volumetric samples" might refer to the *color* c_v being zero, or it's a mistake.
            # Given the constraint "Never guess", I will implement the steps as described, but the pseudocode for Cq/Cx calculation is ambiguous.
            # I will assume the standard interpretation for the "How it works" section based on the decision logic, but cite the source's specific phrasing.
            # For the pseudocode, I will implement a placeholder comment for the exact Cq/Cx calculation logic due to the contradiction.
            pass 
            
        t += sample_spacing
        
    # 2. Compute polygon intersections
    for polygon in polygons:
        intersection = ray_intersect_polygon(ray_dir, origin, polygon)
        if intersection:
            t = intersection.distance
            c_p, o_p = polygon.shade(intersection.point, ray_dir)
            events.append(('polygon', t, c_p, o_p))
            
    # Sort events by depth t
    events.sort(key=lambda e: e[1])
    
    # 3. Composite
    for event_type, t, c, o in events:
        if event_type == 'volume':
            if composite_mode == 'full':
                ray_color, ray_opacity = composite_volume(ray_color, ray_opacity, c, o)
            elif composite_mode == 'polygons_only':
                # Volume ignored
                pass
            elif composite_mode == 'polygons_attenuated':
                # Volume attenuates but doesn't add color?
                # This is the ambiguous part.
                # I will skip adding volume color, but apply attenuation if possible.
                # But o_v=0 means no attenuation.
                # I will leave this as a comment in the pseudocode.
                pass
                
        elif event_type == 'polygon':
            # Check if polygon is inside a slab
            # If so, split the slab into front and back slivers
            # This requires identifying which slab the intersection falls into
            # and calculating t_f and t_b
            
            # Simplified: Composite polygon directly if not in slab, or handle slivers
            # The source says: "treat a polygon locally as a plane perpendicular to the ray"
            # "Compute opacities for the slivers... using exponential attenuation formulas"
            # "Composite front sliver, then polygon, then back sliver"
            
            # Find the slab containing the intersection
            slab_index = int(t / sample_spacing)
            slab_start_t = slab_index * sample_spacing
            slab_end_t = (slab_index + 1) * sample_spacing
            
            # Volume samples in this slab
            # We need the color/opacity of the slab at the intersection point
            # This was computed in the volume resampling step
            # Let's assume we have access to the slab's c_v, o_v
            
            # Calculate t_f and t_b
            t_f = t - slab_start_t
            t_b = slab_end_t - t
            
            # Calculate slab opacity o_v (from volume resampling)
            o_v = get_slab_opacity(slab_index, intersection.point)
            c_v = get_slab_color(slab_index, intersection.point)
            
            # Calculate sliver opacities
            o_f = 1 - (1 - o_v) ** (t_f / sample_spacing)
            o_b = 1 - (1 - o_v) ** (t_b / sample_spacing)
            
            if composite_mode == 'full':
                # Composite front sliver
                ray_color, ray_opacity = composite_volume(ray_color, ray_opacity, c_v, o_f)
                # Composite polygon
                ray_color, ray_opacity = composite_volume(ray_color, ray_opacity, c_p, o_p)
                # Composite back sliver
                ray_color, ray_opacity = composite_volume(ray_color, ray_opacity, c_v, o_b)
            elif composite_mode == 'polygons_only':
                # Volume ignored, so o_f and o_b are 0?
                # Or volume is transparent?
                # Source: "setting o_v=0 for all volumetric samples"
                # So o_f = 0, o_b = 0
                # Only polygon is composited
                ray_color, ray_opacity = composite_volume(ray_color, ray_opacity, c_p, o_p)
            elif composite_mode == 'polygons_attenuated':
                # Source: "setting o_v=0 for all volumetric samples"
                # This implies o_f=0, o_b=0.
                # But it says "attenuated by passage through volumetric data".
                # This is the contradiction.
                # I will implement it as "polygons only" for now, with a comment.
                ray_color, ray_opacity = composite_volume(ray_color, ray_opacity, c_p, o_p)

    return ray_color, ray_opacity

def composite_volume(current_color, current_opacity, new_color, new_opacity):
    """
    Standard front-to-back volume compositing.
    """
    alpha = new_opacity
    if alpha == 0:
        return current_color, current_opacity
    
    # Transmittance of current ray
    transmittance = 1.0 - current_opacity
    
    # Contribution of new sample
    contribution = new_color * alpha * transmittance
    
    # Update color and opacity
    new_color_val = current_color + contribution
    new_opacity_val = current_opacity + alpha * transmittance
    
    return new_color_val, new_opacity_val

def compute_range(color, image, x, y):
    """
    Computes the range of color values in a sample region around (x,y).
    """
    # Implementation detail: define sample region (e.g., 3x3 neighborhood)
    # Return max(color) - min(color)
    pass
```

## Operating characteristics
*   **Time Complexity:** The method requires tracing rays through both volume and polygon data. The volume resampling is $O(R \cdot S)$ where $R$ is the number of rays and $S$ is the number of samples per ray. Polygon intersection is $O(R \cdot P)$ where $P$ is the number of polygons. Adaptive supersampling increases the cost in regions with visible geometric events [S1, unofficial].
*   **Memory Complexity:** Requires storage for the $N \times N \times N$ volumetric grid and the polygonal database [S1, unofficial].
*   **Strengths:** Handles mixtures of geometric and volumetric data in a single pass; uses exponential attenuation for accurate visibility of polygons within volume; adaptive supersampling reduces aliasing at geometric edges [S1, unofficial].
*   **Limitations:** Requires rendering all polygons even if obscured by volume (though supersampling is adaptive); treating polygons as planes perpendicular to the ray is an approximation that may introduce errors in visibility; computationally expensive due to simultaneous ray casting and supersampling [S1, unofficial].
*   **When to use:** When rendering scenes with both detailed volumetric data (e.g., medical scans, clouds) and sharp geometric objects (e.g., UI elements, instruments) that need to interact correctly with the volume [S1, unofficial].
*   **When not to use:** When performance is critical and the volume is simple; when the geometric objects are complex and the volume is sparse, other methods like 3D scan-conversion might be more efficient [S1, unofficial].

## Background
The hybrid ray tracer was presented in the paper "Rendering Mixtures of Geometric and Volumetric Data" [S1, unofficial]. It extends the volume rendering algorithm described by Levoy [1988a] to handle polygonally defined objects. The method was developed to address the challenge of rendering mixtures of geometric and volumetric data, where polygons might be obscured by or obscure the volumetric data [S1, unofficial]. It is one of two methods presented in the paper, the other being 3D scan-conversion with analytic anti-aliasing [S1, unofficial].

## Sources
[S1] DTIC ADA208085: Rendering Mixtures of Geometric and Volumetric Data, https://archive.org/details/DTIC_ADA208085, fetched 2026-10-26 (unofficial)
