# Sketch-based Transfer Function Editing

Also called: Sketch-based TF Design, Sketch-based TF Design using Strokes, Sketch-based 1D Transfer Function Design (Strokes), Sketch-based Transfer Function Design (Slices)

## Overview
Sketch-based Transfer Function Editing allows users to define or refine classification functions by drawing strokes directly on spatial volume views or dedicated 1D/2D sketching surfaces, rather than manipulating abstract control points in a parameter space [S1, S2, unofficial]. It solves the cognitive load and precision issues of traditional widget-based editing by leveraging the user's spatial intuition and motor skills [S2, unofficial]. It sets itself apart from [2D Transfer Function Widget](5735-2d-transfer-function-widget.md) by prioritizing direct manipulation of the rendered image or free-form sketching over structured geometric widgets, and from [Image-based TF Design](7807-image-based-tf-design.md) by often generating the underlying transfer function parameters (e.g., via neural networks or stroke-to-function mapping) rather than just selecting regions in an image [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** User input strokes (2D coordinates on screen or volume slice), optionally combined with semantic labels (foreground/background) or color selections.
*   **Output:** A Transfer Function (TF) defined over the data domain (e.g., intensity, gradient magnitude). The TF maps input values to opacity $\alpha \in [0,1]$ and color $C \in [0,1]^3$.

**Data Structures:**
The specific internal representation depends on the variant:
1.  **Stroke List:** A sequence of points $(x_i, y_i)$ representing the user's drawing path.
2.  **Control Point Array:** A set of keypoints $(v_k, \alpha_k, C_k)$ defining a piecewise linear or spline function.
3.  **Neural Network Weights:** In systems using AI, a set of weights $W$ mapping sketch features to TF parameters [S1, unofficial].

**Steps:**
1.  **Capture Stroke:** The system records the user's input as a sequence of 2D coordinates $(u, v)$ on the display surface [S2, unofficial].
2.  **Interpretation:**
    *   *Direct Mapping:* The stroke is interpreted as a function graph. For example, in a 1D sketching interface, the horizontal axis maps to the data value domain (e.g., intensity), and the vertical axis maps to opacity or color [S2, unofficial].
    *   *Spatial Mapping:* If sketching on a volume slice, the stroke's position and color are mapped to the underlying voxel intensities at those locations. A neural network or heuristic then aggregates these spatial samples to construct a global 1D or multi-dimensional TF [S1, unofficial].
3.  **Refinement:** The user can iteratively add strokes or modify existing ones. The system updates the TF in real-time [S2, unofficial].
4.  **Evaluation:** The resulting TF is evaluated for each sample point during ray casting to determine optical properties [S2, unofficial].

**Formulas:**
*   *Linear Interpolation (Ramp-like):* For two nodes $(v_1, \alpha_1)$ and $(v_2, \alpha_2)$, the opacity at value $v$ is:
    $$ \alpha(v) = \alpha_1 + \frac{v - v_1}{v_2 - v_1} (\alpha_2 - \alpha_1) $$
    [S2, unofficial]
*   *Sliding Window:* A rectangular pulse $P(v)$ over interval $[a, b]$ with height $h$:
    $$ P(v) = \begin{cases} h & \text{if } a \le v \le b \\ 0 & \text{otherwise} \end{cases} $$
    [S2, unofficial]

