# Sketch-based Transfer Function Editing

Also called: Sketch-based TF Design, Sketch-based TF Design using Strokes, Sketch-based 1D Transfer Function Design (Strokes), Sketch-based Transfer Function Design (Slices)

## Overview
Sketch-based Transfer Function Editing allows users to define volume rendering appearance by directly drawing strokes on spatial domain views (volume slices or rendered images) or via direct manipulation interfaces, which are then interpreted to generate or refine transfer functions (TFs) [S1, S2, unofficial]. It solves the disconnect between abstract 1D/2D TF widgets and the visual features in the volume, enabling intuitive assignment of color and transparency based on spatial context [S1, S2, unofficial]. It sets itself apart from [2D Transfer Function Widget](5735-2d-transfer-function-widget.md) by operating on spatial sketches rather than abstract TF graphs, and from [Image-based TF Design](7807-image-based-tf-design.md) by often employing neural networks or explicit stroke-to-TF mapping algorithms to interpret the sketches [S1, S2, unofficial].

## How it works
The method encompasses several distinct approaches found in literature, primarily categorized by the interaction medium (mouse/WIMP vs. multi-touch) and the interpretation mechanism (neural networks vs. direct geometric mapping).

### Inputs and Outputs
*   **Inputs:** User-drawn strokes on volume slices or rendered images [S1, unofficial]; touch gestures on a multi-touch surface [S2, unofficial]; volume dataset (scalar field) [S2, unofficial].
*   **Outputs:** A 1D or multi-dimensional transfer function mapping intensity/gradient values to color and opacity [S1, S2, unofficial]; updated volume rendering [S1, S2, unofficial].

### Data Structures
1.  **Stroke Data:** A sequence of 2D coordinates $(x, y)$ representing the user's input on the screen [S1, S2, unofficial].
2.  **Transfer Function Representation:**
    *   **Ramp-like:** Piecewise linear function defined by nodes $(v_i, o_i, c_i)$ where $v$ is value, $o$ is opacity, $c$ is color [S2, unofficial].
    *   **Sliding Window:** Rectangular pulse defined by interval $[v_{start}, v_{end}]$ and amplitude $A$ [S2, unofficial].
    *   **Mixing Table:** Set of vertical sliders, each associated with a rectangular pulse of fixed domain width and adjustable height [S2, unofficial].
    *   **Neural Network Model:** A trained artificial neural network (ANN) mapping sketch features to TF parameters [S1, unofficial].
3.  **Thumbnails:** Visual representations of data subsets, where each thumbnail represents voxels with intensity values in a bounded interval of length 10% of the full parameter domain [S2, unofficial].

### Steps

#### Approach 1: Neural Network Interpretation (Tzeng et al.)
1.  **Sketch Acquisition:** User draws strokes on volume slices to assign color and transparency [S1, unofficial].
2.  **Feature Extraction:** Extract features from the strokes (e.g., position, length, density) [S1, unofficial].
3.  **TF Generation:** Pass features through a pre-trained artificial neural network to generate high-dimensional transfer function parameters [S1, unofficial].
4.  **Rendering:** Apply the generated TF to the volume data [S1, unofficial].

#### Approach 2: Direct Stroke-to-TF Mapping (Ropinski et al.)
1.  **Stroke Definition:** User draws strokes to define foreground and background regions in the volume [S1, unofficial].
2.  **Layer Extraction:** Identify layers in the volume corresponding to the strokes [S1, unofficial].
3.  **Refinement:** Adjust color and opacity of each extracted layer to refine the 1D transfer function [S1, unofficial].

#### Approach 3: Multi-Touch Interface (Voxel TIPs)
1.  **Initialization:** Display volume rendering in upper area and TF editing tools in lower area [S2, unofficial].
2.  **Initial Concept Sketching:**
    *   **Sliding Window:** User drags cursors to adjust pulse height and width, then drags the window across the domain to scan intensity/gradient values [S2, unofficial].
    *   **Thumbnails:** User taps thumbnails to apply full opacity to the corresponding 10% intensity interval [S2, unofficial].
    *   **Mixing Table:** User adjusts vertical sliders to assign nonzero opacity values to regions of interest [S2, unofficial].
3.  **Fine-Tuning:**
    *   **Sketching Tool:** User draws free-form strokes on the TF graph to locally refine or globally define the function [S2, unofficial].
    *   **Ramp-like Function:** User moves linear interpolation nodes to adjust opacity ramps [S2, unofficial].
4.  **Color Assignment:**
    *   Double-tap color bar to evoke a handle and HSV color picker [S2, unofficial].
    *   Drag handle to set color interpolation between neighboring handles [S2, unofficial].
5.  **Geometric Transformation:** Apply touch gestures (rotation, scale, translation) to the volume view to aid spatial understanding [S2, unofficial].

