# Face-based volume rendering with edge highlights

Also called: Volume Color and Highlight Edges Rendering

## Overview
This method renders volumetric data by treating individual cells (faces) as the primary sampling units, applying a scalar-based colormap for the volume body while simultaneously detecting and highlighting cell edges. It distinguishes itself from sibling methods like [Line Illustration Detection](772-line-illustration-detection.md) and [Outline and Contour Generation](4137-outline-and-contour-generation.md) by computing edge highlights directly within the face fragment shader based on distance to cell vertices and edges, rather than relying on post-processing contours or separate line-detection passes. It solves the problem of visualizing internal structures and boundaries in 3D volumes by providing explicit edge cues that enhance depth perception and structural clarity without obscuring the underlying scalar data.

## How it works
The method operates in a fragment shader stage where each fragment corresponds to a projected cell face. It computes a base volume color and then determines if the fragment is near any of the four edges of the current cell. If so, it blends an edge highlight color over the volume color, modulated by Level of Detail (LOD) and focus factors.

### Inputs and Outputs
**Inputs:**
- `vFragmentPositionWorld`: World-space position of the fragment.
- `vVertexPositions[4]`: World-space positions of the four vertices of the current cell face.
- `vVertexAttributes[4]`: Scalar values at the four vertices.
- `vLineAttributes[4]`: Flags indicating if an edge is "important" (1.0) or not (0.0).
- `vEdgeLodValues[4]`: Pre-computed LOD values for each edge.
- `uCellScalarTex`: Texture containing scalar values for cells (1 texel per face).
- `uLineWidth`: Base width of the edge highlight.
- `uMaxLodValue`: Maximum LOD value scale.
- `uSelectedLodValueFocus`, `uSelectedLodValueContext`: Thresholds for focus and context LOD.
- `uImportantLineBoostFactor`: Threshold to distinguish important lines.
- `uSphereCenter`, `uSphereRadius`: Defines the focus region (sphere).
- `uCameraPosition`, `uLookingDirection`: Camera parameters.
- `uEdgeColor`, `uBackgroundColor`, `uForegroundColor`: Color uniforms.
- `uScalarMin`, `uScalarMax`: Scalar range for normalization.
- `uFovY`, `uViewportSize`: Viewport parameters for anti-aliasing.

**Outputs:**
- `blendedColor`: The final RGBA color of the fragment.
- Depth information (via `gatherFragmentCustomDepth`).

### Data Structures
The implementation relies on a Parallel Prefix Linked List (PPLL) structure for fragment management, though the core rendering logic described here is per-fragment.
- **Start Offset Buffer**: `image2D` of type `R32UI`, size $W \times H$ (viewport). Initialized to `0xFFFFFFFF` (-1). Stores the head of the linked list for each pixel.
- **Fragment Buffer**: `image2D` of type `RGBA32UI`, size $W \times H$. Each pixel stores a fragment node: color (RGBA) and next pointer. Initialized to zero.

### Numbered Steps and Formulas

1.  **Calculate Line Radius**:
    Compute the effective line width based on distance to the focus point.
    $$ eWidthPrime = uLineWidth \times (-distanceToFocusPointNormalized \times 0.3 + 1.0) $$
    $$ lineRadius = eWidthPrime / 2.0 $$
    *Note: `distanceToFocusPointNormalized` is the normalized distance from the fragment to `uSphereCenter`.*

2.  **Compute Base Volume Color**:
    Sample the scalar value for the cell.
    $$ cellScalar = 0.25 \times \sum_{i=0}^{3} vVertexAttributes[i] $$
    Normalize the scalar:
    $$ t = \frac{cellScalar - uScalarMin}{\max(uScalarMax - uScalarMin, 10^{-6})} $$
    Apply colormap:
    $$ baseColor = colormap(t) $$
    Set initial volume color:
    $$ volumeColor = vec4(baseColor, 1.0) $$
    Modulate alpha by context factor (derived from distance to focus sphere):
    $$ volumeColor.a \times= contextFactor $$

3.  **Initialize LOD Constants**:
    $$ LOD\_EPSILON = 0.001 $$
    $$ discreteSelectedLodValueFocus = \max(uSelectedLodValueFocus \times uMaxLodValue, LOD\_EPSILON) $$
    $$ discreteSelectedLodValueContext = \max(uSelectedLodValueContext \times uMaxLodValue, LOD\_EPSILON) $$

