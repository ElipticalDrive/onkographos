# Adaptive Undersampling

Also called: Fast Adaptive Undersampling, Two-pass Adaptive Undersampling with Pre-computed Intervals

## Overview
Adaptive Undersampling accelerates volume rendering by replacing the full volume integral calculation for a subset of pixels with interpolation or partial integration, based on pre-computed interval data [S1]. It partitions the pixel screen into blocks, processing one pixel per block in a first pass to store integral values over equal-length intervals in a G-buffer [S1]. In the second pass, unprocessed pixels are either bilinearly interpolated from neighbors or computed using only the necessary intervals from the G-buffer, significantly reducing the number of volume queries [S1]. It differs from standard adaptive undersampling (e.g., recursive block division) by using a fixed two-pass approach with pre-computed intervals, avoiding GPU-inefficient recursive subdivision overhead [S1].

## How it works
**Inputs:**
*   Viewport resolution $W \times H$.
*   3D volume texture (unit cube, side length 1).
*   Transfer function $f$.
*   Parameters: block size $n$ (integer $\ge 2$), maximum number of intervals $M$ (integer).

**Outputs:**
*   Final image buffer.

**Data Structures:**
*   **G-buffer:** Contains $M+1$ parallel textures (or a single texture with $M+1$ channels) with resolution $\lceil W/n \rceil \times \lceil H/n \rceil$.
    *   Entry at index $(u, v)$ stores:
        *   $C$: 4-component vector $(R, G, B, A)$ representing the composited color and transparency of the full ray. Stored in channel 0 (or `G[0].rgb` for color, `G[0].a` for alpha).
        *   $I_i$: 4-component vector $(R_i, G_i, B_i, A_i)$ representing the partial integral over the $i$-th interval, for $0 \le i < M$. Stored in channels $1$ to $M$.
        *   Note: The last interval may be shorter; its value is stored in $I_{M-1}$ or handled via a separate flag/length if precision requires, but typically the recurrence handles the remainder [S1].

**Algorithm Steps:**

1.  **Pass 1: Pre-computation**
    For each block of $n \times n$ pixels in the viewport:
    1.  Select one representative pixel (e.g., top-left).
    2.  Cast a ray from the camera through this pixel.
    3.  Find the starting point $v$ of the non-zero transfer function domain:
        *   Call `GetDistanceForStart(s, f)` where $s$ is the ray origin and $f$ is the far plane.
        *   This function returns the distance to the first point $v$ where $f(v) \neq 0$.
        *   If no such point exists (returns -1), skip this pixel (background).
    4.  Calculate interval length $L$:
        $$ L = \frac{\sqrt{3}}{M} $$
        (Since the volume is a unit cube, the maximum ray length is $\sqrt{3}$).
    5.  Initialize composited color $C_{curr} = (0,0,0,0)$ and transparency $A_{curr} = 0$.
    6.  Initialize interval integrals $I_i = (0,0,0,0)$ for $i = 0 \dots M-1$.
    7.  Iterate through intervals $i$ from 0 to $M-1$:
        *   Determine the start and end points of the $i$-th interval along the ray, starting from $v$.
        *   Let $t_{start} = v + i \cdot L$ and $t_{end} = v + (i+1) \cdot L$.
        *   If $t_{end}$ exceeds the ray exit point (far plane or volume boundary), clamp $t_{end}$ to the exit point. This handles the case where the last interval is shorter [S1].
        *   Perform volume integration (e.g., ray marching) from $t_{start}$ to $t_{end}$ using the compositing recurrence relation:
            $$ C_{new} = C_{curr} + (1 - A_{curr}) \cdot a \cdot c $$
            $$ A_{new} = A_{curr} + (1 - A_{curr}) \cdot a $$
            where $a$ and $c$ are the sample transparency and color at each step.
        *   Store the resulting partial integral $(C_{new} - C_{curr}, A_{new} - A_{curr})$ as $I_i$.
        *   Update $C_{curr} = C_{new}$ and $A_{curr} = A_{new}$.
    8.  Store $C_{curr}$ as $C$ in the G-buffer at the block's coordinate.
    9.  Store $I_0, \dots, I_{M-1}$ in the G-buffer.

