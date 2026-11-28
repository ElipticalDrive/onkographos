# Peak-based Transfer Function Model
Also called: Peak model, Sum of window functions
## Overview
The Peak-based Transfer Function Model defines a 1D transfer function as the sum of localized window functions called "peaks," where each peak highlights a specific voxel value range with a distinct color [S1, unofficial]. It solves the problem of fine-grained control required by piecewise linear models, offering a more intuitive interface for hand-held devices by reducing the number of parameters a user must modify [S1, unofficial]. Unlike its sibling [Gaussian Sum Transfer Function](1812-gaussian-sum-transfer-function.md), which uses Gaussian functions, this model uses sinusoidal window functions that are zero outside their defined support, providing compact support and distinct color highlighting [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: A scalar voxel value $x \in [0, 1]$.
- Input: A set of $n$ peaks, where each peak $i$ is defined by center $c_i$, width $w_i$, height $h_i$, and color $C_i$.
- Output: A final color $T(x)$ and opacity $\alpha(x)$ computed via alpha blending.

**Data Structures:**
Each peak $i$ stores:
- $c_i$: Center of the peak (scalar).
- $w_i$: Half-width of the peak (scalar).
- $h_i$: Maximum height (opacity contribution) of the peak (scalar).
- $C_i$: Color associated with the peak (e.g., RGB vector).

**Formulas:**
A single peak $p_i(x)$ is defined as [S1, unofficial]:
$$
p_i(x) = \begin{cases} 
h_i \cdot \sin\left(\frac{\pi}{2w_i}(x - c_i + w_i)\right) & \text{if } c_i - w_i \leq x \leq c_i + w_i \\
0 & \text{otherwise}
\end{cases}
$$
The total opacity contribution at $x$ is the sum of all peak values:
$$
\alpha_{total}(x) = \sum_{i=1}^{n} p_i(x)
$$
The final color $T(x)$ is computed by alpha-blending the peak colors. Let $C_{bg}$ be the background color (typically black or transparent). The blending is performed sequentially or via standard alpha compositing where the opacity of each peak is $p_i(x)$ [S1, unofficial].

**Parameters:**
- Number of peaks $n$: In practical use, $n < 8$ is recommended. More peaks tend to clutter the visualization result [S1, unofficial].

**Pitfalls:**
- Sensitivity to width changes: The visualization result is especially sensitive to changes in peak widths $w$. Users are advised to first define the position ($c$) and height ($h$) of a peak, and then fine-tune the result by adjusting its width ($w$) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_peak_function(x, peaks):
    """
    Evaluates the peak-based transfer function for a given voxel value x.
    
    Args:
        x: Scalar voxel value.
        peaks: List of peak dictionaries, each containing:
               'c': center, 'w': width, 'h': height, 'color': (r, g, b)
    
    Returns:
        final_color: Tuple (r, g, b)
        final_alpha: Float in [0, 1]
    """
    # Initialize accumulated color and alpha
    acc_r, acc_g, acc_b = 0.0, 0.0, 0.0
    acc_alpha = 0.0
    
    # Background color assumption (e.g., black)
    bg_r, bg_g, bg_b = 0.0, 0.0, 0.0
    
    # Sort peaks? The source does not specify an order. 
    # Standard alpha blending order matters. We assume the order in the list.
    
    for peak in peaks:
        c = peak['c']
        w = peak['w']
        h = peak['h']
        cr, cg, cb = peak['color']
        
        # Calculate peak value p(x)
        if c - w <= x <= c + w:
            # Avoid division by zero if w is 0
            if w == 0:
                p_x = 0.0
            else:
                p_x = h * math.sin((math.pi / (2 * w)) * (x - c + w))
        else:
            p_x = 0.0
            
        # Alpha blend this peak's color with the accumulated result
        # Standard alpha compositing: C_out = C_fg * alpha_fg + C_bg * (1 - alpha_fg)
        # Here, the "foreground" is the peak color with opacity p_x
        # The "background" is the currently accumulated color
        
        alpha_fg = p_x
        # Clamp alpha to [0, 1] just in case, though sin range and h should handle it
        alpha_fg = max(0.0, min(1.0, alpha_fg))
        
        new_r = cr * alpha_fg + acc_r * (1.0 - alpha_fg)
        new_g = cg * alpha_fg + acc_g * (1.0 - alpha_fg)
        new_b = cb * alpha_fg + acc_b * (1.0 - alpha_fg)
        new_alpha = alpha_fg + acc_alpha * (1.0 - alpha_fg)
        
        acc_r, acc_g, acc_b = new_r, new_g, new_b
        acc_alpha = new_alpha
        
    return (acc_r, acc_g, acc_b), acc_alpha
```
## Subtopics

### Peak Parameter Adjustment via Controller Axes
This variant describes the specific interaction mapping for adjusting peak parameters using a Move controller in a Virtual Reality environment [S1, unofficial]. To change the center $c$, width $w$, and height $h$ of a peak, the user employs the Move controller. The user must choose (using buttons on the Move controller) to adjust either ($c$ and $h$) or $w$. This ensures both adjustments take place in the x/y-plane. Using the z-axis for parameter adjustment proved to be unintuitive and difficult to control, so it was excluded from the final mapping [S1, unofficial]. The separation of $w$ from $c$ and $h$ addresses the sensitivity to width changes, allowing users to first define position and height, then fine-tune width [S1, unofficial].

## Operating characteristics
- **Complexity:** Time complexity is $O(n)$ per voxel, where $n$ is the number of peaks. Memory complexity is $O(n)$ to store peak parameters.
- **Performance:** No specific hardware benchmarks are provided in the sources.
- **Strengths:** Reduces the number of parameters a user has to modify compared to piecewise linear building blocks. Allows more intuitive manipulation using hand-held devices that favor coarser control movements [S1, unofficial].
- **Limitations:** The visualization result is especially sensitive to changes of peak widths [S1, unofficial]. More than 8 peaks tend to clutter the visualization result [S1, unofficial].
- **When to use:** When using hand-held VR controllers for transfer function editing, where coarse control is preferred over fine-grained mouse-based control [S1, unofficial].
## Background
The Peak-based Transfer Function Model was introduced in the context of user interfaces for volume rendering in Virtual Reality environments [S1, unofficial]. It was developed to address the limitations of piecewise linear transfer functions, which require very fine-grained control typically using a mouse-based interface [S1, unofficial]. The model uses a sum of window functions (peaks) to allow more intuitive manipulation using hand-held devices [S1, unofficial].
## Sources
[S1] User Interface for Volume Rendering in Virtual Reality Environments, https://arxiv.org/abs/1302.2024v1, fetched 2026-10-26 (unofficial)
