# Sketch-based Transfer Function Editing

Also called: Sketch-based TF Design, Sketch-based TF Design using Strokes, Sketch-based 1D Transfer Function Design (Strokes), Sketch-based Transfer Function Design (Slices)

## Overview
Sketch-based Transfer Function Editing allows users to define volume rendering appearance by drawing strokes directly on spatial views (volume slices or rendered images) or via dedicated sketching interfaces, which are then automatically converted into mathematical transfer functions [S1, unofficial][S2, unofficial]. It solves the difficulty of manually adjusting numerical parameters in traditional interfaces by leveraging the user's intuitive understanding of spatial features [S1, unofficial]. It sets itself apart from [2D Transfer Function Widget](5735-2d-transfer-function-widget.md) by mapping interaction to the spatial domain (pixels/voxels) rather than the abstract parameter domain, and from [Image-based TF Design](7807-image-based-tf-design.md) by explicitly generating the underlying classification function from the sketch geometry rather than just using the image as a reference [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** User-drawn strokes (2D coordinates in screen space or spatial coordinates in volume space) [S1, unofficial][S2, unofficial].
*   **Output:** A 1D or multi-dimensional transfer function $T(v)$ mapping scalar values $v$ (intensity or gradient magnitude) to opacity $\alpha \in [0, 1]$ and color $C \in [0, 1]^3$ [S1, unofficial][S2, unofficial].

**Data Structures:**
The transfer function is typically represented as a piecewise linear function defined by a set of keypoints $(v_i, \alpha_i, C_i)$ or as a lookup table (LUT) [S2, unofficial]. In systems like Voxel TIPs, the interface maintains a state for each editing mode (sliding window, mixing table, ramp-like, sketching) [S2, unofficial].

**Steps:**
1.  **Sketch Acquisition:** The user draws strokes on the interface.
    *   *Spatial Sketching:* Strokes are drawn on volume slices or the rendered image [S1, unofficial].
    *   *Interface Sketching:* Strokes are drawn on a dedicated 1D graph area or via specific tools (sliding window, sliders) [S2, unofficial].
2.  **Feature Extraction/Mapping:**
    *   If sketching on spatial views, the system analyzes the stroke locations to identify intensity/gradient ranges corresponding to the drawn features [S1, unofficial].
    *   If using tools like the Sliding Window, a rectangular pulse is defined over a selected domain interval [S2, unofficial].
    *   If using the Mixing Table, vertical sliders define rectangular pulses with adjustable height [S2, unofficial].
3.  **Transfer Function Generation:**
    *   *Direct Mapping:* For 1D TFs, strokes are converted into opacity/color ramps. For example, a stroke defining a foreground layer sets opacity to 1 and color to the selected value within the corresponding intensity range [S1, unofficial].
    *   *Neural Network:* In some systems, high-dimensional TFs are generated using artificial neural networks based on the sketches [S1, unofficial].
    *   *Fusion:* Multiple features from distinct rendering results can be fused into a comprehensive visualization based on user sketches [S1, unofficial].
4.  **Refinement:** The user can manually fine-tune the function via free-form sketching or by adjusting handles (color, opacity) [S2, unofficial].
5.  **Evaluation:** The resulting TF is applied during volume rendering (pre-classification or post-classification) [S2, unofficial].

**Formulas:**
*   **Sliding Window:** A rectangular pulse $P(v)$ with controlled amplitude $A$ and width $W$ centered at $v_c$:
    $$
    P(v) = \begin{cases} 
    A & \text{if } |v - v_c| \le W/2 \\
    0 & \text{otherwise}
    \end{cases}
    $$
    [S2, unofficial]
*   **Mixing Table:** A sum of rectangular pulses $R_i(v)$ associated with sliders $i$:
    $$
    T(v) = \sum_{i} H_i \cdot R_i(v)
    $$
    where $H_i$ is the height (opacity) of the $i$-th slider [S2, unofficial].
