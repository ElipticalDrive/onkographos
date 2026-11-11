# Optimized LIC

Also called: Fast LIC, LIC with Streamline Reuse, Stalling and Hege LIC

## Overview
Optimized LIC (also known as Fast LIC) is a method to accelerate the Line Integral Convolution algorithm by eliminating redundant numerical integrations and reusing convolution results for adjacent pixels along the same streamline [S1, unofficial][S2]. It differs from standard [Line Integral Convolution (LIC)](7940-line-integral-convolution-lic.md) by computing long field line segments once and incrementally updating the convolution integral for all pixels covered by that segment, rather than recomputing the integral independently for each pixel [S2]. This approach significantly reduces computational cost while maintaining visual fidelity, serving as a foundational optimization for subsequent techniques like [Dye-Advection LIC](7865-dye-advection-lic.md) [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 2D vector field $\mathbf{v}(x,y)$, a random noise texture $T$, and parameters for integration step size $h_t$, convolution half-length $n$, and minimum hit count $minNumHits$.
*   **Output:** An image where each pixel $p$ contains the accumulated and normalized LIC intensity.

**Data Structures:**
*   **Hit Counter Array:** An integer array `numHits` of the same dimensions as the output image, initialized to zero. It tracks how many field lines have passed through each pixel to enable normalization.
*   **Accumulator Array:** A floating-point array `accumulatedIntensity` of the same dimensions as the output image, initialized to zero. It stores the sum of convolution values for each pixel.

**Parameters:**
*   $h_t$: Distance between sample points along the field line. Typically $h_t = 0.5 \times \text{width of a texture cell}$ [S2].
*   $n$: Half-length of the convolution kernel in terms of sample steps.
*   $k$: Normalization constant for the kernel, set to $k = 1/(2n + 1)$ [S2].
*   $M$: Maximum number of steps to traverse along a field line from the starting point. The field line length is typically larger than the convolution length [S2].
*   $minNumHits$: Threshold to determine if a new field line needs to be computed for a pixel [S2].

**Algorithm Steps:**

1.  **Initialization:** Initialize `numHits` and `accumulatedIntensity` to zero for all pixels.
2.  **Field Line Generation and Convolution:** Iterate through each pixel $p$ in the output image.
    *   If `numHits(p)` < $minNumHits$:
        *   Set the starting point $x_0$ to the center of pixel $p$.
        *   Compute the field line segment $\sigma(s)$ starting at $x_0$ using a fourth-order Runge-Kutta method [S2].
        *   Sample the texture $T$ at evenly spaced locations $x_i = \sigma(s_0 + i \cdot h_t)$ for $i = -n, \dots, n$.
        *   Compute the initial convolution integral $I(x_0)$:
            $$ I(x_0) = k \sum_{i=-n}^{n} T(x_i) $$
        *   Add $I(x_0)$ to `accumulatedIntensity(p)`.
        *   Increment `numHits(p)`.
        *   Set $m = 1$.
        *   **Incremental Update Loop:** While $m \le M$:
            *   Compute the forward sample point $x_{m+n}$ and backward sample point $x_{-m-n}$ (relative to the current window).
            *   Update the convolution integral for the forward point $x_m$:
                $$ I(x_{m+1}) = I(x_m) + k [T(x_{m+1+n}) - T(x_{m-n})] $$
            *   Update the convolution integral for the backward point $x_{-m}$:
                $$ I(x_{m-1}) = I(x_m) + k [T(x_{m-1-n}) - T(x_{m+n})] $$
            *   Determine the pixels containing $x_{m+1}$ and $x_{m-1}$.
            *   Add $I(x_{m+1})$ to `accumulatedIntensity` of the pixel containing $x_{m+1}$.
            *   Add $I(x_{m-1})$ to `accumulatedIntensity` of the pixel containing $x_{m-1}$.
            *   Increment `numHits` for the pixels containing $x_{m+1}$ and $x_{m-1}$.
            *   Increment $m$.
3.  **Normalization:** Iterate through each pixel $p$.
    *   If `numHits(p)` > 0:
        *   `output(p)` = `accumulatedIntensity(p)` / `numHits(p)`.
    *   Else:
        *   `output(p)` = 0.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimized_lic(vector_field, texture, image_width, image_height, 
                  h_t, n, M, min_num_hits):
    # k is the normalization constant for the kernel
    k = 1.0 / (2 * n + 1)
    
    # Initialize data structures
    num_hits = [[0 for _ in range(image_width)] for _ in range(image_height)]
    accumulated_intensity = [[0.0 for _ in range(image_width)] for _ in range(image_height)]
    
    # Helper to get pixel coordinates from field line position
    def get_pixel_coords(x, y):
        return int(round(x)), int(round(y))

    # Helper to integrate field line using 4th order Runge-Kutta
    def integrate_field_line(x0, y0, steps, step_size):
        points = []
        x, y = x0, y0
        for i in range(steps):
            points.append((x, y))
            # RK4 integration step
            k1 = vector_field(x, y)
            k2 = vector_field(x + 0.5 * step_size * k1.x, y + 0.5 * step_size * k1.y)
            k3 = vector_field(x + 0.5 * step_size * k2.x, y + 0.5 * step_size * k2.y)
            k4 = vector_field(x + step_size * k3.x, y + step_size * k3.y)
            x += (step_size / 6.0) * (k1.x + 2*k2.x + 2*k3.x + k4.x)
            y += (step_size / 6.0) * (k1.y + 2*k2.y + 2*k3.y + k4.y)
        return points

    # Main loop
    for p_y in range(image_height):
        for p_x in range(image_width):
            if num_hits[p_y][p_x] < min_num_hits:
                # Start field line at pixel center
                x0, y0 = p_x + 0.5, p_y + 0.5
                
                # Compute field line segment
                # We need enough points to cover the convolution window at both ends
                # Total points needed: (M + n) forward + (M + n) backward + 1 center
                # Simplified: generate M*2 + 2*n + 1 points centered at start
                total_points = 2 * M + 2 * n + 1
                field_line = integrate_field_line(x0, y0, total_points, h_t)
                
                # Compute initial convolution I(x0)
                # x0 corresponds to the center of the field line array
                center_idx = M + n
                current_convolution = 0.0
                for i in range(-n, n + 1):
                    idx = center_idx + i
                    tx, ty = field_line[idx]
                    current_convolution += texture(tx, ty)
                current_convolution *= k
                
                # Add initial result to pixel
                accumulated_intensity[p_y][p_x] += current_convolution
                num_hits[p_y][p_x] += 1
                
                # Incremental updates
                current_idx = center_idx
                
                for m in range(1, M + 1):
                    # Update forward: I(x_{m}) from I(x_{m-1})
                    # Note: The formula in source uses I(x_{m+1}) from I(x_m).
                    # Here we iterate m from 1 to M, computing position m relative to start.
                    # Let's align with source notation:
                    # Source: I(x_{m+1}) = I(x_m) + k[T(x_{m+1+n}) - T(x_{m-n})]
                    # We are at index corresponding to x_{m-1} (if m starts at 1, we just did x_0)
                    # Actually, let's track index in field_line array.
                    # Start: index = center_idx (x_0)
                    # Step 1 (m=1): compute x_1 and x_{-1}
                    
                    # Forward step to x_m (where m is current loop variable)
                    # Previous index was center_idx + (m-1) - 1? No.
                    # Let's stick to the source formula directly.
                    # Source defines I(x_m) where m=0 is start.
                    # Loop m from 0 to M-1 to compute m+1.
                    pass 
                
                # Re-implementing loop to match source formula exactly
                # current_idx tracks the index in field_line for x_0
                # We need to compute x_1, x_2... and x_{-1}, x_{-2}...
                
                # Reset for clarity
                current_idx = center_idx # Index for x_0
                
                # We already processed x_0. Now process x_1...x_M and x_{-1}...x_{-M}
                
                # Forward direction
                for m in range(0, M):
                    # Compute I(x_{m+1}) from I(x_m)
                    # Source: I(x_{m+1}) = I(x_m) + k [T(x_{m+1+n}) - T(x_{m-n})]
                    # Indices in field_line:
                    # x_m is at current_idx
                    # x_{m+1} is at current_idx + 1
                    # x_{m+1+n} is at current_idx + 1 + n
                    # x_{m-n} is at current_idx - n
                    
                    next_idx = current_idx + 1
                    add_idx = next_idx + n
                    sub_idx = current_idx - n
                    
                    # Check bounds for texture access
                    if 0 <= add_idx < len(field_line) and 0 <= sub_idx < len(field_line):
                        tx_add, ty_add = field_line[add_idx]
                        tx_sub, ty_sub = field_line[sub_idx]
                        current_convolution += k * (texture(tx_add, ty_add) - texture(tx_sub, ty_sub))
                        
                        # Add to pixel containing x_{m+1}
                        px, py = get_pixel_coords(field_line[next_idx][0], field_line[next_idx][1])
                        if 0 <= px < image_width and 0 <= py < image_height:
                            accumulated_intensity[py][px] += current_convolution
                            num_hits[py][px] += 1
                    
                    current_idx = next_idx

                # Backward direction
                # Reset current_idx to center_idx for backward calculation
                current_idx = center_idx
                # We need to compute I(x_{-1}) from I(x_0)
                # Source: I(x_{m-1}) = I(x_m) + k [T(x_{m-1-n}) - T(x_{m+n})]
                # For m=0: I(x_{-1}) = I(x_0) + k [T(x_{-1-n}) - T(x_{0+n})]
                
                # Re-calculate I(x_0) for backward start? 
                # The source implies I(x_m) is the current value. 
                # For backward, we start from I(x_0) and move to I(x_{-1}).
                # But I(x_0) was already added. We need to compute I(x_{-1}) and add it.
                # The formula updates I(x_{m-1}) based on I(x_m).
                # So we need I(x_0) value.
                
                current_convolution_back = 0.0
                for i in range(-n, n + 1):
                    idx = center_idx + i
                    tx, ty = field_line[idx]
                    current_convolution_back += texture(tx, ty)
                current_convolution_back *= k
                
                for m in range(0, -M, -1): # m goes 0, -1, -2...
                    # We want to compute I(x_{m-1}) from I(x_m)
                    # Current m is 0. We compute I(x_{-1}).
                    # Next m is -1. We compute I(x_{-2}) from I(x_{-1}).
                    
                    # Source: I(x_{m-1}) = I(x_m) + k [T(x_{m-1-n}) - T(x_{m+n})]
                    # Current index in field_line for x_m is current_idx
                    # x_{m-1} is at current_idx - 1
                    # x_{m-1-n} is at current_idx - 1 - n
                    # x_{m+n} is at current_idx + n
                    
                    prev_idx = current_idx - 1
                    add_idx = prev_idx - n
                    sub_idx = current_idx + n
                    
                    if 0 <= add_idx < len(field_line) and 0 <= sub_idx < len(field_line):
                        tx_add, ty_add = field_line[add_idx]
                        tx_sub, ty_sub = field_line[sub_idx]
                        current_convolution_back += k * (texture(tx_add, ty_add) - texture(tx_sub, ty_sub))
                        
                        # Add to pixel containing x_{m-1}
                        px, py = get_pixel_coords(field_line[prev_idx][0], field_line[prev_idx][1])
                        if 0 <= px < image_width and 0 <= py < image_height:
                            accumulated_intensity[py][px] += current_convolution_back
                            num_hits[py][px] += 1
                    
                    current_idx = prev_idx

    # Normalization
    output_image = [[0.0 for _ in range(image_width)] for _ in range(image_height)]
    for p_y in range(image_height):
        for p_x in range(image_width):
            if num_hits[p_y][p_x] > 0:
                output_image[p_y][p_x] = accumulated_intensity[p_y][p_x] / num_hits[p_y][p_x]
            else:
                output_image[p_y][p_x] = 0.0
                
    return output_image
```

**Edge Cases and Pitfalls:**
*   **Aliasing:** If the step size $h_t$ is too large, pixels along the field line may be missed, causing aliasing artifacts [S2].
*   **Accuracy:** Accuracy is critical near vortex centers; incorrect field lines disturb the LIC pattern [S2]. The method uses fourth-order Runge-Kutta to ensure this accuracy, unlike the variable step Euler's method used in CL-LIC [S2].
*   **Boundary Conditions:** Field lines may exit the domain. The algorithm must handle indices out of bounds when accessing the texture or updating pixel accumulators [G].

## Operating characteristics
*   **Time Complexity:** Significantly faster than standard LIC. By reusing field lines and updating integrals incrementally, it avoids redundant numerical integrations for pixels on the same streamline [S1, unofficial]. The cost is proportional to the number of unique field lines generated (determined by $minNumHits$) rather than the total number of pixels [S2].
*   **Memory Complexity:** Requires storage for `numHits` and `accumulatedIntensity` arrays, both $O(W \times H)$ where $W, H$ are image dimensions [S2].
*   **Strengths:** Great saving in computing time compared to standard LIC; maintains visual quality by using accurate integration [S1, unofficial][S2].
*   **Limitations:** Requires careful tuning of $h_t$ to avoid aliasing; accuracy depends heavily on the integration method (RK4) [S2].

## Background
Optimized LIC was proposed by Stalling and Hege in 1995 [S2]. It was introduced as an improvement over the original Line Integral Convolution algorithm by Cabral and Leedom (CL-LIC) [S2]. The method addresses the computational expense of LIC by exploiting the redundancy in field line coverage and convolution integrals [S1, unofficial]. It is widely used in vector field visualization where performance is a concern [S1, unofficial].

## Sources
[S1] Flow Visualization Techniques for CDF using Volume Rendering, https://www.osti.gov/servlets/purl/790542, fetched 2026-10-26 (unofficial)
[S2] PDF Lecture notes: Visualization I Visualization of vector fields using ..., https://www.sci.utah.edu/~pascucci/classes/2009_fall/files/lec07-notes.pdf, fetched 2026-10-27
