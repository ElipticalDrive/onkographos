# MAB Compositing with Gaze-Contingent Blending

Also called: MAB compositing, Multi-Anchor Buffer compositing

## Overview
MAB compositing is a gaze-contingent rendering technique that maps framebuffer fragments to a Multi-Anchor Buffer (MAB) to retrieve and blend color values from different resolution levels based on the distance to the center of gaze [S1]. It solves the performance bottleneck in immersive VR volume rendering by dynamically adjusting resolution distribution to maintain high frame rates, thereby preventing cybersickness caused by performance fluctuations [S1]. Unlike standard fixed-resolution ray casting, it blends multiple resolution levels simultaneously using a smooth-step function to ensure unnoticeable transitions between detail levels [S1].

## How it works
The method operates in two distinct phases: MAB generation (raycasting) and MAB compositing (blending) [S1]. It relies on three core functions: $f_a$ (gaze distance to blending factors), $f_b$ (screen coordinates to MAB coordinates), and $f_c$ (MAB coordinates to resolution level and framebuffer position) [S1].

### Inputs and Outputs
*   **Inputs:**
    *   Screen-space coordinates $(u, v)$ of the current fragment [S1].
    *   Gaze center $(u_g, v_g)$ in screen space [S1].
    *   MAB data structure containing color values at multiple resolution levels [S1].
    *   $f_a$: A 1D lookup texture mapping gaze distance to blending factors [S1].
*   **Outputs:**
    *   Final composited color $C_{final}$ for the framebuffer fragment [S1].

### Data Structures
**Multi-Anchor Buffer (MAB):**
A buffer storing rendered volume data at multiple resolution levels [S1]. The specific internal layout is not detailed, but it is accessed via coordinates derived from $f_b$ [S1].

**Blending Factor Texture ($f_a$):**
A 1D lookup texture.
*   **Layout:** The individual resolution levels of the MAB are mapped to the four color channels (R, G, B, A) of the texture [S1].
*   **Content:** Each channel's value represents the blending factor for the corresponding resolution level [S1].
*   **Properties:** Contains a smooth-step function between adjacent resolution levels to ensure unnoticeable blending [S1].

### Numbered Steps

**Phase 1: MAB Generation (Raycasting)**
1.  Render a full-screen primitive covering the entire MAB to generate fragments for all resolution levels [S1].
2.  For each generated MAB fragment, evaluate function $f_c$ using its MAB coordinates to output:
    *   The resolution level $L$ of the fragment [S1].
    *   The corresponding position $(u_{fb}, v_{fb})$ in the output framebuffer [S1].
3.  Use $(u_{fb}, v_{fb})$ to calculate the distance $d$ to the center of gaze [S1].
4.  Evaluate function $f_a$ using $d$ to retrieve the set of blending factors for all resolution levels [S1].
5.  Select the specific blending factor corresponding to resolution level $L$ from the result of $f_a$ [S1].
6.  If the selected blending factor is zero, discard the MAB fragment [S1].
7.  Otherwise, store the raycast color value at the MAB coordinates [S1].

**Phase 2: MAB Compositing (Blending)**
1.  Render a full-screen primitive covering the output framebuffer to initiate compositing for each generated fragment [S1].
2.  For each framebuffer fragment with screen-space coordinates $(u, v)$:
    a. Compute the distance $d$ to the center of gaze based on $(u, v)$ [S1].
    b. Evaluate function $f_a$ using $d$ to output a set of blending factors $\{ \beta_0, \beta_1, \beta_2, \beta_3 \}$ for the MAB resolution levels [S1].
    c. Evaluate function $f_b$ using $(u, v)$ to output a set of corresponding coordinates $\{ (u_0, v_0), (u_1, v_1), (u_2, v_2), (u_3, v_3) \}$ in the MAB [S1].
    d. Initialize the accumulated color $C_{acc} = (0, 0, 0)$ [G].
    e. For each resolution level $i$ where the blending factor $\beta_i > 0$:
        i. Perform a lookup into the MAB at coordinates $(u_i, v_i)$ to obtain the stored color value $C_i$ [S1].
        ii. Blend $C_i$ into $C_{acc}$ [S1].
3.  Output $C_{acc}$ as the final pixel color [S1].

### Formulas and Definitions
*   **Distance to Gaze:** $d = \sqrt{(u - u_g)^2 + (v - v_g)^2}$ [G].
*   **Blending Factors:** $\beta_i = f_a(d)_i$, where $f_a$ is the 1D texture and $i$ indexes the color channel/resolution level [S1].
*   **MAB Coordinates:** $(u_i, v_i) = f_b(u, v)_i$ [S1].
*   **Inverse Mapping (for culling):** $f_c(u_{mab}, v_{mab}) \rightarrow (L, u_{fb}, v_{fb})$ [S1].
*   **Culling Condition:** Fragment is discarded if $\beta_i = 0$ [S1].

### Parameters
*   **$f_a$ Implementation:** 1D lookup texture [S1].
*   **$f_b$ Implementation:** Maps screen-space coordinates to MAB coordinates [S1].
*   **$f_c$ Implementation:** Maps MAB coordinates to resolution level and framebuffer position [S1].
*   **Smooth-step:** Added between adjacent resolution levels in $f_a$ to ensure unnoticeable blending [S1].