### Formulas
*   **Sliding Window Opacity:**
    $$
    T(v) = \begin{cases} 
    A & \text{if } v_{start} \le v \le v_{end} \\
    0 & \text{otherwise}
    \end{cases}
    $$
    Where $A$ is the pulse height (amplitude), $v$ is the intensity/gradient value, and $[v_{start}, v_{end}]$ is the interval defined by the window [S2, unofficial].

*   **Ramp-like Interpolation:**
    Intermediate opacity values between nodes $i$ and $i+1$ are determined by linear interpolation:
    $$
    o(v) = o_i + \frac{o_{i+1} - o_i}{v_{i+1} - v_i} (v - v_i)
    $$
    Where $v_i, o_i$ are the value and opacity of node $i$ [S2, unofficial].

*   **Thumbnail Interval:**
    Each thumbnail represents an interval of length $L = 0.1 \times V_{max}$, where $V_{max}$ is the full parameter domain [S2, unofficial].

### Parameters
*   **Thumbnail Interval Length:** 10% of the full parameter domain [S2, unofficial].
*   **Mixing Table Sliders:** 10 equally sized sliders [S2, unofficial].
*   **Hardware:** Samsung UE55F8000 55” TV, ZaagTech X Series 16 Touch Overlay [S2, unofficial].

### Edge Cases and Pitfalls
*   **Tool Switching:** Switching from sketching to multiple sliders converts the function into a rectangular pulse train according to the average of the values in the range of each slider [S2, unofficial]. Switching from ramp-like to sketching leaves the function unchanged [S2, unofficial].
*   **Precision:** Touch gestures are preferred over mouse-based drawing for more accurate reproduction of user intentions [S2, unofficial].
*   **Degenerate Input:** Empty strokes or strokes outside the domain are ignored or handled as no-op [G].

## Subtopics

### [Interactive Volume Rendering with Multi-Dimensional TFs](6183-interactive-volume-rendering-with-multi-dimensional-tfs.md)
Enables users to define volume rendering appearance by directly manipulating a multi-dimensional transfer function (TF) in response to spatial interaction. It solves the limitation of 1D TFs, which cannot accurately capture complex material boundaries (e.g., distinguishing enamel, dentin, and pulp). It sets itself apart by allowing users to "paint" opacity and color into a 3D TF domain (value, 1st derivative, 2nd derivative) by pointing at spatial features, bridging the gap between the data space and the abstract TF domain.

### Sliding Window Scanning
A technique to scan the intensity and gradient magnitude domains to sketch initial concepts for intensity-based and gradient-based transfer function terms. It involves applying a sliding window to scan the domains and configuring multiple sliders to assign nonzero opacity values to the region of interest [S2, unofficial].

### Sketch-based Feature Fusion
A method where, based on user sketches on the rendered images, multiple features in distinct rendering results are fused into a comprehensive visualization. It involves fusing features from distinct rendering results based on user sketches [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** In user tests, Voxel TIPs (multi-touch) was preferred over VoTracer (mouse/WIMP) for both intensity-based (87%) and gradient-based (93%) function design [S2, unofficial]. Users found it more difficult to sketch with a mouse, preferring touch gestures that more accurately reproduce intentions [S2, unofficial].
*   **Strengths:** Intuitive interaction; direct manipulation of visual features; supports rapid initial concept design and fine-tuning [S1, S2, unofficial].
*   **Limitations:** Relies on user interpretation of sketches; neural network approaches require training data [S1, S2, unofficial].
*   **When to use:** When users need to intuitively map visual features in the volume to TF parameters without navigating abstract widgets [S1, S2, unofficial].
*   **When not to use:** When precise, parametric control is required without spatial context [G].

## Background
Originated from the need for more intuitive transfer function design interfaces. Tzeng et al. proposed using neural networks to generate high-dimensional TFs from sketches [S1, unofficial]. Ropinski et al. introduced a sketch-based UI for 1D TF design using strokes to define layers [S1, unofficial]. Guo et al. developed a system for direct manipulation of optical properties via strokes [S1, unofficial]. Lopes et al. developed Voxel TIPs, a multi-touch interface combining sliding windows, thumbnails, mixing tables, and sketching tools [S2, unofficial].

## Sources
[S1] MULTIVARIATE TRANSFER FUNCTION DESIGN, https://www.sci.utah.edu/~lzhou/dissertation_final_zhoul.pdf, fetched 2026-10-27 (unofficial)
[S2] Explicit design of transfer functions for volume-rendered images, https://web.tecnico.ulisboa.pt/daniel.s.lopes/papers/Explicit%20design%20of%20transfer%20functions%20for%20volume-rendered%20images%20-%20Vis%20Comp%202017.pdf, fetched 2026-10-29 (unofficial)
