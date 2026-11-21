# Transfer Function Editor (Alpha Pins)

Also called: Alpha Pins, Colour Pins, Pin-based Transfer Function Editor

## Overview
The Transfer Function Editor (Alpha Pins) is an interactive visualization tool that maps scalar voxel values $S(x,y,z)$ to color vectors $F(x,y,z)$ and opacity $\alpha_F(x,y,z)$ [S1]. It distinguishes itself from other classification methods by using a pin-based interface where users manipulate specific geometric primitives (ramps, hats, blanks) to define opacity and vertical color markers to define gradients, rather than editing a continuous curve or parametric function directly [S1].

## How it works
**Inputs and Outputs**
*   **Input:** Scalar voxel values $S$ from the volume data domain (e.g., $[0, 255]$ for 8-bit data) [S1].
*   **Output:** A 4-tuple $(R, G, B, \alpha)$ for each scalar value, where $R,G,B \in [0, 255]$ and $\alpha \in [0, 1]$ [S1].

**Data Structures**
The editor maintains two distinct sets of control points:
1.  **Alpha Pins:** Defined in a top panel where the Y-axis represents opacity $[0, 1]$ [S1]. Supported types include straight lines (ramps) with controllable slope and position, trapezoidal functions (hats) with controllable height, width, and edge slope (special cases: tophat, triangle), and blanks with controllable width and position [S1].
2.  **Colour Pins:** Vertical dashed lines in the color mapping panel [S1]. Each colour pin defines an RGB color value in the range $[0, 255]$ at a specific scalar position [S1].

**Algorithm**
The transfer function $T(S)$ is computed in two stages: opacity composition and color interpolation [S1].
1.  **Opacity Composition:** For a given scalar value $S$, determine the opacity contribution $\alpha_i(S)$ from each alpha pin $i$ [S1].
    *   If $S$ falls within the domain of a **Blank** pin, $\alpha_{blank}(S) = 0$ [S1].
    *   If $S$ falls within the domain of a **Ramp** or **Hat** pin, $\alpha_{pin}(S)$ is calculated based on the pin's geometric parameters (slope, height, width) [S1].
    *   The total opacity $\alpha_F(S)$ is the maximum of all individual pin opacities, except that blanks (totally transparent) have precedence over all other pins [S1].
2.  **Color Interpolation:** Colours are linearly interpolated between the colour pins [S1].
    *   Sort the colour pins by their scalar position $p_j$ [G].
    *   Find the interval $[p_j, p_{j+1}]$ such that $p_j \le S < p_{j+1}$ [G].
    *   Let $C_j = (R_j, G_j, B_j)$ and $C_{j+1} = (R_{j+1}, G_{j+1}, B_{j+1})$ be the colors of the adjacent pins [G].
    *   Compute the normalized position $t$ within the interval: $t = \frac{S - p_j}{p_{j+1} - p_j}$ [G].
    *   Interpolate the color components linearly: $F(S) = C_j + t \cdot (C_{j+1} - C_j)$ [G].