4.  **Initialize Distance Tracking**:
    $$ minDistance = 10^9, minDistanceIndex = 0 $$
    $$ minDistanceAll = 10^9, minDistanceIndexAll = 0 $$
    $$ isLineNear = false $$

5.  **Iterate Over Cell Edges**:
    For each edge $i$ from 0 to 3:
    a. Calculate distance from fragment to the line segment defined by vertices $i$ and $(i+1)\%4$:
    $$ currentDistance = getDistanceToLineSegment(vFragmentPositionWorld, vVertexPositions[i], vVertexPositions[(i + 1) \% 4]) $$
    
    b. Determine LOD level for focus:
    $$ lodLevelFocus = \max\left( \begin{cases} 0.0 & \text{if } vLineAttributes[i] < 1.0 - uImportantLineBoostFactor \\ 1.0 & \text{otherwise} \end{cases} \times uMaxLodValue, discreteSelectedLodValueFocus \right) $$
    
    c. Determine LOD level for the line:
    $$ lodLineValue = vEdgeLodValues[i] $$
    $$ discreteLodValue = lodLineValue \times uMaxLodValue $$
    
    d. Calculate opacity factor based on LOD thresholds:
    $$ lodLevelOpacityFactor = mix\left( \begin{cases} 1.0 & \text{if } discreteLodValue \le discreteSelectedLodValueContext + LOD\_EPSILON \\ 0.0 & \text{otherwise} \end{cases}, \begin{cases} 1.0 & \text{if } discreteLodValue \le discreteSelectedLodValueFocus + LOD\_EPSILON \\ 0.0 & \text{otherwise} \end{cases}, focusFactor \right) $$
    
    e. Determine if the line should be drawn:
    $$ drawLine = (lodLevelOpacityFactor > 0.01) $$
    
    f. Update minimum distances:
    If $currentDistance < minDistance$ and $drawLine$:
    $$ minDistance = currentDistance, minDistanceIndex = i, isLineNear = true $$
    If $currentDistance < minDistanceAll$:
    $$ minDistanceAll = currentDistance, minDistanceIndexAll = i $$

6.  **Calculate Edge Highlight**:
    If $isLineNear$ and $minDistance \le lineRadius$:
    a. Calculate depth cue factor:
    $$ depthCueFactor = \min(contextFactor, focusFactor) $$
    
    b. Recalculate LOD opacity for the closest line:
    $$ lodLineValue = vEdgeLodValues[minDistanceIndex] $$
    $$ lodLevelOpacityFactor = mix\left( \begin{cases} 1.0 & \text{if } lodLineValue \le uSelectedLodValueContext + LOD\_EPSILON \\ 0.0 & \text{otherwise} \end{cases}, \begin{cases} 1.0 & \text{if } lodLineValue \le uSelectedLodValueFocus + LOD\_EPSILON \\ 0.0 & \text{otherwise} \end{cases}, focusFactor \right) $$
    
    c. Determine line base color:
    $$ lineBaseColor = uEdgeColor $$
    $$ lineBaseColorFocus = mix(lineBaseColor, uBackgroundColor, 0.3) $$
    $$ lineBaseColorContext = mix(baseColor, uForegroundColor, 0.3) $$
    $$ lineBaseColor = mix(lineBaseColorContext, lineBaseColorFocus, focusFactor) $$
    
    d. Calculate line coordinates (normalized distance):
    $$ lineCoordinates = \max(minDistance / lineRadius, 0.0) $$
    
    e. Calculate anti-aliasing factor and coverage:
    $$ EPSILON = clamp(getAntialiasingFactor(fragmentDistance / lineRadius), 0.0, 0.49) $$
    $$ coverage = 1.0 - smoothstep(1.0 - 2.0 \times EPSILON, 1.0, lineCoordinates) $$
    
    f. Set line color:
    $$ lineColor = vec4(lineBaseColor, coverage \times lodLevelOpacityFactor) $$

7.  **Blend Colors**:
    Perform back-to-front alpha blending of the line color over the volume color:
    $$ blendedColor.a = lineColor.a + volumeColor.a \times (1.0 - lineColor.a) $$
    $$ blendedColor.rgb = lineColor.rgb \times lineColor.a + volumeColor.rgb \times volumeColor.a \times (1.0 - lineColor.a) $$
    If $blendedColor.a > 10^{-4}$, premultiply RGB:
    $$ blendedColor.rgb /= blendedColor.a $$

