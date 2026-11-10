# Adaptive Undersampling

Also called: Fast Adaptive Undersampling, Two-pass Adaptive Undersampling with Pre-computed Intervals

## Overview
Adaptive Undersampling is a two-pass rendering method that accelerates volume integral calculation by pre-computing integral values over $M$ equal intervals for a subset of pixels and storing them in a G-buffer. On the second pass, pixels use bilinear interpolation of these interval values for speed, or calculate integrals only over specific intervals if interpolation accuracy is insufficient, rather than integrating over the entire ray [S1]. It builds on [Adaptive Sampling and Resolution](1412-adaptive-sampling-and-resolution.md) by increasing the adaptive undersampling factor, allowing fewer pixels to be processed in the first pass while maintaining image quality through spatial interpolation in the second pass [S1]. Compared to standard adaptive undersampling, it increases frames per second (FPS) by a factor of 1.5 on average, and provides further acceleration over two-pass adaptive screen sampling without interval partitioning [S1].

## How it works
**Inputs:**
- 3D volume texture (data to be visualized).
- Transfer function $f$.
- Viewport resolution $(W, H)$.
- Block size parameter $n$ (side length of square blocks).
- Maximum number of intervals $M$.

**Outputs:**
- Final rendered image.

**Data Structures:**
- **G-buffer:** Contains $M+1$ parallel textures with resolution $\frac{W}{n} \times \frac{H}{n}$.
    - Texture 0 (`G[0]`): Stores the final composited color $C$ (4-component vector: RGB + Transparency $A$).
    - Textures $1 \dots M$ (`G[1] \dots G[M]`): Store the volume integral values $I_i$ for each of the $M$ equal-length intervals. Each $I_i$ is a 4-component vector containing the color and transparency components accumulated over that specific interval.

**Numbered Steps:**

1.  **Pass 1: Pre-computation**
    1.  Partition the pixel set into $n \times n$ blocks.
    2.  For one pixel per block (e.g., the top-left pixel), perform the following:
        a.  Find the starting point $v$ of the non-zero transfer function domain using `GetDistanceForStart(s, f)`, where $s$ is the ray origin and $f$ is the final point. If $v$ is not found (function returns -1), halt processing for this ray.
        b.  Calculate the interval length. Since samples are in a unit cube, the maximum ray length is $\sqrt{3}$. The length of each of the $M$ equal intervals is $L = \frac{\sqrt{3}}{M}$.
        c.  Compute the volume integrals over the $M$ equal intervals starting from $v$. Let $I_i$ be the integral over the $i$-th interval ($0 \le i < M$).
        d.  Compute the final pixel color $C$ by integrating over the entire active ray segment.
        e.  Store $C$ in `G[0]` and the interval integrals $I_i$ in `G[i+1]` for the corresponding block coordinate.
    3.  Handle the last interval: The last interval of integration may be shorter than the others because the ray may exit the volume before completing the $M$-th equal interval. This is handled by a separate calculation for the remaining segment.

2.  **Pass 2: Interpolation and Refinement**
    1.  For each remaining pixel (those not processed in Pass 1):
        a.  Retrieve the colors $C_{neighbor}$ of the four adjacent pixels processed in Pass 1.
        b.  Check if the colors of the adjacent pixels are sufficiently close.
        c.  **If yes:** Interpolate the current pixel color $C_P$ bilinearly from the adjacent pixels' colors stored in `G[0]`.
        d.  **If no:** Calculate $C_P$ using the recurrence relation, starting from the beginning of the ray. For $0 \le i \le M-1$:
            $$C_{P_{i+1}} = C_{P_i} + (1 - A_{P_i}) \cdot A_i^* \cdot C_i^*$$
            $$A_{P_{i+1}} = A_{P_i} + (1 - A_{P_i}) \cdot A_i^*$$
            Where $A_i^*$ and $C_i^*$ are the transparency and color components of the interval integral $I_i$.
            - If the adjacent pixels' interval values $I_i$ are sufficiently close, $A_i^*$ and $C_i^*$ are interpolated bilinearly from the G-buffer values.
            - If interpolation is insufficient, $A_i^*$ and $C_i^*$ are calculated via volume integration over that specific interval only.
        e.  The final color $C_P$ is the result after processing all necessary intervals.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def get_distance_for_start(ray_start, ray_end, transfer_function):
    """
    Finds the distance from ray_start to the point v where the transfer function
    first becomes non-zero. Returns -1 if not found.
    [G] Implementation details depend on specific transfer function and volume data structure.
    """
    # Placeholder for actual ray-marching logic to find start of opacity
    pass

