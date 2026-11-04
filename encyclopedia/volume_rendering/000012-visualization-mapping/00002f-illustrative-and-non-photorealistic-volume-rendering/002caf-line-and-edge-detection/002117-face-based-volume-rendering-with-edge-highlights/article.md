# Face-based volume rendering with edge highlights

Also called: Volume Color and Highlight Edges Rendering

## Overview
This method renders volumetric data by coloring scalar cells based on their average value while simultaneously detecting and highlighting topological edges (faces of tetrahedra or quads) to provide structural clarity. It distinguishes itself from [Line Illustration Detection](772-line-illustration-detection.md) by operating on face-based primitives rather than extracted skeletal lines, and from [Outline and Contour Generation](4137-outline-and-contour-generation.md) by integrating edge highlights directly into the volume fragment shader rather than post-processing silhouettes. The technique uses a Per-Pixel Linked List (PPLL) structure to manage fragment depth sorting, allowing for correct alpha blending of semi-transparent volume cells and sharp edge highlights in a single pass.

## How it works

### Inputs and Outputs
**Inputs:**
*   **Geometry:** A set of faces (quads or triangles) representing the volume mesh. For each fragment, the shader receives:
    *   `vFragmentPositionWorld`: World-space position of the fragment.
    *   `vVertexPositions[4]`: World-space positions of the four vertices of the face.
    *   `vVertexAttributes[4]`: Scalar values at each vertex.
    *   `vLineAttributes[4]`: Importance/weight flags for each edge (1.0 if important, 0.0 otherwise).
    *   `vEdgeLodValues[4]`: Level-of-Detail (LOD) values for each edge.
*   **Textures:**
    *   `uCellScalarTex`: A 2D texture where each texel stores the scalar value for a cell/face.
*   **Uniforms:**
    *   `uLineWidth`: Base width of the edge highlight.
    *   `uMaxLodValue`: Maximum LOD scale factor.
    *   `uSelectedLodValueFocus`, `uSelectedLodValueContext`: Thresholds for LOD selection in focus and context regions.
    *   `uImportantLineBoostFactor`: Threshold to distinguish important lines.
    *   `uSphereCenter`, `uSphereRadius`: Defines the "focus" region (sphere) for depth cueing.
    *   `uCameraPosition`, `uLookingDirection`: Camera parameters.
    *   `uScalarMin`, `uScalarMax`: Scalar range for normalization.
    *   `uEdgeColor`, `uBackgroundColor`, `uForegroundColor`: Color palette.
    *   `uViewportSize`: Viewport dimensions in pixels.
    *   `uFovY`: Vertical field of view.

**Outputs:**
*   **Blended Color:** The final RGBA color for the fragment.
*   **Depth:** The depth value for the fragment, used for PPLL insertion.

### Data Structures
The method relies on a **Per-Pixel Linked List (PPLL)** structure implemented via GPU textures to handle depth sorting of overlapping fragments (volume cells and edge highlights).

1.  **Start Offset Buffer (`startOffsetImage`):**
    *   Format: `R32UI` (Unsigned 32-bit Integer, Red channel only).
    *   Dimensions: $W \times H$ (Viewport width and height).
    *   Layout: Each pixel stores the index of the head of the linked list for that screen coordinate.
    *   Initialization: All values set to `0xFFFFFFFF` (-1), acting as the end-of-list sentinel.

2.  **Fragment Buffer (`fragmentBufferImage`):**
    *   Format: `RGBA32UI` (Unsigned 32-bit Integer, 4 channels).
    *   Dimensions: $W \times H$.
    *   Layout: Each pixel represents a node in the linked list.
        *   Channel R: Color (or part of it, depending on implementation details not fully specified, but typically used for color/depth packing in PPLL). *Note: The source code initializes this to zero. The exact packing of color/depth/next-pointer into RGBA32UI is implied by the PPLL gather step but not explicitly detailed in the provided shader snippets. We assume standard PPLL packing: e.g., R=next_index, G=depth, B/R=color components, or similar.*
        *   *Correction based on standard PPLL:* Usually, one buffer stores the list head, and another stores the nodes. The node structure typically contains: `next_index`, `depth`, `color`. The provided code initializes `fragmentBufferImage` to zeros. The `gatherFragmentCustomDepth` function (not fully shown) likely reads this buffer to resolve the final color.

### Numbered Steps

1.  **Calculate Edge Width and Radius:**
    *   Compute normalized distance to focus point: $d_{norm} = \text{distanceToFocusPointNormalized}$.
    *   Calculate effective line width: $eWidthPrime = uLineWidth \times (-d_{norm} \times 0.3 + 1.0)$.
    *   Calculate line radius: $lineRadius = eWidthPrime / 2.0$.

