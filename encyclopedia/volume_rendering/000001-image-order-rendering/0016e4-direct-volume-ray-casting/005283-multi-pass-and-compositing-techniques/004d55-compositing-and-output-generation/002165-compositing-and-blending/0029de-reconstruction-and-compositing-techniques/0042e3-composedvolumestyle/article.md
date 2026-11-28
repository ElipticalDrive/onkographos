# ComposedVolumeStyle

Also called: CompositeVolumeStyle

## Overview
ComposedVolumeStyle is a rendering style node that allows compositing multiple volume rendering styles together into a single rendering pass by listing them in its `styles` field [S1, S2, unofficial]. It solves the problem of combining distinct visual effects—such as lighting, edge enhancement, and opacity mapping—into one coherent image without requiring multiple separate rendering passes [S1, S2, unofficial]. It sets itself apart from sibling methods like [Composition via Convolution (Power Form)](1591-composition-via-convolution-power-form.md) by operating at the style node level, aggregating stylistic contributions rather than mathematically convolving transmittance or color values directly in the ray-casting loop.

## How it works
**Inputs and Outputs:**
The method takes a list of `X3DVolumeRenderStyleNode` instances as input via the `styles` field. It outputs a single composite rendering pass where the visual effects of the constituent styles are applied to the volume data [S1, S2, unofficial].

**Data Structures:**
The node is defined by the following fields:
*   `enabled`: `SFBool`, default `TRUE`. Enables or disables the style [S1, unofficial].
*   `ordered`: `SFBool`, default `FALSE`. Determines if styles must be applied strictly in the order declared [S1, unofficial].
*   `styles`: `MFNode` (list of `X3DVolumeRenderStyleNode`). Contains the contributing style node references [S1, unofficial].

**Steps:**
1.  Initialize the rendering context for the volume object.
2.  If `ordered` is `TRUE`, iterate through the `styles` list from index 0 to $N-1$, applying each style strictly in sequence [S1, unofficial].
3.  If `ordered` is `FALSE`, the implementation may apply the styles in any order or in parallel, provided the underlying implementation supports it [S1, unofficial].
4.  Composite the visual results of the applied styles into a single image [S1, S2, unofficial].

**Pseudocode:**
The following pseudocode illustrates the logic for handling the `ordered` flag and style application. Note that the actual pixel-level compositing of individual style effects (e.g., how `ShadedVolumeStyle` color blends with `OpacityMapVolumeStyle` color) is implementation-specific and not defined by the node specification itself [S1, S2, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_composed_volume_style(styles, ordered):
    """
    Applies a list of volume rendering styles.
    
    styles: list of X3DVolumeRenderStyleNode objects.
    ordered: boolean. If True, styles are applied sequentially.
             If False, order is flexible or parallel.
    """
    if not styles:
        return None

    if ordered:
        # Apply styles strictly in declared order
        for i in range(len(styles)):
            style = styles[i]
            style.apply()
    else:
        # Implementation may apply in any order or in parallel
        # This block represents the flexibility allowed by the spec
        # Actual parallel execution depends on the renderer backend
        for style in styles:
            style.apply()
```

**Parameters:**
*   `ordered`: `FALSE` (default). Allows flexible ordering. `TRUE` enforces strict sequential application [S1, unofficial].
*   `enabled`: `TRUE` (default). Controls visibility of the style [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Order Dependency:** If `ordered` is `FALSE`, the visual result may vary depending on the renderer's internal optimization or parallelization strategy [S1, unofficial].
*   **Empty Styles:** If the `styles` list is empty, no specific style is applied, potentially resulting in default rendering behavior [S1, unofficial].

## Subtopics

### ShadedVolumeStyle
Provides lighting calculations for the volume. Often used in conjunction with ComposedVolumeStyle to add illumination to the final composite [S2, unofficial].

### OpacityMapVolumeStyle
Applies transfer functions to map scalar values to color and opacity. Frequently combined with lighting styles to control visibility [S2, unofficial].

### EdgeEnhancementVolumeStyle
Adds edge enhancement effects based on gradient thresholds. Can be composited with other styles to highlight boundaries [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of styles $N$ and the complexity of each style. If `ordered` is `TRUE`, complexity is $O(N \times T_{style})$. If `FALSE`, it may be $O(T_{style})$ if parallelized, or $O(N \times T_{style})$ if sequential but unordered [S1, unofficial].
*   **Memory Complexity:** $O(N)$ to store references to the style nodes [S1, unofficial].
*   **Strengths:** Simplifies scene graph construction by allowing multiple visual effects in a single node. Supports flexible ordering for performance optimization [S1, S2, unofficial].
*   **Limitations:** The exact blending behavior of multiple styles is not strictly defined by the node specification, relying on implementation details [S1, S2, unofficial].
*   **When to use:** When multiple visual effects (e.g., lighting, edges, opacity) are needed for a single volume object.
*   **When not to use:** When strict, deterministic ordering of style application is required and the renderer does not support `ordered=True` efficiently, or when fine-grained control over pixel-level blending is needed beyond what the style nodes provide.

## Background
ComposedVolumeStyle (also known as CompositeVolumeStyle) is part of the X3D standard specification for volume rendering [S1, unofficial]. It was developed as part of the "Development of an Amendment to X3D to Create a Standard Specification of Medical Image Volume Rendering, Segmentation, and Registration" project [S1, unofficial]. The node is implemented in the instantreality engine, where it is used to combine styles like `ShadedVolumeStyle` and `OpacityMapVolumeStyle` [S2, unofficial].

## Sources
[S1] DTIC ADA469447: Development of an Amendment to X3D to Create a Standard Specification of Medical Image Volume Rendering, Segmentation, and Registration, https://archive.org/details/DTIC_ADA469447, fetched 2026-10-26 (unofficial)
[S2] instantreality 1.0 - tutorial - Volume Rendering, https://doc.instantreality.org/tutorial/volume-rendering/, fetched 2026-10-28 (unofficial)