8.  **Output**:
    Call `gatherFragmentCustomDepth(blendedColor, fragmentDistance)` to store the fragment in the PPLL structure for subsequent sorting and compositing.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_face_fragment(vFragmentPositionWorld, vVertexPositions, vVertexAttributes, 
                         vLineAttributes, vEdgeLodValues, uCellScalarTex, 
                         uLineWidth, uMaxLodValue, uSelectedLodValueFocus, 
                         uSelectedLodValueContext, uImportantLineBoostFactor,
                         uSphereCenter, uSphereRadius, uCameraPosition, 
                         uLookingDirection, uEdgeColor, uBackgroundColor, 
                         uForegroundColor, uScalarMin, uScalarMax, 
                         uFovY, uViewportSize):
    # Step 1: Calculate Line Radius
    # distanceToFocusPointNormalized is assumed to be computed from vFragmentPositionWorld and uSphereCenter
    distanceToFocusPointNormalized = compute_normalized_distance(vFragmentPositionWorld, uSphereCenter, uSphereRadius)
    eWidthPrime = uLineWidth * (-distanceToFocusPointNormalized * 0.3 + 1.0)
    lineRadius = eWidthPrime / 2.0

    # Step 2: Compute Base Volume Color
    cellScalar = 0.25 * sum(vVertexAttributes)
    scalar_range = max(uScalarMax - uScalarMin, 1e-6)
    t = (cellScalar - uScalarMin) / scalar_range
    baseColor = colormap(clamp(t, 0.0, 1.0))
    volumeColor = vec4(baseColor, 1.0)
    # contextFactor is assumed to be computed based on distance to focus sphere
    volumeColor.a *= contextFactor

    # Step 3: Initialize LOD Constants
    LOD_EPSILON = 0.001
    discreteSelectedLodValueFocus = max(uSelectedLodValueFocus * uMaxLodValue, LOD_EPSILON)
    discreteSelectedLodValueContext = max(uSelectedLodValueContext * uMaxLodValue, LOD_EPSILON)

    # Step 4: Initialize Distance Tracking
    minDistance = 1e9
    minDistanceIndex = 0
    minDistanceAll = 1e9
    minDistanceIndexAll = 0
    isLineNear = False

    # Step 5: Iterate Over Cell Edges
    for i in range(4):
        currentDistance = get_distance_to_line_segment(vFragmentPositionWorld, vVertexPositions[i], vVertexPositions[(i + 1) % 4])
        
        # LOD Level Focus
        is_important = vLineAttributes[i] >= 1.0 - uImportantLineBoostFactor
        lod_level_focus_val = 1.0 if is_important else 0.0
        lodLevelFocus = max(lod_level_focus_val * uMaxLodValue, discreteSelectedLodValueFocus)
        
        # LOD Line Value
        lodLineValue = vEdgeLodValues[i]
        discreteLodValue = lodLineValue * uMaxLodValue
        
        # LOD Level Opacity Factor
        context_threshold = discreteSelectedLodValueContext + LOD_EPSILON
        focus_threshold = discreteSelectedLodValueFocus + LOD_EPSILON
        cond_context = 1.0 if discreteLodValue <= context_threshold else 0.0
        cond_focus = 1.0 if discreteLodValue <= focus_threshold else 0.0
        lodLevelOpacityFactor = mix(cond_context, cond_focus, focusFactor)
        
        drawLine = lodLevelOpacityFactor > 0.01
        
        if currentDistance < minDistance and drawLine:
            minDistance = currentDistance
            minDistanceIndex = i
            isLineNear = True
            
        if currentDistance < minDistanceAll:
            minDistanceAll = currentDistance
            minDistanceIndexAll = i

    # Step 6: Calculate Edge Highlight
    if isLineNear and minDistance <= lineRadius:
        depthCueFactor = min(contextFactor, focusFactor)
        
        lodLineValue = vEdgeLodValues[minDistanceIndex]
        context_thresh_val = uSelectedLodValueContext + LOD_EPSILON
        focus_thresh_val = uSelectedLodValueFocus + LOD_EPSILON
        cond_context_val = 1.0 if lodLineValue <= context_thresh_val else 0.0
        cond_focus_val = 1.0 if lodLineValue <= focus_thresh_val else 0.0
        lodLevelOpacityFactor = mix(cond_context_val, cond_focus_val, focusFactor)
        
        lineBaseColor = uEdgeColor
        lineBaseColorFocus = mix(lineBaseColor, uBackgroundColor, 0.3)
        lineBaseColorContext = mix(baseColor, uForegroundColor, 0.3)
        lineBaseColor = mix(lineBaseColorContext, lineBaseColorFocus, focusFactor)
        
        lineCoordinates = max(minDistance / lineRadius, 0.0)
        
        # Anti-aliasing
        aa_factor = get_antialiasing_factor(fragmentDistance / lineRadius, uFovY, uViewportSize)
        EPSILON = clamp(aa_factor, 0.0, 0.49)
        coverage = 1.0 - smoothstep(1.0 - 2.0 * EPSILON, 1.0, lineCoordinates)
        
        lineColor = vec4(lineBaseColor, coverage * lodLevelOpacityFactor)
        
        # Step 7: Blend Colors
        blendedColor_a = lineColor.a + volumeColor.a * (1.0 - lineColor.a)
        blendedColor_rgb = lineColor.rgb * lineColor.a + volumeColor.rgb * volumeColor.a * (1.0 - lineColor.a)
        
        if blendedColor_a > 1e-4:
            blendedColor_rgb /= blendedColor_a
            
        return vec4(blendedColor_rgb, blendedColor_a)
    else:
        return volumeColor