*   **Ramp-like Function:** Piecewise linear interpolation between nodes $(v_i, \alpha_i)$:
    $$
    \alpha(v) = \alpha_i + \frac{\alpha_{i+1} - \alpha_i}{v_{i+1} - v_i} (v - v_i) \quad \text{for } v_i \le v < v_{i+1}
    $$
    [S2, unofficial]

**Parameters:**
*   **Thumbnail Interval Length:** 10% of the full parameter domain [S2, unofficial].
*   **Mixing Table Sliders:** 10 equally sized sliders [S2, unofficial].
*   **Hardware:** Samsung UE55F8000 55” TV, ZaagTech X Series 16 Touch Overlay [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Tool Switching:** Switching from sketching to multiple sliders converts the function into a rectangular pulse train according to the average of values in the range of each slider [S2, unofficial]. Switching from ramp-like to sketching leaves the function unchanged [S2, unofficial].
*   **Precision:** Touch gestures are preferred over mouse-based drawing for more accurate reproduction of user intentions [S2, unofficial].

## Subtopics

### [Interactive Volume Rendering with Multi-Dimensional TFs](6183-interactive-volume-rendering-with-multi-dimensional-tfs.md)
This method enables users to define volume rendering appearance by directly manipulating a multi-dimensional transfer function (TF) in response to spatial interaction. It solves the limitation of 1D TFs, which cannot accurately capture complex material boundaries (e.g., distinguishing enamel, dentin, and pulp). It sets itself apart by allowing users to "paint" opacity and color into a 3D TF domain (value, 1st derivative, 2nd derivative) by pointing at spatial features, bridging the gap between the data space and the abstract TF domain.

### Sliding Window Scanning
A technique to scan the intensity and gradient magnitude domains to sketch initial concepts for intensity-based and gradient-based transfer function terms. It applies a sliding window to scan the domains and configures multiple sliders to assign nonzero opacity values to the region of interest [S2, unofficial].

### Sketch-based Feature Fusion
A method where, based on user sketches on the rendered images, multiple features in distinct rendering results are fused into a comprehensive visualization [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** In user tests with 15 participants, Voxel TIPs (sketch-based) was compared to VoTracer (WIMP/mouse). Users found it more difficult to sketch a function with a mouse, preferring touch gestures that more accurately reproduce user intentions [S2, unofficial]. Users preferred Voxel TIPs for both intensity-based (87%) and gradient-based (93%) function design over the conventional mouse approach [S2, unofficial].
*   **Strengths:** Intuitive interaction; rapid design of initial concepts; supports both intensity and gradient-based TFs; integrates histograms and thumbnails for context [S2, unofficial].
*   **Limitations:** Relies on user's ability to map spatial features to data values; may require refinement for complex TFs [S2, unofficial].
*   **When to use:** When users have a clear visual idea of the desired appearance and want to quickly prototype TFs based on spatial features [S2, unofficial].
*   **When not to use:** When precise numerical control over the TF shape is required without spatial reference [S2, unofficial].

## Background
Originating from research in interactive volume rendering, sketch-based TF editing was proposed by researchers such as Ropinski et al. [80], Guo et al. [30], Tzeng et al. [90, 91], and Wu and Qu [99] [S1, unofficial]. A notable implementation is Voxel TIPs, developed by Lopes et al., which uses a multi-touch tabletop interface [S2, unofficial]. The method is used in medical imaging and scientific visualization to facilitate the design of transfer functions by domain experts who may not be familiar with abstract parameter spaces [S2, unofficial].

## Sources
[S1] MULTIVARIATE TRANSFER FUNCTION DESIGN, https://www.sci.utah.edu/~lzhou/dissertation_final_zhoul.pdf, fetched 2026-10-27 (unofficial)
[S2] Explicit design of transfer functions for volume-rendered images, https://web.tecnico.ulisboa.pt/daniel.s.lopes/papers/Explicit%20design%20of%20transfer%20functions%20for%20volume-rendered%20images%20-%20Vis%20Comp%202017.pdf, fetched 2026-10-29 (unofficial)