2.  **Compute Volume Cell Color:**
    *   Calculate cell scalar value: $cellScalar = 0.25 \times \sum_{i=0}^{3} vVertexAttributes[i]$.
    *   Normalize scalar: $t = \frac{cellScalar - uScalarMin}{\max(uScalarMax - uScalarMin, 10^{-6})}$.
    *   Get base color: $baseColor = colormap(t)$.
    *   Set volume color: $volumeColor = vec4(baseColor, 1.0)$.
    *   Apply context factor: $volumeColor.a \times= contextFactor$.

3.  **Initialize LOD Constants:**
    *   Define epsilon: $LOD\_EPSILON = 0.001$.
    *   Calculate discrete LOD thresholds:
        *   $discreteSelectedLodValueFocus = \max(uSelectedLodValueFocus \times uMaxLodValue, LOD\_EPSILON)$.
        *   $discreteSelectedLodValueContext = \max(uSelectedLodValueContext \times uMaxLodValue, LOD\_EPSILON)$.

4.  **Initialize Distance Tracking:**
    *   Set $minDistance = 10^9$, $minDistanceIndex = 0$, $minDistanceAll = 10^9$, $minDistanceIndexAll = 0$, $isLineNear = false$.

5.  **Iterate Over Edges (i = 0 to 3):**
    *   Calculate distance to line segment: $currentDistance = getDistanceToLineSegment(vFragmentPositionWorld, vVertexPositions[i], vVertexPositions[(i+1)\%4])$.
    *   Calculate LOD level for focus:
        *   $isImportant = (vLineAttributes[i] < 1.0 - uImportantLineBoostFactor) ? 0.0 : 1.0$.
        *   $lodLevelFocus = \max(isImportant \times uMaxLodValue, discreteSelectedLodValueFocus)$.
    *   Calculate LOD line value: $lodLineValue = vEdgeLodValues[i]$.
    *   Calculate discrete LOD value: $discreteLodValue = lodLineValue \times uMaxLodValue$.
    *   Calculate LOD opacity factor:
        *   $condContext = (discreteLodValue \le discreteSelectedLodValueContext + LOD\_EPSILON) ? 1.0 : 0.0$.
        *   $condFocus = (discreteLodValue \le discreteSelectedLodValueFocus + LOD\_EPSILON) ? 1.0 : 0.0$.
        *   $lodLevelOpacityFactor = mix(condContext, condFocus, focusFactor)$.
    *   Determine if line should be drawn: $drawLine = (lodLevelOpacityFactor > 0.01)$.
    *   Update minimum distances:
        *   If $currentDistance < minDistance$ and $drawLine$:
            *   $minDistance = currentDistance$.
            *   $minDistanceIndex = i$.
            *   $isLineNear = true$.
        *   If $currentDistance < minDistanceAll$:
            *   $minDistanceAll = currentDistance$.
            *   $minDistanceIndexAll = i$.

6.  **Calculate Edge Highlight Color (if near line):**
    *   If $isLineNear$ and $minDistance \le lineRadius$:
        *   Calculate depth cue factor: $depthCueFactor = \min(contextFactor, focusFactor)$.
        *   Retrieve LOD value for the closest line: $lodLineValue = vEdgeLodValues[minDistanceIndex]$.
        *   Calculate LOD opacity factor for the line:
            *   $condContext = (lodLineValue \le uSelectedLodValueContext + LOD\_EPSILON) ? 1.0 : 0.0$.
            *   $condFocus = (lodLineValue \le uSelectedLodValueFocus + LOD\_EPSILON) ? 1.0 : 0.0$.
            *   $lodLevelOpacityFactor = mix(condContext, condFocus, focusFactor)$.
        *   Calculate base line color:
            *   $lineBaseColor = uEdgeColor$.
            *   $lineBaseColorFocus = mix(lineBaseColor, uBackgroundColor, 0.3)$.
            *   $lineBaseColorContext = mix(baseColor, uForegroundColor, 0.3)$.
            *   $lineBaseColor = mix(lineBaseColorContext, lineBaseColorFocus, focusFactor)$.
        *   Calculate line coordinates: $lineCoordinates = \max(minDistance / lineRadius, 0.0)$.
        *   Calculate anti-aliasing factor: $EPSILON = clamp(getAntialiasingFactor(fragmentDistance / lineRadius), 0.0, 0.49)$.
        *   Calculate coverage: $coverage = 1.0 - smoothstep(1.0 - 2.0 \times EPSILON, 1.0, lineCoordinates)$.
        *   Set line color: $lineColor = vec4(lineBaseColor, coverage \times lodLevelOpacityFactor)$.