**Parameters:**
*   **Stroke Width:** Not explicitly defined in sources, but implied by touch gesture sensitivity [S2, unofficial].
*   **Thumbnail Interval Length:** 10% of the full parameter domain [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Ambiguity:** Sketching on a 2D projection of a 3D volume can be ambiguous if multiple structures overlap. Systems mitigate this by allowing selection of specific slices or using multi-dimensional TFs [S1, unofficial].
*   **Precision:** Free-form sketching may lack the precision of widget-based editing for fine-tuning. Hybrid approaches (initial sketch + fine-tuning) are recommended [S2, unofficial].
*   **Touch vs. Mouse:** Touch gestures are preferred for accuracy in sketching TFs compared to mouse input [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Example: Voxel TIPs Sketching Tool Logic
def update_tf_from_stroke(stroke_points, domain_range, current_tf):
    """
    Updates the transfer function based on user strokes.
    stroke_points: List of (x_pixel, y_pixel) tuples.
    domain_range: Tuple (min_val, max_val) of the data domain.
    current_tf: Current piecewise linear TF.
    """
    new_points = []
    for x_px, y_px in stroke_points:
        # Map pixel coordinates to data domain
        # x_px maps to intensity/gradient value
        # y_px maps to opacity (0 to 1)
        val = map_linear(x_px, 0, screen_width, domain_range[0], domain_range[1])
        alpha = map_linear(y_px, 0, screen_height, 0.0, 1.0)
        new_points.append((val, alpha))
    
    # Sort by value
    new_points.sort(key=lambda p: p[0])
    
    # Merge with existing TF or replace, depending on tool mode
    # For sketching tool, it typically refines the current function
    refined_tf = merge_tf(current_tf, new_points)
    return refined_tf

def merge_tf(existing_tf, new_points):
    """
    Merges new sketch points into the existing piecewise linear TF.
    [G] Implementation detail: Interpolate new points into the existing structure,
    resolving overlaps by averaging or overwriting based on tool context.
    """
    # Combine all control points
    all_points = existing_tf.control_points + new_points
    # Sort and remove duplicates (averaging if necessary)
    all_points.sort(key=lambda p: p[0])
    # Return new piecewise linear function
    return PiecewiseLinearFunction(all_points)
```

## Subtopics

### [Interactive Volume Rendering with Multi-Dimensional TFs](6183-interactive-volume-rendering-with-multi-dimensional-tfs.md)
This method enables interactive volume rendering by allowing users to directly manipulate multi-dimensional transfer functions (TFs) to control material appearance. It solves the limitation of 1D TFs, which cannot accurately capture complex material boundaries defined by multiple data properties. It sets itself apart by using a 3D TF space (value, 1st derivative, 2nd derivative) where users can "paint" opacity directly into the TF domain by pointing at spatial features.

### Sliding Window Scanning
A technique to scan the intensity and gradient magnitude domains to sketch initial concepts for intensity-based and gradient-based transfer function terms [S2, unofficial]. It involves applying a sliding window (rectangular pulse) to scan the domain, followed by configuring multiple sliders to assign nonzero opacity values to the region of interest [S2, unofficial].

### Sketch-based Feature Fusion
A method where, based on user sketches on the rendered images, multiple features in distinct rendering results are fused into a comprehensive visualization [S1, unofficial]. It involves fusing features from different rendering results based on user sketches [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Real-time interaction is supported. Stroke processing is $O(N)$ where $N$ is the number of stroke points. TF evaluation remains $O(1)$ per sample point after construction [S2, unofficial].
*   **Memory Complexity:** Depends on the TF representation. Piecewise linear TFs are $O(K)$ where $K$ is the number of control points [S2, unofficial].
*   **Performance:** Users preferred touch gestures over mouse-based drawing for sketching TFs, finding them more accurate [S2, unofficial]. Completion times for TF design were measured in user studies (e.g., ~2-5 minutes per task in Voxel TIPs vs. VoTracer) [S2, unofficial].
*   **Strengths:** Intuitive, fast for initial concept generation, leverages spatial memory [S2, unofficial].
*   **Limitations:** Less precise for fine-tuning compared to widget-based methods; requires careful design to avoid ambiguity in spatial sketching [S1, S2, unofficial].
*   **When to Use:** Initial exploration, rapid prototyping, or when users have strong spatial intuition for the data [S2, unofficial].
*   **When Not to Use:** When high precision is required for specific intensity ranges, or when the data domain is high-dimensional and cannot be easily mapped to 2D sketches [S1, unofficial].

## Background
Originating from the need for more intuitive interfaces in volume rendering, sketch-based TF editing was proposed by researchers such as Ropinski et al. [80] and Tzeng et al. [90, 91] [S1, unofficial]. Tzeng et al. proposed systems using neural networks to generate high-dimensional TFs from sketches on volume slices [S1, unofficial]. Guo et al. developed a system for direct manipulation of optical properties via strokes [S1, unofficial]. Wu and Qu proposed feature fusion based on sketches [S1, unofficial]. Lopes et al. introduced Voxel TIPs, a multi-touch tabletop interface using sliding windows, thumbnails, and mixing tables for sketch-based TF design [S2, unofficial].

## Sources
[S1] MULTIVARIATE TRANSFER FUNCTION DESIGN, https://www.sci.utah.edu/~lzhou/dissertation_final_zhoul.pdf, fetched 2026-10-27 (unofficial)
[S2] Explicit design of transfer functions for volume-rendered images, https://web.tecnico.ulisboa.pt/daniel.s.lopes/papers/Explicit%20design%20of%20transfer%20functions%20for%20volume-rendered%20images%20-%20Vis%20Comp%202017.pdf, fetched 2026-10-29 (unofficial)