**Parameters**
*   **X-axis range:** Scalar data domain, e.g., $[0, 255]$ for 8-bit data [S1].
*   **Y-axis range (opacity panel):** Opacity in the range $[0, 1]$ [S1].
*   **Colour pin values:** Red, green, and blue values in the range $[0, 255]$ [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_transfer_function(scalar_value: float, alpha_pins: list, color_pins: list) -> tuple:
    """
    Computes RGBA for a scalar value using the Alpha Pins method.
    
    Args:
        scalar_value: The voxel scalar S.
        alpha_pins: List of dicts with keys: 'type' ('ramp', 'hat', 'blank'), 
                    'pos' (scalar position), 'width' (for hats/blanks), 
                    'height' (for hats), 'slope' (for ramps).
        color_pins: List of dicts with keys: 'pos' (scalar position), 
                    'r', 'g', 'b' (integers 0-255).
                    
    Returns:
        (r, g, b, alpha) tuple.
    """
    # 1. Compute Opacity
    max_alpha = 0.0
    is_blank = False
    
    for pin in alpha_pins:
        pin_type = pin['type']
        pos = pin['pos']
        
        if pin_type == 'blank':
            # Blanks have precedence over all other pins [S1]
            # Assuming 'width' defines the extent from pos
            if pos <= scalar_value < pos + pin.get('width', 0):
                is_blank = True
                break
        elif pin_type == 'ramp':
            # Linear ramp from pos with given slope [G]
            slope = pin.get('slope', 0.0)
            alpha_val = max(0.0, min(1.0, slope * (scalar_value - pos)))
            if alpha_val > max_alpha:
                max_alpha = alpha_val
        elif pin_type == 'hat':
            # Trapezoidal hat: rises from pos, plateaus, falls [G]
            width = pin.get('width', 0.0)
            height = pin.get('height', 1.0)
            edge_slope = pin.get('edge_slope', 1.0)
            rise_width = height / edge_slope if edge_slope > 0 else 0
            fall_start = pos + width - rise_width
            fall_end = pos + width
            if pos <= scalar_value < pos + rise_width:
                alpha_val = min(height, edge_slope * (scalar_value - pos))
            elif pos + rise_width <= scalar_value < fall_start:
                alpha_val = height
            elif fall_start <= scalar_value < fall_end:
                alpha_val = max(0.0, height - edge_slope * (scalar_value - fall_start))
            else:
                alpha_val = 0.0
            if alpha_val > max_alpha:
                max_alpha = alpha_val

    if is_blank:
        alpha = 0.0
    else:
        alpha = max_alpha

    # 2. Compute Color
    # Sort colour pins by their scalar position [G]
    sorted_pins = sorted(color_pins, key=lambda p: p['pos'])
    
    r, g, b = 0, 0, 0
    
    if not sorted_pins:
        return (0, 0, 0, alpha)
        
    # Find interval [p_j, p_{j+1}] such that p_j <= S < p_{j+1} [G]
    if scalar_value <= sorted_pins[0]['pos']:
        pin = sorted_pins[0]
        return (pin['r'], pin['g'], pin['b'], alpha)
    elif scalar_value >= sorted_pins[-1]['pos']:
        pin = sorted_pins[-1]
        return (pin['r'], pin['g'], pin['b'], alpha)
    else:
        for i in range(len(sorted_pins) - 1):
            p1 = sorted_pins[i]
            p2 = sorted_pins[i+1]
            if p1['pos'] <= scalar_value < p2['pos']:
                t = (scalar_value - p1['pos']) / (p2['pos'] - p1['pos'])
                r = int(p1['r'] + t * (p2['r'] - p1['r']))
                g = int(p1['g'] + t * (p2['g'] - p1['g']))
                b = int(p1['b'] + t * (p2['b'] - p1['b']))
                break
                
    return (r, g, b, alpha)
```

**Edge Cases and Pitfalls**
*   **Blanks Precedence:** If a blank overlaps a ramp, the result is transparent, not the ramp's opacity [S1].
*   **Domain:** The X-axis range must match the scalar data domain (e.g., $[0, 255]$) [S1].
*   **Precision:** Not given in the sources.

## Operating characteristics
*   **Complexity:** Time complexity is $O(N)$ where $N$ is the number of alpha pins and color pins per voxel evaluation [G]. Memory complexity is proportional to the number of pins stored in the editor [G].
*   **Performance:** The method is computationally lightweight per voxel, relying on simple comparisons and linear interpolation [G].
*   **Strengths:** Intuitive control over specific features (ramps, blanks) allows users to isolate signal from noise effectively [S1].
*   **Limitations:** Requires manual placement of pins; not suitable for automatic classification without user input [G].

## Background
The method was introduced in the context of visualizing large astronomical cubes using a distributed-data perspective shear-warp volume rendering algorithm [S1]. It was designed to provide an effective way to map scalar data to color and opacity by leveraging strong color gradients and moderate opacity for interesting signal parts, while using gradual gradients and low/zero opacities for noise [S1].

## Sources
[S1] "A Distributed-Data Implementation of the Perspective Shear-Warp Volume Rendering Algorithm for Visualisation of Large Astronomical Cubes", http://web.eng.ucsd.edu/~jschulze/projects/Beeson2003.pdf, fetched 2026-10-29