def compute_interval_integral(ray_start, ray_end, interval_index, num_intervals, transfer_function, volume_data):
    """
    Computes the volume integral (color and transparency) over a specific interval.
    """
    # Calculate start and end points of the specific interval along the ray
    # Interval length is sqrt(3) / num_intervals
    pass

def bilinear_interpolate(g_buffer, x, y, texture_index):
    """
    Performs bilinear interpolation on the G-buffer texture at index texture_index.
    """
    pass

def adaptive_undersampling_pass1(viewport_w, viewport_h, n, m, transfer_function, volume_data):
    """
    Pass 1: Process one pixel per n x n block.
    """
    # G-buffer dimensions
    buf_w = viewport_w // n
    buf_h = viewport_h // n
    
    # Initialize G-buffer with M+1 textures
    g_buffer = [create_texture(buf_w, buf_h) for _ in range(m + 1)]
    
    max_ray_length = math.sqrt(3)
    interval_length = max_ray_length / m
    
    for block_y in range(buf_h):
        for block_x in range(buf_w):
            # Select one pixel per block (e.g., top-left)
            pixel_x = block_x * n
            pixel_y = block_y * n
            
            ray_start, ray_end = get_ray_for_pixel(pixel_x, pixel_y, viewport_w, viewport_h)
            
            # Find start of non-zero transfer function domain
            v = get_distance_for_start(ray_start, ray_end, transfer_function)
            
            if v == -1:
                continue # Ray does not hit volume
            
            # Calculate integrals for M intervals
            interval_integrals = []
            current_pos = v
            
            for i in range(m):
                # Determine end of this interval
                next_pos = current_pos + interval_length
                if next_pos > ray_end:
                    next_pos = ray_end
                
                # Compute integral for this interval
                # [G] Actual integration logic using compositing recurrence
                c_i, a_i = integrate_interval(current_pos, next_pos, transfer_function, volume_data)
                interval_integrals.append((c_i, a_i))
                
                current_pos = next_pos
                if current_pos >= ray_end:
                    break
            
            # Compute final color C by integrating from v to ray_end
            # [G] This can be done by summing the interval integrals or re-integrating
            final_c, final_a = integrate_ray(v, ray_end, transfer_function, volume_data)
            
            # Store in G-buffer
            # G[0] stores final color C (RGB + A)
            g_buffer[0].set(block_x, block_y, final_c.r, final_c.g, final_c.b, final_a)
            
            # G[1..M] store interval integrals I_i
            for i, (c_i, a_i) in enumerate(interval_integrals):
                if i < m:
                    g_buffer[i + 1].set(block_x, block_y, c_i.r, c_i.g, c_i.b, a_i)
    
    return g_buffer

def adaptive_undersampling_pass2(viewport_w, viewport_h, n, m, g_buffer, transfer_function, volume_data):
    """
    Pass 2: Process remaining pixels using interpolation or partial integration.
    """
    image = create_image(viewport_w, viewport_h)
    
    for y in range(viewport_h):
        for x in range(viewport_w):
            # Skip pixels processed in Pass 1
            if x % n == 0 and y % n == 0:
                continue
            
            # Get neighbors from G-buffer
            # Neighbors are at block coordinates
            block_x = x // n
            block_y = y // n
            
            # Retrieve colors from G[0] for the 4 surrounding blocks
            # [G] Handle boundary conditions for blocks at edges
            neighbors_colors = [
                g_buffer[0].get(block_x - 1, block_y - 1),
                g_buffer[0].get(block_x, block_y - 1),
                g_buffer[0].get(block_x - 1, block_y),
                g_buffer[0].get(block_x, block_y)
            ]
            
            # Check if neighbors are sufficiently close
            if are_colors_close(neighbors_colors):
                # Interpolate color
                c_p = bilinear_interpolate_colors(neighbors_colors, x % n, y % n)
                image.set(x, y, c_p)
            else:
                # Calculate using recurrence relation
                ray_start, ray_end = get_ray_for_pixel(x, y, viewport_w, viewport_h)
                v = get_distance_for_start(ray_start, ray_end, transfer_function)
                
                if v == -1:
                    image.set(x, y, 0, 0, 0, 0)
                    continue
                
                c_p = (0, 0, 0, 0) # Initial color
                a_p = 0.0 # Initial transparency
                
                # Iterate through intervals
                for i in range(m):
                    # Get interval integral I_i from G-buffer
                    # [G] Interpolate I_i from neighbors if close, else compute
                    c_i_star, a_i_star = get_interval_value(i, block_x, block_y, g_buffer, transfer_function, volume_data)
                    
                    # Recurrence relation
                    # C_{i+1} = C_i + (1 - A_i) * a_i * c_i
                    # A_{i+1} = A_i + (1 - A_i) * a_i
                    
                    # [G] Vector operations for color and transparency
                    one_minus_a_p = 1.0 - a_p
                    c_p = (
                        c_p[0] + one_minus_a_p * a_i_star * c_i_star[0],
                        c_p[1] + one_minus_a_p * a_i_star * c_i_star[1],
                        c_p[2] + one_minus_a_p * a_i_star * c_i_star[2],
                        c_p[3] # Alpha doesn't add to color, but transparency accumulates
                    )
                    a_p = a_p + one_minus_a_p * a_i_star
                    
                    if a_p >= 1.0:
                        break
                
                image.set(x, y, c_p[0], c_p[1], c_p[2], a_p)
    
    return image
