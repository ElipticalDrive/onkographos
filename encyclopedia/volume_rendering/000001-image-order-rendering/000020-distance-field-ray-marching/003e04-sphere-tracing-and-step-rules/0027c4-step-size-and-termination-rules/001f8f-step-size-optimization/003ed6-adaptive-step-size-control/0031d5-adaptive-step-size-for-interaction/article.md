# Adaptive Step Size for Interaction

Also called: Interactive Step Size Increase, Variable Step Size

## Overview
Adaptive Step Size for Interaction is a performance optimization technique used during interactive volume rendering or ray marching to maintain high frame rates while the user manipulates the viewpoint. It solves the problem of excessive computational cost by dynamically increasing the sampling step size during interaction, accepting lower visual fidelity in exchange for fluency, and then reverting to high-quality parameters for the final still image. It differs from static step size methods by being time-dependent (interaction state) rather than geometry-dependent, and from other adaptive methods like [Adaptive Step Size Ray Marching](2312-adaptive-step-size-ray-marching.md) by prioritizing temporal smoothness over geometric accuracy during the interaction phase.

## How it works
**Inputs and Outputs:**
- **Input:** Current ray parameters ($\mathbf{o}, \mathbf{d}$), scene data (volume or heightmap), interaction state flag ($I$), high-quality step size ($\Delta t_{high}$), low-quality step size ($\Delta t_{low}$), and maximum traversal distance ($t_{max}$).
- **Output:** Rendered pixel color/opacity.

**Data Structures:**
- **Step Size Parameters:** Two scalar values defining the step size for interaction and non-interaction states.
- **Interaction Flag:** A boolean or integer indicating if the user is currently manipulating the camera.

**Algorithm Steps:**
1. **Detect Interaction State:** Determine if the user is currently interacting (e.g., rotating, zooming). Let this be $I$.
2. **Select Step Size:**
   - If $I$ is true, set $\Delta t = \Delta t_{low}$.
   - If $I$ is false, set $\Delta t = \Delta t_{high}$.
   - Note: $\Delta t_{low} > \Delta t_{high}$ typically.
3. **Ray Marching Loop:**
   - Initialize $t = 0$.
   - While $t < t_{max}$:
     - Evaluate scene function $f(\mathbf{p})$ at $\mathbf{p} = \mathbf{o} + t\mathbf{d}$.
     - Accumulate color/opacity.
     - Update $t \leftarrow t + \Delta t$.
4. **Final Pass (Optional but Recommended):** If $I$ becomes false after interaction, perform a final render pass with $\Delta t_{high}$ to produce a high-quality still image.

**Formulas:**
- Ray: $\mathbf{p}(t) = \mathbf{o} + t\mathbf{d}$
- Step update: $t_{i+1} = t_i + \Delta t$
- Interaction condition: $\Delta t = \begin{cases} \Delta t_{low} & \text{if } I \\ \Delta t_{high} & \text{if } \neg I \end{cases}$

**Parameters:**
- $\Delta t_{low}$: Larger step size for interaction. Value depends on scene resolution and desired frame rate. Not given in the sources.
- $\Delta t_{high}$: Standard step size for high-quality rendering. Not given in the sources.
- $t_{max}$: Maximum distance for ray traversal. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Skipping Features:** If $\Delta t_{low}$ is too large, significant geometric features (e.g., "big mountains" in heightmap rendering) may be completely skipped, leading to visual artifacts [S2, unofficial].
- **Artifacts:** Increasing step size reduces computational effort but can introduce visible artifacts in the visualization [S1, unofficial].
- **Transition Artifacts:** Switching between step sizes may cause visual popping if not handled smoothly. The method suggests producing a high-quality still image *after* interaction ceases, rather than blending step sizes dynamically.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_pixel_interactive(o, d, scene, is_interacting, delta_t_low, delta_t_high, t_max):
    """
    Renders a single pixel using adaptive step size based on interaction state.
    
    Args:
        o: Ray origin (vec3)
        d: Ray direction (vec3, normalized)
        scene: Scene data structure with evaluate(p) method
        is_interacting: Boolean, True if user is currently manipulating camera
        delta_t_low: Step size for interaction (larger)
        delta_t_high: Step size for high quality (smaller)
        t_max: Maximum ray distance
        
    Returns:
        Color (vec3)
    """
    # Select step size based on interaction state
    if is_interacting:
        delta_t = delta_t_low
    else:
        delta_t = delta_t_high
    
    t = 0.0
    accumulated_color = vec3(0.0)
    accumulated_opacity = 0.0
    
    # Ray marching loop
    while t < t_max:
        p = o + t * d
        # Evaluate scene function (e.g., density, color)
        # Note: Specific evaluation depends on scene type (volume/heightmap)
        sample = scene.evaluate(p)
        
        # Accumulate color (simplified front-to-back compositing)
        # This is a placeholder for actual compositing logic
        color = sample.color
        opacity = sample.opacity
        
        # Standard front-to-back compositing
        alpha = opacity * (1.0 - accumulated_opacity)
        accumulated_color += color * alpha
        accumulated_opacity += alpha
        
        if accumulated_opacity >= 1.0:
            break
            
        t += delta_t
        
    return accumulated_color
```

## Subtopics

### Early Ray Termination
Stops traversal when accumulated opacity is too high, reducing computation for distant regions. [Link](2312-adaptive-step-size-ray-marching.md)

### Dynamic Sampling Step Adjustment
Adjusts step size based on local scene complexity or density gradients. [Link](2442-dynamic-sampling-step-adjustment.md)

### Variance-based Step Size Adaptation
Uses variance of samples to determine step size, reducing samples in homogeneous areas. [Link](13161-variance-based-step-size-adaptation.md)

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of samples. During interaction, $N$ is reduced by a factor of $\Delta t_{high} / \Delta t_{low}$.
- **Memory Complexity:** $O(1)$ per ray, assuming scene data is accessed on-demand.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Performance improves linearly with increased step size.
- **Strengths:** Significantly improves frame rate during interaction, providing a more responsive user experience.
- **Limitations:** Visual quality is degraded during interaction; large features may be skipped if step size is too aggressive; requires a final high-quality pass for still images.
- **When to Use:** Interactive visualization where frame rate is critical and minor visual artifacts are acceptable during manipulation.
- **When Not to Use:** Final image rendering, scenes with fine details that may be skipped, or when visual fidelity is paramount over interaction speed.

## Background
This method was described by Hadwiger and Rezk Salama in 2004 [S1, unofficial]. It is a common trick in real-time volume rendering and heightmap visualization to balance performance and quality. The concept of discarding information as distance from the camera increases is also discussed in the context of variable step size for heightmaps [S2, unofficial].

## Sources
[S1] PDF Direct Volume Rendering - KTH, https://www.kth.se/social/files/565e35dff27654457fb84363/08_VolumeRendering.pdf, fetched 2026-10-27 (unofficial)
[S2] flipcode - Realtime Voxel Landscape Engines, https://www.flipcode.com/archives/Realtime_Voxel_Landscape_Engines-Part_4_Level_of_Detail_and_Optimisation.shtml, fetched 2026-10-30 (unofficial)
