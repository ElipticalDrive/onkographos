# Ray Casting Loop with Supersampling

Also called: Image-order Rendering Loop with Supersampling

## Overview
Renders an image by casting rays per pixel through a scene, with optional multi-sample anti-aliasing using uniform sub-pixel jittering [S1, unofficial]. It extends the basic [Image-order Rendering Loop](2938-image-order-rendering-loop.md) by introducing a sampling loop that averages multiple jittered ray samples per pixel to reduce aliasing artifacts [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Camera object (providing `getWidth()`, `getHeight()`, `drawPixel()`), boolean flag `SAMPLING`, integer `SN` (samples per axis), and a `trace(x, y)` function that returns a `Color` for a given sub-pixel coordinate.
- Output: An image drawn via `camera->drawPixel()`.

**Data Structures:**
- `Color`: A vector-like structure supporting addition, scalar division, and clamping.
- `Vector3D`: A 3D vector structure supporting arithmetic operations (addition, subtraction, multiplication, division) [S1, unofficial].
- `Ray`: Contains origin, direction, and hit information (not fully detailed in the loop context, but assumed to be handled by `trace`) [S1, unofficial].

**Parameters:**
- `SN`: Number of samples per axis for supersampling. Typical value: 5 [S1, unofficial].
- `SAMPLING`: Boolean flag enabling/disabling supersampling. Typical value: false [S1, unofficial].

**Steps:**
1. Initialize pixel index `i = 0`.
2. Loop while `renderLoop()` returns false.
3. Inside `renderLoop()`, parallelize over the vertical pixel coordinate `j` from 0 to `camera->getHeight() - 1`.
4. If `SAMPLING` is false:
   a. Call `trace(i, j)` to get the color.
   b. Clamp the color.
   c. Draw the pixel at `(i, j)` with the clamped color.
5. If `SAMPLING` is true:
   a. Initialize an accumulator `color` to zero.
   b. Loop `p` from 0 to `SN - 1`.
   c. Loop `q` from 0 to `SN - 1`.
   d. Generate a random number `e` from a uniform distribution in `[0, 1)` using `randGen(1)`.
   e. Compute sub-pixel coordinates:
      $$x = i + \frac{p + e}{SN}$$
      $$y = j + \frac{q + e}{SN}$$
   f. Call `trace(x, y)` to get the sample color `c`.
   g. Clamp `c`.
   h. Accumulate: `color += c`.
   i. After the inner loops, draw the pixel at `(i, j)` with `color / (SN * SN)`.
6. Increment `i`.
7. If `i == camera->getWidth()`, reset `i = 0` and return true to terminate the loop.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import random

def render_loop(camera, trace, sampling=False, sn=5):
    """
    Renders the image using ray casting with optional supersampling.
    
    Args:
        camera: Object with getWidth(), getHeight(), and drawPixel(x, y, color).
        trace: Function that takes (x, y) and returns a Color object.
        sampling: Boolean flag to enable supersampling.
        sn: Number of samples per axis.
    """
    width = camera.getWidth()
    height = camera.getHeight()
    i = 0
    
    # Parallelization is handled by the caller or framework in the source [S1, unofficial]
    # Here we simulate the loop structure
    
    while True:
        # Parallel loop over j
        for j in range(height):
            if not sampling:
                # Single sample
                color = trace(i, j)
                color.clamp()
                camera.drawPixel(i, j, color)
            else:
                # Supersampling
                color_accumulator = Color(0, 0, 0) # Assuming Color supports initialization to zero
                total_samples = sn * sn
                
                for p in range(sn):
                    for q in range(sn):
                        # Generate random e in [0, 1)
                        e = random.random()
                        
                        # Compute sub-pixel coordinates
                        x = i + (p + e) / sn
                        y = j + (q + e) / sn
                        
                        # Trace ray
                        c = trace(x, y)
                        c.clamp()
                        
                        # Accumulate
                        color_accumulator += c
                
                # Average samples
                final_color = color_accumulator / total_samples
                camera.drawPixel(i, j, final_color)
        
        i += 1
        if i == width:
            break
```

**Edge Cases and Pitfalls:**
- Precision: The sub-pixel coordinates `x` and `y` are floating-point values. Ensure `trace` handles these correctly.
- Boundaries: The loop assumes `i` and `j` are within valid bounds. The reset logic `if i == width` ensures `i` wraps around.
- Randomness: `randGen(1)` is assumed to return a value in `[0, 1)`. If it returns `[0, 1]`, the jitter range changes slightly, potentially affecting aliasing quality [S1, unofficial].

## Operating characteristics
- Time Complexity: $O(W \times H \times SN^2)$ where $W$ is width, $H$ is height, and $SN$ is samples per axis. With supersampling, the cost is $SN^2$ times higher than single-sample rendering.
- Memory Complexity: $O(1)$ per pixel during rendering, as only the accumulator color is stored.
- Strengths: Reduces aliasing artifacts by averaging multiple samples. Simple to implement.
- Limitations: Increases rendering time by a factor of $SN^2$. Does not account for depth-of-field or motion blur unless extended.
- When to use: When anti-aliasing is needed and performance is not critical, or for offline rendering.
- When not to use: For real-time applications where $SN^2$ overhead is prohibitive.

## Background
Originated in the context of volume ray casting implementations, specifically in educational or research codebases for computer graphics courses [S1, unofficial]. The method is a standard technique in image-order rendering to improve visual quality by mitigating aliasing.

## Sources
[S1] CSE-333-Computer-Graphics-2020/VolumeRayCasting, https://github.com/CSE-333-Computer-Graphics-2020/VolumeRayCasting, fetched 2026-10-26 (unofficial)