```

**Parameters:**
- $n$: The side length of the square blocks into which the pixel set is partitioned. One pixel is processed per $n \times n$ block on the first pass. [S1]
- $M$: The user-selected maximum number of equal-length intervals into which the integration domain is divided. The G-buffer stores $M$ parallel textures. [S1]

**Edge Cases and Pitfalls:**
- **Woodgrain artifacts:** Caused by high derivatives of the opacity function near the ray start. Mitigated by finding the starting point $v$ where the transfer function becomes non-zero, rather than starting at the ray origin. This also increases the probability that the first integrals $I_0$ coincide on adjacent rays [S1].
- **Last interval length:** The last interval of integration may be shorter than the others because the ray may exit the volume before completing the $M$-th equal interval. This is handled by a separate 'lastStep' calculation [S1].
- **GPU Efficiency:** Unlike recursive block division methods which are inefficient on GPUs due to concurrent processing groups, this fixed two-pass approach avoids the overhead of recursive subdivision [S1].

## Subtopics

### [Two-pass Adaptive Screen Sampling with Interval Partitioning](5460-two-pass-adaptive-screen-sampling-with-interval-partitioning.md)
This method accelerates volume integral calculation by partitioning the ray integration interval into $M$ equal pieces and using a second pass to interpolate or recompute integrals for pixels based on neighborhood coherence. It builds on [Adaptive Undersampling](5397-adaptive-undersampling.md) by increasing the adaptive undersampling factor, allowing fewer pixels to be processed in the first pass while maintaining image quality through spatial interpolation in the second pass. Compared to standard adaptive undersampling, it increases frames per second (FPS) by a factor of 1.5 on average, and provides further acceleration over two-pass adaptive screen sampling without interval partitioning.

## Operating characteristics
- **Time Complexity:** Accelerates rendering by reducing the length of the interval of integration. On average, it accelerates adaptive undersampling by a factor of 1.5 [S1].
- **Measured Performance:** In a real dataset example, 91.7% of pixels were interpolated, 5.7% required 1 interval calculation, 2.1% required 2, 0.1% required 3, and 0.4% required >3 intervals [S1].
- **Scaling:** The method scales with the viewport resolution and the number of intervals $M$. The G-buffer resolution is $n$ times less than the viewport along each side [S1].
- **Strengths:** Avoids recursive subdivision overhead on GPUs; reduces computation for most pixels via interpolation; handles high-opacity gradients effectively by starting integration at the non-zero transfer function domain [S1].
- **Limitations:** Requires two passes; performance depends on the coherence of adjacent pixels; requires storing $M$ parallel textures in the G-buffer [S1].
- **When to use:** When rendering volumetric data on GPUs where recursive algorithms are inefficient, and when high frame rates are needed with acceptable interpolation artifacts [S1].
- **When not to use:** When the volume has high-frequency details that are not coherent between adjacent pixels, leading to frequent interval recomputation [S1].

## Background
Adaptive Undersampling was proposed in the paper "Fast Adaptive Undersampling for Volume Rendering" by Smirnova and Shubnikov (2019) [S1]. It addresses the computational expense of direct volume rendering, where running time is proportional to the number of pixels. The method improves upon earlier two-pass algorithms by pre-computing interval integrals, allowing for more efficient partial integration in the second pass [S1].

## Sources
[S1] Fast Adaptive Undersampling for Volume Rendering, http://wscg.zcu.cz/wscg2019/Full/A37-full.PDF, fetched 2026-10-27