def get_antialiasing_factor(distance, fov_y, viewport_size):
    return distance / viewport_size.y * fov_y

def colormap(t):
    # Simplified Viridis-like colormap
    if t < 0.16:
        return vec3(0.267, 0.004, 0.329)
    elif t < 0.33:
        return mix(vec3(0.267, 0.004, 0.329), vec3(0.282, 0.141, 0.459), (t - 0.16) / 0.17)
    elif t < 0.50:
        return mix(vec3(0.282, 0.141, 0.459), vec3(0.254, 0.265, 0.530), (t - 0.33) / 0.17)
    elif t < 0.66:
        return mix(vec3(0.254, 0.265, 0.530), vec3(0.190, 0.486, 0.520), (t - 0.50) / 0.16)
    elif t < 0.83:
        return mix(vec3(0.190, 0.486, 0.520), vec3(0.460, 0.660, 0.390), (t - 0.66) / 0.17)
    else:
        return mix(vec3(0.460, 0.660, 0.390), vec3(0.993, 0.906, 0.144), (t - 0.83) / 0.17)
```

### Edge Cases and Pitfalls
- **Precision**: The use of `1e-6` in scalar normalization prevents division by zero if `uScalarMax == uScalarMin`.
- **Boundaries**: `lineCoordinates` is clamped to `>= 0.0` to handle fragments inside the line radius.
- **Degenerate Input**: If `uLineWidth` is zero, `lineRadius` becomes zero, potentially causing division by zero in `lineCoordinates`. The sources do not specify a guard for this, but `getDistanceToLineSegment` should handle coincident points.
- **LOD Thresholds**: The `LOD_EPSILON` is used to avoid floating-point comparison issues when checking LOD values against thresholds.

## Operating characteristics
- **Time Complexity**: $O(1)$ per fragment, as it iterates over a fixed number of edges (4) per cell face.
- **Memory Complexity**: $O(W \times H)$ for the PPLL buffers (Start Offset and Fragment Buffer).
- **Performance**: The method is efficient as it avoids complex ray-marching or post-processing passes for edge detection. Performance depends on the cost of `getDistanceToLineSegment` and the number of fragments processed.
- **Strengths**: Provides clear edge highlights that enhance structural understanding. Integrates seamlessly with existing volume rendering pipelines. LOD support allows for performance scaling.
- **Limitations**: Edge highlights are only visible if the fragment is close to a cell edge. May not capture all internal structures if the cell size is large relative to the feature size.
- **When to use**: When structural clarity is needed in volumetric data, especially for scientific visualization.
- **When not to use**: When photorealism is the primary goal and edge highlights are considered distracting.

## Background
This method is derived from the "ClearView" rendering technique, specifically the "HexMeshUnified" shader implementation found in the HexVolumeRenderer project. It is part of the broader family of illustrative and non-photorealistic volume rendering techniques. The PPLL structure for fragment management is a common optimization in GPU-based volume rendering to handle depth sorting without CPU intervention.

## Sources
[S1] Amory0709/geresim-spike, https://github.com/Amory0709/geresim-spike, fetched 2026-10-27 (unofficial)