7.  **Alpha Blend:**
    *   Perform back-to-front alpha blend:
        *   $blendedColor.a = lineColor.a + volumeColor.a \times (1.0 - lineColor.a)$.
        *   $blendedColor.rgb = lineColor.rgb \times lineColor.a + volumeColor.rgb \times volumeColor.a \times (1.0 - lineColor.a)$.
    *   Premultiply if alpha is significant: If $blendedColor.a > 10^{-4}$, $blendedColor.rgb /= blendedColor.a$.

8.  **Store Fragment:**
    *   Call $gatherFragmentCustomDepth(blendedColor, fragmentDistance)$. This function inserts the fragment into the PPLL structure for later resolution.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def face_based_volume_rendering(vFragmentPositionWorld, vVertexPositions, vVertexAttributes, vLineAttributes, vEdgeLodValues, uLineWidth, uMaxLodValue, uSelectedLodValueFocus, uSelectedLodValueContext, uImportantLineBoostFactor, uSphereCenter, uSphereRadius, uCameraPosition, uLookingDirection, uScalarMin, uScalarMax, uEdgeColor, uBackgroundColor, uForegroundColor, uViewportSize, uFovY, contextFactor, focusFactor):
    # Step 1: Edge Width and Radius
    distanceToFocusPointNormalized = calculate_distance_to_focus(vFragmentPositionWorld, uSphereCenter, uSphereRadius)
    eWidthPrime = uLineWidth * (-distanceToFocusPointNormalized * 0.3 + 1.0)
    lineRadius = eWidthPrime / 2.0

    # Step 2: Volume Cell Color
    cellScalar = 0.25 * sum(vVertexAttributes)
    t = (cellScalar - uScalarMin) / max(uScalarMax - uScalarMin, 1e-6)
    baseColor = colormap(clamp(t, 0.0, 1.0))
    volumeColor = vec4(baseColor, 1.0)
    volumeColor.a *= contextFactor

    # Step 3: LOD Constants
    LOD_EPSILON = 0.001
    discreteSelectedLodValueFocus = max(uSelectedLodValueFocus * uMaxLodValue, LOD_EPSILON)
    discreteSelectedLodValueContext = max(uSelectedLodValueContext * uMaxLodValue, LOD_EPSILON)

    # Step 4: Initialize Distance Tracking
    minDistance = 1e9
    minDistanceIndex = 0
    minDistanceAll = 1e9
    minDistanceIndexAll = 0
    isLineNear = False

    # Step 5: Iterate Over Edges
    for i in range(4):
        currentDistance = get_distance_to_line_segment(vFragmentPositionWorld, vVertexPositions[i], vVertexPositions[(i + 1) % 4])
        
        isImportant = 1.0 if vLineAttributes[i] >= (1.0 - uImportantLineBoostFactor) else 0.0
        lodLevelFocus = max(isImportant * uMaxLodValue, discreteSelectedLodValueFocus)
        
        lodLineValue = vEdgeLodValues[i]
        discreteLodValue = lodLineValue * uMaxLodValue
        
        condContext = 1.0 if discreteLodValue <= (discreteSelectedLodValueContext + LOD_EPSILON) else 0.0
        condFocus = 1.0 if discreteLodValue <= (discreteSelectedLodValueFocus + LOD_EPSILON) else 0.0
        lodLevelOpacityFactor = mix(condContext, condFocus, focusFactor)
        
        drawLine = lodLevelOpacityFactor > 0.01
        
        if currentDistance < minDistance and drawLine:
            minDistance = currentDistance
            minDistanceIndex = i
            isLineNear = True
            
        if currentDistance < minDistanceAll:
            minDistanceAll = currentDistance
            minDistanceIndexAll = i

    # Step 6: Edge Highlight Color
    lineColor = vec4(0.0, 0.0, 0.0, 0.0)
    if isLineNear and minDistance <= lineRadius:
        depthCueFactor = min(contextFactor, focusFactor)
        
        lodLineValue = vEdgeLodValues[minDistanceIndex]
        condContext = 1.0 if lodLineValue <= (uSelectedLodValueContext + LOD_EPSILON) else 0.0
        condFocus = 1.0 if lodLineValue <= (uSelectedLodValueFocus + LOD_EPSILON) else 0.0
        lodLevelOpacityFactor = mix(condContext, condFocus, focusFactor)
        
        lineBaseColor = uEdgeColor
        lineBaseColorFocus = mix(lineBaseColor, uBackgroundColor, 0.3)
        lineBaseColorContext = mix(baseColor, uForegroundColor, 0.3)
        lineBaseColor = mix(lineBaseColorContext, lineBaseColorFocus, focusFactor)
        
        lineCoordinates = max(minDistance / lineRadius, 0.0)
        EPSILON = clamp(get_antialiasing_factor(fragmentDistance / lineRadius, uViewportSize, uFovY), 0.0, 0.49)
        coverage = 1.0 - smoothstep(1.0 - 2.0 * EPSILON, 1.0, lineCoordinates)
        
        lineColor = vec4(lineBaseColor, coverage * lodLevelOpacityFactor)

    # Step 7: Alpha Blend
    blendedColor = vec4(0.0, 0.0, 0.0, 0.0)
    blendedColor.a = lineColor.a + volumeColor.a * (1.0 - lineColor.a)
    blendedColor.rgb = lineColor.rgb * lineColor.a + volumeColor.rgb * volumeColor.a * (1.0 - lineColor.a)
    
    if blendedColor.a > 1e-4:
        blendedColor.rgb /= blendedColor.a

    # Step 8: Store Fragment
    gather_fragment_custom_depth(blendedColor, fragmentDistance)
    
    return blendedColor