2.  **Pass 2: Reconstruction**
    For each pixel $P$ not processed in Pass 1:
    1.  Identify the four nearest G-buffer entries (neighbors) corresponding to the blocks surrounding $P$. Let these be $G_0, G_1, G_2, G_3$.
    2.  Check if the colors of the adjacent pixels are sufficiently close. Define a threshold $\tau$.
        *   If $|Color(G_k) - Color(P_{center})| < \tau$ for all relevant neighbors (or a weighted average), proceed to interpolation.
        *   *Note:* The source specifies checking if "adjacent pixels' colors are sufficiently close" [S1].
    3.  **Interpolation Path:**
        *   If colors are close, compute $C_P$ by bilinearly interpolating the $C$ values from the four G-buffer entries.
        *   Set $A_P$ similarly.
    4.  **Partial Integration Path:**
        *   If colors are not close, calculate $C_P$ using the recurrence relation, starting from the background (0,0,0,0) or the first interval's contribution.
        *   Initialize $CP = (0,0,0,0)$, $AP = 0$.
        *   For $i$ from 0 to $M-1$:
            *   Get $A_i^*$ and $C_i^*$ for the current pixel $P$.
            *   If the neighbor values are deemed "close enough" for this specific interval (or based on global color proximity), interpolate $A_i^*$ and $C_i^*$ bilinearly from the G-buffer entries $I_i$ of the neighbors [S1].
            *   Otherwise, calculate the integral for this interval from scratch (or from the start of the volume, depending on implementation details not fully specified, but the text implies calculating the integral over the specific interval if interpolation is insufficient) [S1]. *Clarification:* The source states $A_i^*$ and $C_i^*$ are interpolated if close, otherwise calculated from the volume integral. To implement "calculated from volume integral" without full ray cast, one might need to re-sample the volume for that interval. However, the text says "calculate integrals only over specific intervals... rather than integrating over the entire ray" [S1]. This implies a full ray cast is avoided, but the specific interval integral is computed. Given the G-buffer stores *partial* integrals, if interpolation fails, the pixel must compute the integral for the *entire* ray or the *remaining* intervals. The text says "calculate CP using recurrence relation... for $0 \le i \le M-1$". This suggests that if interpolation fails, the pixel performs the integration loop, using $A_i^*, C_i^*$ which are either interpolated or freshly computed. If freshly computed, it requires volume access.
            *   Update:
                $$ CP_{i+1} = CP_i + (1 - AP_i) \cdot A_i^* \cdot C_i^* $$
                $$ AP_{i+1} = AP_i + (1 - AP_i) \cdot A_i^* $$
        *   Store $CP$ as the final color.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pass1_block(block_u, block_v, n, M, volume, transfer_func):
    # Coordinates in G-buffer
    gu, gv = block_u, block_v
    
    # Ray setup for the representative pixel (e.g., top-left of block)
    ray_origin, ray_direction = get_ray_for_pixel(block_u * n, block_v * n)
    
    # Find start of non-zero transfer function
    v_dist = get_distance_for_start(ray_origin, ray_direction, transfer_func)
    if v_dist == -1:
        return None # Background
    
    # Interval length
    interval_len = sqrt(3.0) / M
    
    # Initialize accumulators
    comp_color = (0.0, 0.0, 0.0, 0.0)
    comp_alpha = 0.0
    
    interval_integrals = []
    
    current_t = v_dist
    
    for i in range(M):
        t_start = current_t
        t_end = current_t + interval_len
        
        # Clamp to ray exit (simplified: assume ray_exit_dist is known or computed)
        # In unit cube, max dist is sqrt(3). Ray exit depends on geometry.
        # Assuming ray_exit_dist is passed or computed via GetDistanceForStart logic extended
        if t_end > ray_exit_dist:
            t_end = ray_exit_dist
            
        # Integrate over [t_start, t_end]
        # Using standard ray marching compositing
        step_size = get_step_size(volume, t_start, t_end)
        local_color = (0.0, 0.0, 0.0, 0.0)
        local_alpha = 0.0
        
        t = t_start
        while t < t_end:
            sample_pos = ray_origin + ray_direction * t
            sample = volume.sample(sample_pos)
            a = sample.alpha
            c = sample.color
            
            # Compositing recurrence
            local_color = local_color + (1.0 - local_alpha) * a * c
            local_alpha = local_alpha + (1.0 - local_alpha) * a
            
            t += step_size
            
        # Store partial integral
        interval_integrals.append((local_color, local_alpha))
        
        # Update global accumulator
        comp_color = comp_color + (1.0 - comp_alpha) * local_color
        comp_alpha = comp_alpha + (1.0 - comp_alpha) * local_alpha
        
        current_t = t_end
        
    # Store in G-buffer
    gbuffer[gu, gv].color = comp_color
    for i in range(M):
        gbuffer[gu, gv].intervals[i] = interval_integrals[i]
        
    return comp_color