### Edge Cases and Pitfalls
*   **Blending Artifacts:** To prevent visible seams between resolution levels, a smooth-step function must be integrated into the $f_a$ texture [S1].
*   **Performance Fluctuations:** In VR, camera position and viewing direction fluctuations can cause severe performance variations. The system must track rendering performance and adjust the resolution distribution (via $f_a$) on the fly to maintain a threshold refresh rate [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def mab_compositing_with_gaze_contingent_blending(
    framebuffer_width, framebuffer_height, mab_width, mab_height,
    gaze_center_u, gaze_center_v,
    mab_buffer, f_a_texture, f_b_function, f_c_function
):
    """
    Performs MAB compositing with gaze-contingent blending.
    
    Args:
        framebuffer_width, framebuffer_height: int, dimensions of output framebuffer.
        mab_width, mab_height: int, dimensions of MAB.
        gaze_center_u, gaze_center_v: float, gaze center in screen space [S1].
        mab_buffer: object, stores color values for MAB fragments.
        f_a_texture: 1D array/texture, maps distance to blending factors [R, G, B, A] [S1].
        f_b_function: callable, takes (u, v) and returns list of MAB coordinates [S1].
        f_c_function: callable, takes (u_mab, v_mab) and returns (level, u_fb, v_fb) [S1].
        
    Returns:
        list of vec3, the final composited colors for each framebuffer pixel.
    """
    
    # Phase 1: MAB Generation (Raycasting)
    # Render full-screen primitive over MAB to generate fragments for all levels [S1]
    for v_mab in range(mab_height):
        for u_mab in range(mab_width):
            # Evaluate f_c: MAB coords -> resolution level + framebuffer position [S1]
            level, u_fb, v_fb = f_c_function(u_mab, v_mab)
            
            # Compute distance to gaze using framebuffer position [S1]
            dist = math.sqrt((u_fb - gaze_center_u)**2 + (v_fb - gaze_center_v)**2)
            
            # Evaluate f_a: distance -> blending factors for all levels [S1]
            blending_factors = f_a_texture.sample(dist)
            
            # Select factor for this level [S1]
            beta = blending_factors[level]
            
            # Discard if zero, otherwise store color [S1]
            if beta == 0:
                continue
            # Raycast color computation is assumed [G]
            c_ray = vec3(0.0, 0.0, 0.0)
            mab_buffer.set_color(u_mab, v_mab, c_ray)

    # Phase 2: MAB Compositing (Blending)
    final_colors = []
    for v in range(framebuffer_height):
        for u in range(framebuffer_width):
            # Compute distance to gaze [S1]
            dist = math.sqrt((u - gaze_center_u)**2 + (v - gaze_center_v)**2)
            
            # Evaluate f_a: distance -> blending factors [S1]
            blending_factors = f_a_texture.sample(dist)
            
            # Evaluate f_b: screen coords -> MAB coordinates [S1]
            mab_coords_list = f_b_function(u, v)
            
            # Initialize accumulator [G]
            c_acc = vec3(0.0, 0.0, 0.0)
            
            # Blend non-zero factors [S1]
            for i in range(4):
                beta_i = blending_factors[i]
                if beta_i == 0:
                    continue
                u_mab, v_mab = mab_coords_list[i]
                c_i = mab_buffer.get_color(u_mab, v_mab)
                # Blend C_i into C_acc [S1]
                # Exact blending equation not specified in sources [S1]
                c_acc = c_acc + beta_i * c_i
            
            final_colors.append(c_acc)
            
    return final_colors
```

## Operating characteristics
*   **Time Complexity:** Not explicitly given in the sources. The method involves multiple function evaluations ($f_a, f_b, f_c$) and texture lookups per pixel. The complexity depends on the number of resolution levels (4 in the implementation) [S1].
*   **Memory Complexity:** Depends on the size of the MAB, which stores multiple resolution levels. The sources do not specify the exact memory footprint [S1].
*   **Measured Performance:** The system is designed to maintain refresh rates above a set threshold by adjusting resolution levels on the fly. Performance fluctuations in VR can lead to cybersickness, which this method mitigates [S1].
*   **Strengths:**
    *   Maintains high frame rates in VR by adapting resolution to gaze distance [S1].
    *   Unnoticeable blending between resolution levels due to smooth-step functions [S1].
    *   Straightforward adjustment system for runtime performance [S1].
*   **Limitations:**
    *   Performance can still be impacted by parametric changes (e.g., transfer function modifications) [S1].
*   **When to use:** VR volume rendering where maintaining a high, stable frame rate is critical to prevent cybersickness [S1].
*   **When not to use:** Non-VR applications where gaze tracking is unavailable or frame rate stability is less critical [S1].

## Background
The method was presented in the paper "Embracing Raycasting for Virtual Reality" at WSCG 2022 [S1]. It addresses the challenge of maintaining high update rates in VR volume rendering, where camera and gaze fluctuations can cause performance issues leading to cybersickness [S1]. The technique uses a Multi-Anchor Buffer (MAB) and gaze-contingent blending to dynamically adjust resolution distribution [S1].

## Sources
[S1] Embracing Raycasting for Virtual Reality, https://doi.org/10.24132/csrn.3201.16, fetched 2026-10-29