def get_antialiasing_factor(distance, viewportSize, fovY):
    return distance / viewportSize.y * fovY

def mix(a, b, t):
    return a + t * (b - a)

def clamp(x, min_val, max_val):
    return max(min_val, min(x, max_val))

def smoothstep(edge0, edge1, x):
    t = clamp((x - edge0) / (edge1 - edge0), 0.0, 1.0)
    return t * t * (3 - 2 * t)

def vec4(r, g, b, a):
    return {'r': r, 'g': g, 'b': b, 'a': a}

def sum(lst):
    return sum(lst)

def max(a, b):
    return a if a > b else b

def min(a, b):
    return a if a < b else b
```

### Parameters with Typical Values
*   `uLineWidth`: Not given in the sources.
*   `uMaxLodValue`: Not given in the sources.
*   `uSelectedLodValueFocus`, `uSelectedLodValueContext`: Not given in the sources.
*   `uImportantLineBoostFactor`: Not given in the sources.
*   `LOD_EPSILON`: 0.001.
*   `uSphereRadius`: Not given in the sources.

### Edge Cases and Pitfalls
*   **Precision:** The use of `1e-6` in denominator normalization prevents division by zero if `uScalarMax == uScalarMin`.
*   **Boundaries:** The `clamp` function is used for `lineCoordinates` and `EPSILON` to ensure valid ranges for `smoothstep` and anti-aliasing calculations.
*   **Degenerate Input:** If `uScalarMax - uScalarMin` is zero, the scalar normalization uses `1e-6` as the denominator. If `lineRadius` is zero, division by zero may occur in `lineCoordinates` calculation; however, `uLineWidth` and focus factor should prevent this in practice.

## Subtopics

### Line Illustration Detection
Extracts skeletal lines from volume data for stylized rendering. Choose when a more abstract, line-based representation is desired. [Link](772-line-illustration-detection.md)

### Outline and Contour Generation
Generates silhouettes and contours in post-processing. Choose when edge detection needs to be decoupled from volume shading or when using screen-space techniques. [Link](4137-outline-and-contour-generation.md)

### LOG-based Silhouette Detection
Uses Laplacian of Gaussian for edge detection. Choose when robustness to noise is required. [Link](6322-log-based-silhouette-detection.md)

## Operating characteristics

*   **Time Complexity:** $O(N)$ where $N$ is the number of faces/fragments. The PPLL insertion and gathering are also linear in the number of fragments per pixel, but worst-case can be $O(K)$ where $K$ is the number of overlapping fragments.
*   **Memory Complexity:** $O(W \times H)$ for the PPLL buffers (`startOffsetImage` and `fragmentBufferImage`), where $W$ and $H$ are viewport dimensions.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales with viewport resolution due to PPLL buffer size.
*   **Strengths:** Integrates edge highlights directly with volume shading, providing clear structural information. Uses PPLL for correct depth sorting without requiring complex sorting algorithms.
*   **Limitations:** Requires PPLL buffer management. Performance may degrade with high fragment density per pixel.
*   **When to Use:** When structural clarity is needed in volume rendering, and edge highlights are desired.
*   **When Not to Use:** When performance is critical and fragment density is very high, or when a purely photorealistic rendering is desired without stylized edges.

## Background

This method is derived from the "ClearView" rendering technique, specifically the "HexMeshUnified" fragment shader implementation. It is part of a broader effort to provide illustrative volume rendering with edge highlights. The technique is implemented in the "geresim-spike" project, which is an unofficial implementation.

## Sources

[S1] Amory0709/geresim-spike, https://github.com/Amory0709/geresim-spike, fetched 2026-10-27 (unofficial)