def pass2_pixel(pixel_u, pixel_v, n, M, volume, transfer_func, gbuffer, threshold):
    # Find G-buffer coordinates for 4 neighbors
    # Map pixel_u, pixel_v to block coordinates
    # Get 4 nearest G-buffer entries (top-left, top-right, bottom-left, bottom-right of the block surrounding this pixel)
    neighbors = get_gbuffer_neighbors(pixel_u, pixel_v, n)
    
    # Check color proximity
    neighbor_colors = [nb.color for nb in neighbors]
    avg_neighbor_color = average(neighbor_colors)
    
    # Simple proximity check (example: max difference)
    is_close = max_diff(avg_neighbor_color, neighbor_colors) < threshold
    
    if is_close:
        # Bilinear interpolation of final color
        final_color = bilinear_interpolate(neighbors, pixel_u, pixel_v)
        return final_color
    else:
        # Partial integration
        comp_color = (0.0, 0.0, 0.0, 0.0)
        comp_alpha = 0.0
        
        for i in range(M):
            # Get A_i* and C_i*
            # Interpolate interval integral from neighbors if close enough?
            # Source says: "Ai* and Ci* are interpolated bilinearly ... if those values are close enough and are calculated from the volume integral otherwise."
            # This implies a per-interval check or a global check. The text links it to the color check.
            # Assuming global color check determines the mode for the whole ray for simplicity, 
            # or we check each interval's neighbor consistency.
            
            if is_close:
                ai_star, ci_star = interpolate_interval(neighbors, i, pixel_u, pixel_v)
            else:
                # Calculate integral for this specific interval from scratch
                # This requires ray marching for just this interval
                ai_star, ci_star = calculate_interval_integral(pixel_u, pixel_v, i, M, volume, transfer_func)
            
            # Compositing
            comp_color = comp_color + (1.0 - comp_alpha) * ci_star
            comp_alpha = comp_alpha + (1.0 - comp_alpha) * ai_star
            
        return comp_color
```

**Parameters:**
*   $n$: Side length of square blocks. One pixel processed per $n \times n$ block. Typical values not explicitly given, but $n=2$ is used in examples [S1].
*   $M$: Maximum number of equal-length intervals. User-selected. Typical values not explicitly given, but $M=3$ is used in examples [S1].

**Edge Cases and Pitfalls:**
*   **Woodgrain Artifacts:** Caused by high derivatives of the opacity function near the ray start. Mitigated by finding the starting point $v$ where the transfer function becomes non-zero, rather than starting at the ray origin [S1].
*   **Last Interval Length:** The last interval may be shorter than others if the ray exits the volume before completing the $M$-th interval. This is handled by clamping $t_{end}$ to the ray exit point [S1].
*   **Precision:** Interpolation accuracy depends on the smoothness of the volume and the threshold $\tau$. If neighbors are too different, interpolation fails, and partial integration is required [S1].

## Subtopics

### [Two-pass Adaptive Screen Sampling with Interval Partitioning](5460-two-pass-adaptive-screen-sampling-with-interval-partitioning.md)
This method accelerates volume integral calculation by partitioning the ray integration interval into $M$ equal pieces and using a second pass to interpolate or recompute integrals for pixels based on neighborhood coherence. It builds on [Adaptive Undersampling](5397-adaptive-undersampling.md) by increasing the adaptive undersampling factor, allowing fewer pixels to be processed in the first pass while maintaining image quality through spatial interpolation in the second pass. Compared to standard adaptive undersampling, it increases frames per second (FPS) by a factor of 1.5 on average, and provides further acceleration over two-pass adaptive screen sampling without interval partitioning.

## Operating characteristics
*   **Time Complexity:** Reduces the number of full ray casts by a factor of $n^2$ in the first pass. The second pass performs interpolation for most pixels (91.7% in one example) and partial integration for the rest [S1].
*   **Memory Complexity:** Requires a G-buffer with $M+1$ textures of resolution $\frac{W}{n} \times \frac{H}{n}$.
*   **Performance:** Accelerates adaptive undersampling by a factor of 1.5 on average [S1]. In a specific dataset, 91.7% of pixels were interpolated, 5.7% required 1 interval calculation, 2.1% required 2, 0.1% required 3, and 0.4% required >3 intervals [S1].
*   **Strengths:** Avoids recursive subdivision overhead on GPUs; ensures only necessary intervals are integrated on the second pass [S1].
*   **Limitations:** Requires two passes; interpolation may introduce artifacts if the threshold is not tuned correctly or if geometry changes rapidly [S1].
*   **When to use:** When rendering volumetric data on GPUs where recursive algorithms are inefficient, and high FPS is required with acceptable quality loss [S1].

## Background
Originated in the paper "Fast Adaptive Undersampling for Volume Rendering" by Smirnova and Shubnikov (2019) [S1]. It addresses the computational expense of direct volume rendering, which is proportional to the number of pixels [S1]. Previous methods like recursive block division were inefficient on GPUs [S1]. This method builds on earlier two-pass algorithms [L15] and optimizations for saving integral values in ray intervals [BSM18], but improves upon them by using equal-length intervals and a fixed two-pass structure [S1].

## Sources
[S1] Fast Adaptive Undersampling for Volume Rendering, http://wscg.zcu.cz/wscg2019/Full/A37-full.PDF, fetched 2026-10-27
[S2] Ambient Occlusion Volumes, https://research.nvidia.com/sites/default/files/pubs/2010-06_Ambient-Occlusion-Volumes/McGuire10AOV.pdf, fetched 2026-10-28 (unofficial)
