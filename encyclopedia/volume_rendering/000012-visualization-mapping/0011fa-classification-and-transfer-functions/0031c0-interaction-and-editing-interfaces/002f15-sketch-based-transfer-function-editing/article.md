# Sketch-based Transfer Function Editing

Also called: Sketch-based TF Design, Sketch-based TF Design using Strokes, Sketch-based 1D Transfer Function Design (Strokes), Sketch-based Transfer Function Design (Slices)

## Overview
Sketch-based Transfer Function (TF) Editing allows users to define or modify classification functions by drawing strokes directly on the volume rendering view or on dedicated 1D/2D parameter spaces, rather than manipulating abstract widgets [S1, unofficial][S2, unofficial]. Unlike [Automated Classification Widget Placement](5645-automated-classification-widget-placement.md) which optimizes widget positions, or [2D Transfer Function Widget](5735-2d-transfer-function-widget.md) which uses geometric handles on a 2D plot, sketch-based methods prioritize direct manipulation and intuitive mapping between visual intent and parameter space [S2, unofficial]. It solves the difficulty of manually tuning numerical parameters by enabling rapid, global definition of TF shapes and local refinement through free-form drawing [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** User-drawn strokes (2D coordinates in screen space or parameter space), initial volume data (scalar field $f(\mathbf{x})$ and optionally gradient magnitude $\|\nabla f(\mathbf{x})\|$), and current TF state.
*   **Output:** A modified Transfer Function mapping scalar/gradient values to opacity $\alpha$ and color $C$ [S1, unofficial][S2, unofficial].

**Data Structures:**
The TF is typically represented as a piecewise linear function defined by keypoints (nodes) in the parameter domain (intensity or gradient magnitude) [S2, unofficial]. In systems like Voxel TIPs, the domain is discretized into segments for specific tools (e.g., 10 equal segments for sliders) [S2, unofficial].

**Steps:**
1.  **Sketch Acquisition:** The user draws strokes on the interface.
    *   *Spatial Sketching:* Strokes are drawn on the rendered volume image. The system maps screen coordinates to volume data features or parameter values [S1, unofficial].
    *   *Parameter Sketching:* Strokes are drawn on a 1D graph (intensity/gradient axis) or 2D histogram [S2, unofficial].
2.  **Interpretation/Conversion:** The raw strokes are converted into TF parameters.
    *   *Direct Mapping:* In 1D sketching, the vertical position of the stroke relative to the axis determines opacity, and horizontal position determines the scalar value [S2, unofficial].
    *   *Neural Network Generation:* In some systems (e.g., Tzeng et al.), strokes on volume slices are fed into an artificial neural network to generate high-dimensional TFs [S1, unofficial].
    *   *Feature Fusion:* Strokes on rendered images can fuse multiple features from distinct rendering results into a comprehensive visualization [S1, unofficial].
3.  **Refinement:** Users iteratively adjust the TF using specific interaction modes:
    *   *Sliding Window:* A rectangular pulse with controlled amplitude scans the domain. Opacity is constant within the interval and zero outside. Height and width are adjusted via cursors [S2, unofficial].
    *   *Mixing Table:* Vertical sliders associated with rectangular pulses. Ten equally sized sliders are placed side-by-side. Sliders are dragged to set height (opacity). Thumbnails act as toggle buttons for opacity (0 or 1) [S2, unofficial].
    *   *Ramp-like Function:* Piecewise linear interpolation between nodes. Nodes are moved vertically to adjust opacity [S2, unofficial].
    *   *Free-form Sketching:* Direct drawing on the TF graph updates the function values at the selected domain points [S2, unofficial].
4.  **Adaptation:** When switching tools, the prior TF is adapted to the new tool type. For example, switching from sketching to multiple sliders converts the function into a rectangular pulse train based on the average values in each slider's range [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_tf_from_stroke(stroke_points, current_tf, tool_mode):
    """
    Updates the transfer function based on user strokes.
    stroke_points: List of (x, y) coordinates from user input.
    current_tf: Current piecewise linear TF defined by nodes.
    tool_mode: 'sliding_window', 'mixing_table', 'ramp', 'sketch', 'spatial'.
    """
    if tool_mode == 'sketch':
        # Direct mapping: x is scalar value, y is opacity
        new_nodes = []
        for x, y in stroke_points:
            # Normalize x to domain [0, 1], y to opacity [0, 1]
            scalar = normalize(x, domain_min, domain_max)
            opacity = normalize(y, 0, 1)
            new_nodes.append(Node(scalar, opacity))
        # Interpolate or replace existing nodes
        current_tf = interpolate_nodes(new_nodes)
        
    elif tool_mode == 'sliding_window':
        # Define a rectangular pulse
        # stroke_points define the window boundaries and height
        x_start, x_end, height = parse_window_stroke(stroke_points)
        current_tf = apply_rectangular_pulse(current_tf, x_start, x_end, height)
        
    elif tool_mode == 'mixing_table':
        # Map stroke to nearest slider bin
        bin_index = get_slider_bin(stroke_points[0].x)
        opacity = normalize(stroke_points[0].y, 0, 1)
        current_tf.set_slider_height(bin_index, opacity)
        
    elif tool_mode == 'spatial':
        # Map screen stroke to volume features
        # Requires ray casting to map screen pixel to volume scalar
        volume_scalars = map_screen_to_volume(stroke_points)
        # Use NN or heuristic to generate TF from scalars
        current_tf = generate_tf_from_scalars(volume_scalars)
        
    return current_tf
```

**Parameters:**
*   **Domain Resolution:** In Voxel TIPs, the mixing table uses 10 equally sized sliders [S2, unofficial].
*   **Thumbnail Interval:** Thumbnails represent bounded intervals of length 10% of the full parameter domain [S2, unofficial].

**Edge Cases:**
*   **Tool Switching:** Switching from sketching to sliders requires averaging values to create a rectangular pulse train, which may lose fine details [S2, unofficial].
*   **Precision:** Touch gestures on large screens (e.g., 55” TV) may have lower precision than mouse input, though users prefer them for intent reproduction [S2, unofficial].

## Subtopics

### [Interactive Volume Rendering with Multi-Dimensional TFs](6183-interactive-volume-rendering-with-multi-dimensional-tfs.md)
This method enables users to specify three-dimensional transfer functions (TFs) by interacting directly with a 2D histogram of scalar values versus gradient magnitudes. It solves the limitation of 1D TFs, which cannot accurately capture material boundaries where different materials share similar scalar values but differ in gradient characteristics (e.g., enamel vs. dentin). By allowing users to "paint" opacity regions in this 2D domain using geometric widgets, it provides a more intuitive specification process than manual parameter tuning.

### Sliding Window Scanning
A technique to scan the intensity and gradient magnitude domains to sketch initial concepts for intensity-based and gradient-based transfer function terms [S2, unofficial]. It applies a sliding window (rectangular pulse) to scan the domain, followed by configuring multiple sliders to assign nonzero opacity values to the region of interest [S2, unofficial].

### Sketch-based Feature Fusion
A method where, based on user sketches on the rendered images, multiple features in distinct rendering results are fused into a comprehensive visualization [S1, unofficial]. Wu and Qu [99] propose fusing these features directly from spatial sketches [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Real-time interaction is supported. Stroke processing is $O(N)$ where $N$ is the number of stroke points. Rendering updates depend on the ray marching algorithm [S2, unofficial].
*   **Memory Complexity:** Low. TFs are stored as small sets of keypoints or LUTs. Thumbnails require additional memory for pre-rendered particle representations [S2, unofficial].
*   **Performance:** Voxel TIPs was tested on a Samsung UE55F8000 TV with ZaagTech X Series 16 Touch Overlay [S2, unofficial]. User tests showed that touch gestures were preferred over mouse-based drawing for sketching TFs, with users finding it more difficult to sketch with a mouse [S2, unofficial].
*   **Strengths:** Intuitive, rapid initial definition, supports global and local editing, leverages direct manipulation metaphors (mixing table, sliding window) [S2, unofficial].
*   **Limitations:** Precision may be lower on large touch surfaces compared to mouse. Spatial sketching requires complex mapping from screen to volume parameters [S1, unofficial][S2, unofficial].
*   **When to use:** When users need to quickly explore TFs or have difficulty with abstract widget manipulation. Suitable for collaborative or standing interactions (tabletops) [S2, unofficial].
*   **When not to use:** When high-precision, fine-grained control over specific TF segments is required without iterative refinement [S2, unofficial].

## Background
Sketch-based TF editing emerged as an alternative to traditional WIMP (Window, Icon, Menu, Pointer) interfaces, which were often considered tedious for TF design [S2, unofficial]. Key developments include:
*   **Tzeng et al.:** Proposed systems allowing users to sketch volume slices to assign color and transparency, generating high-dimensional TFs using artificial neural networks [S1, unofficial].
*   **Ropinski et al.:** Proposed a sketch-based UI for 1D TF design where users draw strokes to define foreground/background layers, refined by adjusting color and opacity [S1, unofficial].
*   **Guo et al.:** Developed a system enabling direct manipulation of optical properties on the volume rendering view via user-drawn strokes, intelligently defining 1D TFs [S1, unofficial].
*   **Wu and Qu:** Proposed fusing multiple features from distinct rendering results based on user sketches on rendered images [S1, unofficial].
*   **Lopes et al. (Voxel TIPs):** Developed a sketch-based interface for a multi-touch tabletop, combining histograms, thumbnails, sliding windows, and mixing tables for intensity and gradient-based TF design [S2, unofficial].

## Sources
[S1] MULTIVARIATE TRANSFER FUNCTION DESIGN, https://www.sci.utah.edu/~lzhou/dissertation_final_zhoul.pdf, fetched 2026-10-27 (unofficial)
[S2] Explicit design of transfer functions for volume-rendered images, https://web.tecnico.ulisboa.pt/daniel.s.lopes/papers/Explicit%20design%20of%20transfer%20functions%20for%20volume-rendered%20images%20-%20Vis%20Comp%202017.pdf, fetched 2026-10-29 (unofficial)
