# Constant Density Volume Raycasting

Also called: Cube Raycaster (Constant Density)

## Overview
Constant Density Volume Raycasting is a simplified direct volume rendering method that assumes a uniform density field (value 1) within a defined spatial extent, rendering it as a solid object with additive color accumulation. It differs from standard direct volume ray casting by replacing scalar volume lookup and transfer functions with a binary inside/outside test and a fixed color contribution, using saturation-based early termination rather than opacity-based alpha compositing.

## How it works
**Inputs and Outputs**
- **Input**: Camera position $\mathbf{o}$, ray direction $\mathbf{d}$, ray segment bounds $[t_{near}, t_{far}]$ (or sphere intersection parameters), and a spatial extent definition (e.g., bounding box or sphere).
- **Output**: Final pixel color $\mathbf{C}_{out}$ (RGB).

**Data Structures**
- **Ray**: Defined by origin $\mathbf{o}$ and direction $\mathbf{d}$.
- **State**: Accumulated color $\mathbf{C}_{acc} = (0, 0, 0)$, boolean flag `isSaturated`.
- **Parameters**: `stepSize` (distance between samples), `saturationThreshold` (default $1.0$).

**Steps**
1.  **Initialization**: Set $\mathbf{C}_{acc} = (0, 0, 0)$ and `isSaturated` = false. Determine the starting parameter $t_{start}$ for the ray. If the camera is inside the volume (e.g., a sphere) and the intersection $t_1 > 0$ but $t_0 \le 0$, set $t_{start} = 0$ to avoid starting inside the geometry incorrectly [S1, unofficial]. Otherwise, use the standard entry point $t_{near}$ [G].
2.  **Raymarching Loop**: Iterate while $t < t_{far}$ and `!isSaturated`:
    a.  Update current position: $\mathbf{p} = \mathbf{o} + t \cdot \mathbf{d}$.
    b.  Increment $t \leftarrow t + \text{stepSize}$.
    c.  **Inside Test**: Check if $\mathbf{p}$ is inside the defined extent.
    d.  **Color Contribution**: If inside, compute color contribution $\mathbf{c}$. The density function returns $1$. The color function maps this to a fixed color, typically white $\mathbf{c} = (0.01, 0.01, 0.01)$ [S1, unofficial]. If outside, $\mathbf{c} = (0, 0, 0)$.
    e.  **Accumulation**: Add to accumulator: $\mathbf{C}_{acc} \leftarrow \mathbf{C}_{acc} + \mathbf{c}$.
    f.  **Saturation Check**: If any channel of $\mathbf{C}_{acc}$ exceeds `saturationThreshold` ($1.0$), set `isSaturated` = true.
3.  **Termination**: If `isSaturated` becomes true, break the loop.
4.  **Output**: Return $\mathbf{C}_{acc}$. Note that normalization is typically not applied in this variant [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def constant_density_raycast(cam_pos, ray_dir, t_near, t_far, step_size, extent_checker):
    """
    Renders a constant density volume using additive accumulation and saturation termination.
    """
    # Initialize accumulator
    final_color = [0.0, 0.0, 0.0]
    is_saturated = False
    
    # Handle camera inside volume edge case [S1, unofficial]
    # If ray starts inside, ensure we march forward from a valid entry point.
    # For a sphere, if t0 <= 0 and t1 > 0, start at t=0.
    t_current = t_near
    if t_current < 0.0:
        t_current = 0.0

    while t_current < t_far and not is_saturated:
        # Calculate sample position
        location = [
            cam_pos[0] + t_current * ray_dir[0],
            cam_pos[1] + t_current * ray_dir[1],
            cam_pos[2] + t_current * ray_dir[2]
        ]
        
        # Check if inside the defined extent (e.g., bounding box or sphere)
        if extent_checker.is_inside(location):
            # Constant density function returns 1
            density = 1.0
            
            # Color function: density > 0 yields white with small step contribution
            # Source uses float4(0.01, 0.01, 0.01, 1) [S1, unofficial]
            color_step = [0.01, 0.01, 0.01]
            
            # Additive accumulation
            final_color[0] += color_step[0]
            final_color[1] += color_step[1]
            final_color[2] += color_step[2]
            
            # Saturation check [S1, unofficial]
            if (final_color[0] > 1.0 or 
                final_color[1] > 1.0 or 
                final_color[2] > 1.0):
                is_saturated = True
        
        # Advance ray
        t_current += step_size

    return final_color
```

**Parameters**
- `saturationThreshold`: Float, default $1.0$. The value at which raymarching terminates [S1, unofficial].
- `stepSize`: Distance between samples. Not explicitly defined in sources, but required for implementation [G].
- `extent_checker`: Object or function defining the volume boundary (e.g., sphere intersection logic) [S1, unofficial].

**Edge Cases and Pitfalls**
- **Camera Inside Volume**: If the camera is inside the volume (e.g., a sphere) and the intersection logic yields $t_0 \le 0$ and $t_1 > 0$, the raymarch must start at $t=0$ rather than $t_0$ to avoid incorrect geometry handling [S1, unofficial].
- **No Alpha Compositing**: Unlike standard volume rendering, this method does not compute opacity $\alpha$ or use front-to-back alpha blending. It uses simple additive color accumulation, which can lead to over-brightening if not clamped or terminated via saturation [S1, unofficial].

## Subtopics

## Operating characteristics
- **Time Complexity**: $O(N)$ where $N$ is the number of steps along the ray. Early termination via saturation can reduce the average number of steps significantly for opaque objects.
- **Memory Complexity**: $O(1)$ per ray, as no volume texture or transfer function data is stored/accessed.
- **Strengths**: Extremely fast due to lack of texture lookups, interpolation, and complex transfer functions. Ideal for simple geometric primitives or debugging.
- **Limitations**: Cannot represent internal structure, varying material properties, or transparency gradients. Visual output is limited to solid-colored shapes with additive brightness.
- **When to use**: Simple geometric visualization, performance-critical applications where only the presence of matter matters, or as a baseline for testing ray traversal logic.
- **When not to use**: Scientific visualization requiring data-driven color mapping, translucent materials, or detailed internal structure.

## Background
This method is implemented in the `CubeRaycaster` class within the open-source project `VolumetricRayCasting-GL` by Balázs Banto [S1, unofficial]. It serves as a simplified variant of direct volume raycasting, stripping away the scalar field sampling and transfer function stages to focus on basic ray-extent intersection and color accumulation.

## Sources
[S1] balazsbanto/VolumetricRayCasting-GL, https://github.com/balazsbanto/VolumetricRayCasting-GL, fetched 2026-10-26 (unofficial)
