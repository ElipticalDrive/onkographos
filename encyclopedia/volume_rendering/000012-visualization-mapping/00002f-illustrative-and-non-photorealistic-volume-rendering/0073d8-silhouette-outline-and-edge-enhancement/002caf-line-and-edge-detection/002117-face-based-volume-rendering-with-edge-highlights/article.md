# Face-based volume rendering with edge highlights

Also called: Volume Color and Highlight Edges Rendering

## Overview
Face-based volume rendering with edge highlights renders volumetric data by treating individual cells (faces) as the primary sampling unit, applying a scalar-based colormap for the body, and superimposing anti-aliased edge highlights to enhance structural clarity [S1, unofficial]. It differs from silhouette-based methods by detecting edges locally within each cell based on vertex attribute discontinuities or importance factors, rather than relying on global view-dependent depth discontinuities [S1, unofficial].

## How it works
The method operates in a fragment shader context where each fragment corresponds to a specific location within a volumetric cell (typically a quad or tetrahedron face) [S1, unofficial]. The algorithm blends the volume color with edge highlights using a back-to-front alpha compositing model [S1, unofficial].

### Inputs and Outputs
**Inputs:**
*   `vFragmentPositionWorld`: World-space position of the current fragment [S1, unofficial].
*   `vVertexPositions[4]`: World-space positions of the four vertices of the current cell [S1, unofficial].
*   `vVertexAttributes[4]`: Scalar values at the four vertices (e.g., density, temperature) [S1, unofficial].
*   `vLineAttributes[4]`: Float array used to determine edge importance; an edge is considered important if `vLineAttributes[i] >= 1.0 - uImportantLineBoostFactor` [S1, unofficial].
*   `vEdgeLodValues[4]`: Pre-computed Level of Detail (LOD) values for the four edges [S1, unofficial].
*   `uCellScalarTex`: Texture containing scalar values for the cell (1 texel per face) [S1, unofficial].
*   Uniforms: Camera parameters (`uCameraPosition`, `uLookingDirection`), focus context (`uSphereCenter`, `uSphereRadius`), rendering parameters (`uLineWidth`, `uMaxLodValue`, `uSelectedLodValueFocus`, `uSelectedLodValueContext`, `uImportantLineBoostFactor`), colors (`uBackgroundColor`, `uForegroundColor`, `uEdgeColor`), and scalar range (`uScalarMin`, `uScalarMax`) [S1, unofficial].
*   `contextFactor`, `focusFactor`, `fragmentDistance`: Depth cue and focus modulation factors [S1, unofficial].

**Outputs:**
*   Blended fragment color and alpha [S1, unofficial].
*   Custom depth value for subsequent processing via `gatherFragmentCustomDepth` [S1, unofficial].

### Data Structures
The implementation relies on GPU textures for buffer management, specifically for the Parallel Prefix List List (PPLL) structure used to manage fragments, though the core edge highlight logic is computed per-fragment [S1, unofficial].
*   **Start Offset Buffer:** `R32UI` texture of size $W \times H$ (viewport dimensions). Initialized to `0xFFFFFFFF` (-1 as unsigned int) to serve as an end-of-list sentinel [S1, unofficial].
*   **Fragment Buffer:** `RGBA32UI` texture of size $W \times H$. Each pixel stores a fragment node containing color, depth, and a `next` pointer (initialized to 0) [S1, unofficial].
*   **Cell Scalar Texture:** `RED` `FLOAT` texture. One texel per face, used to retrieve the cell's scalar value [S1, unofficial].

### Algorithm Steps
1.  **Calculate Line Geometry:**
    *   Compute the effective line width: $eWidthPrime = uLineWidth \times (-distanceToFocusPointNormalized \times 0.3 + 1.0)$ [S1, unofficial].
    *   Compute the line radius: $lineRadius = eWidthPrime / 2.0$ [S1, unofficial].
    *   *Note:* `distanceToFocusPointNormalized` is the normalized distance from the fragment to the focus sphere center [G].

2.  **Compute Volume Color:**
    *   Calculate the cell scalar average: $cellScalar = 0.25 \times \sum_{i=0}^{3} vVertexAttributes[i]$ [S1, unofficial].
    *   Normalize the scalar: $t = \frac{cellScalar - uScalarMin}{\max(uScalarMax - uScalarMin, 10^{-6})}$ [S1, unofficial].
    *   Apply colormap: $baseColor = colormap(t)$ [S1, unofficial].
    *   Set volume color: $volumeColor = vec4(baseColor, 1.0)$ [S1, unofficial].
    *   Modulate alpha by context factor: $volumeColor.a \times= contextFactor$ [S1, unofficial].

3.  **Initialize LOD and Distance Tracking:**
    *   Set $LOD\_EPSILON = 0.001$ [S1, unofficial].
    *   Compute discrete LOD thresholds:
        *   $discreteSelectedLodValueFocus = \max(uSelectedLodValueFocus \times uMaxLodValue, LOD\_EPSILON)$ [S1, unofficial].
        *   $discreteSelectedLodValueContext = \max(uSelectedLodValueContext \times uMaxLodValue, LOD\_EPSILON)$ [S1, unofficial].
    *   Initialize variables: $minDistance = 10^9$, $minDistanceIndex = 0$, $minDistanceAll = 10^9$, $minDistanceIndexAll = 0$, $isLineNear = false$ [S1, unofficial].

4.  **Iterate Over Cell Edges:**
    *   For each edge $i$ from 0 to 3:
        *   Calculate distance from fragment to the line segment defined by vertices $i$ and $(i+1)\%4$: $currentDistance = getDistanceToLineSegment(vFragmentPositionWorld, vVertexPositions[i], vVertexPositions[(i+1)\%4])$ [S1, unofficial].
        *   Determine if the edge is "important" based on `vLineAttributes[i]` and `uImportantLineBoostFactor`. Calculate focus LOD level:
            $$lodLevelFocus = \max\left(\left(vLineAttributes[i] < 1.0 - uImportantLineBoostFactor ? 0.0 : 1.0\right) \times uMaxLodValue, discreteSelectedLodValueFocus\right)$$
            [S1, unofficial].
        *   Calculate the edge's LOD value: $lodLineValue = vEdgeLodValues[i] \times uMaxLodValue$ [S1, unofficial].
        *   Determine opacity factor based on LOD thresholds and `focusFactor`:
            $$lodLevelOpacityFactor = mix\left(\begin{aligned} &lodLineValue \le discreteSelectedLodValueContext + LOD\_EPSILON ? 1.0 : 0.0, \\ &lodLineValue \le discreteSelectedLodValueFocus + LOD\_EPSILON ? 1.0 : 0.0, \\ &focusFactor \end{aligned}\right)$$
            [S1, unofficial].
        *   Determine if the line should be drawn: $drawLine = (lodLevelOpacityFactor > 0.01)$ [S1, unofficial].
        *   Update minimum distances:
            *   If $currentDistance < minDistance$ and $drawLine$: $minDistance = currentDistance$, $minDistanceIndex = i$, $isLineNear = true$ [S1, unofficial].
            *   If $currentDistance < minDistanceAll$: $minDistanceAll = currentDistance$, $minDistanceIndexAll = i$ [S1, unofficial].

5.  **Compute Edge Highlight Color:**
    *   If $isLineNear$ and $minDistance \le lineRadius$:
        *   Calculate depth cue factor: $depthCueFactor = \min(contextFactor, focusFactor)$ [S1, unofficial].
        *   Retrieve LOD value for the nearest edge: $lodLineValue = vEdgeLodValues[minDistanceIndex]$ [S1, unofficial].
        *   Recalculate opacity factor using original LOD values:
            $$lodLevelOpacityFactor = mix\left(\begin{aligned} &lodLineValue \le uSelectedLodValueContext + LOD\_EPSILON ? 1.0 : 0.0, \\ &lodLineValue \le uSelectedLodValueFocus + LOD\_EPSILON ? 1.0 : 0.0, \\ &focusFactor \end{aligned}\right)$$
            [S1, unofficial].
        *   Determine base line color:
            *   $lineBaseColor = uEdgeColor$ [S1, unofficial].
            *   $lineBaseColorFocus = mix(lineBaseColor, uBackgroundColor, 0.3)$ [S1, unofficial].
            *   $lineBaseColorContext = mix(baseColor, uForegroundColor, 0.3)$ [S1, unofficial].
            *   $lineBaseColor = mix(lineBaseColorContext, lineBaseColorFocus, focusFactor)$ [S1, unofficial].
        *   Calculate normalized line coordinates: $lineCoordinates = \max(minDistance / lineRadius, 0.0)$ [S1, unofficial].
        *   Calculate anti-aliasing factor: $EPSILON = clamp(getAntialiasingFactor(fragmentDistance / lineRadius), 0.0, 0.49)$ [S1, unofficial].
        *   Calculate coverage: $coverage = 1.0 - smoothstep(1.0 - 2.0 \times EPSILON, 1.0, lineCoordinates)$ [S1, unofficial].
        *   Set line color: $lineColor = vec4(lineBaseColor, coverage \times lodLevelOpacityFactor)$ [S1, unofficial].

6.  **Composite and Output:**
    *   Perform back-to-front alpha blending:
        $$blendedColor.a = lineColor.a + volumeColor.a \times (1.0 - lineColor.a)$$
        $$blendedColor.rgb = lineColor.rgb \times lineColor.a + volumeColor.rgb \times volumeColor.a \times (1.0 - lineColor.a)$$
        [S1, unofficial].
    *   If $blendedColor.a > 10^{-4}$, premultiply RGB: $blendedColor.rgb /= blendedColor.a$ [S1, unofficial].
    *   Call `gatherFragmentCustomDepth(blendedColor, fragmentDistance)` to store depth and color for further processing [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def face_based_edge_highlight(vFragmentPositionWorld, vVertexPositions, vVertexAttributes, vLineAttributes, vEdgeLodValues, uCellScalarTex, uLineWidth, uMaxLodValue, uSelectedLodValueFocus, uSelectedLodValueContext, uImportantLineBoostFactor, uEdgeColor, uBackgroundColor, uForegroundColor, uScalarMin, uScalarMax, contextFactor, focusFactor, fragmentDistance, uFovY, uViewportSize, uSphereCenter, uSphereRadius):
    # 1. Line Geometry
    distanceToFocusPointNormalized = length(vFragmentPositionWorld - uSphereCenter) / uSphereRadius
    eWidthPrime = uLineWidth * (-distanceToFocusPointNormalized * 0.3 + 1.0)
    lineRadius = eWidthPrime / 2.0
    
    # 2. Volume Color
    cellScalar = 0.25 * (vVertexAttributes[0] + vVertexAttributes[1] + vVertexAttributes[2] + vVertexAttributes[3])
    t = (cellScalar - uScalarMin) / max(uScalarMax - uScalarMin, 1e-6)
    baseColor = colormap(clamp(t, 0.0, 1.0))
    volumeColor = [baseColor[0], baseColor[1], baseColor[2], 1.0]
    volumeColor[3] *= contextFactor
    
    # 3. Init LOD and Distances
    LOD_EPSILON = 0.001
    discreteSelectedLodValueFocus = max(uSelectedLodValueFocus * uMaxLodValue, LOD_EPSILON)
    discreteSelectedLodValueContext = max(uSelectedLodValueContext * uMaxLodValue, LOD_EPSILON)
    
    minDistance = 1e9
    minDistanceIndex = 0
    minDistanceAll = 1e9
    minDistanceIndexAll = 0
    isLineNear = False
    
    # 4. Iterate Edges
    for i in range(4):
        currentDistance = get_distance_to_line_segment(vFragmentPositionWorld, vVertexPositions[i], vVertexPositions[(i + 1) % 4])
        
        # LOD Level Focus
        is_important = 1.0 if vLineAttributes[i] >= (1.0 - uImportantLineBoostFactor) else 0.0
        lodLevelFocus = max(is_important * uMaxLodValue, discreteSelectedLodValueFocus)
        
        # LOD Line Value
        lodLineValue = vEdgeLodValues[i] * uMaxLodValue
        
        # LOD Opacity Factor
        cond_context = 1.0 if lodLineValue <= discreteSelectedLodValueContext + LOD_EPSILON else 0.0
        cond_focus = 1.0 if lodLineValue <= discreteSelectedLodValueFocus + LOD_EPSILON else 0.0
        lodLevelOpacityFactor = mix(cond_context, cond_focus, focusFactor)
        
        drawLine = lodLevelOpacityFactor > 0.01
        
        if currentDistance < minDistance and drawLine:
            minDistance = currentDistance
            minDistanceIndex = i
            isLineNear = True
            
        if currentDistance < minDistanceAll:
            minDistanceAll = currentDistance
            minDistanceIndexAll = i
            
    # 5. Edge Highlight
    lineColor = [0.0, 0.0, 0.0, 0.0]
    
    if isLineNear and minDistance <= lineRadius:
        depthCueFactor = min(contextFactor, focusFactor)
        
        lodLineValue = vEdgeLodValues[minDistanceIndex]
        
        cond_context = 1.0 if lodLineValue <= uSelectedLodValueContext + LOD_EPSILON else 0.0
        cond_focus = 1.0 if lodLineValue <= uSelectedLodValueFocus + LOD_EPSILON else 0.0
        lodLevelOpacityFactor = mix(cond_context, cond_focus, focusFactor)
        
        lineBaseColor = uEdgeColor
        lineBaseColorFocus = mix(lineBaseColor, uBackgroundColor, 0.3)
        lineBaseColorContext = mix(baseColor, uForegroundColor, 0.3)
        lineBaseColor = mix(lineBaseColorContext, lineBaseColorFocus, focusFactor)
        
        lineCoordinates = max(minDistance / lineRadius, 0.0)
        
        # Anti-aliasing
        aa_factor_raw = get_antialiasing_factor(fragmentDistance / lineRadius, uFovY, uViewportSize)
        EPSILON = clamp(aa_factor_raw, 0.0, 0.49)
        coverage = 1.0 - smoothstep(1.0 - 2.0 * EPSILON, 1.0, lineCoordinates)
        
        lineColor = [lineBaseColor[0], lineBaseColor[1], lineBaseColor[2], coverage * lodLevelOpacityFactor]
        
    # 6. Composite
    blendedColor_a = lineColor[3] + volumeColor[3] * (1.0 - lineColor[3])
    blendedColor_rgb = [
        lineColor[0] * lineColor[3] + volumeColor[0] * volumeColor[3] * (1.0 - lineColor[3]),
        lineColor[1] * lineColor[3] + volumeColor[1] * volumeColor[3] * (1.0 - lineColor[3]),
        lineColor[2] * lineColor[3] + volumeColor[2] * volumeColor[3] * (1.0 - lineColor[3])
    ]
    
    if blendedColor_a > 1e-4:
        blendedColor_rgb = [c / blendedColor_a for c in blendedColor_rgb]
        
    return blendedColor_rgb + [blendedColor_a]

def get_distance_to_line_segment(p, a, b):
    ab = [b[j] - a[j] for j in range(3)]
    ap = [p[j] - a[j] for j in range(3)]
    ab_dot_ab = sum(x*x for x in ab)
    if ab_dot_ab == 0.0:
        return length([p[j] - a[j] for j in range(3)])
    t = sum(ap[j]*ab[j] for j in range(3)) / ab_dot_ab
    t = max(0.0, min(1.0, t))
    closest = [a[j] + ab[j]*t for j in range(3)]
    return length([p[j] - closest[j] for j in range(3)])

def get_antialiasing_factor(distance, fovY, viewportSize):
    return distance / viewportSize[1] * fovY

def colormap(t):
    # Viridis 5-stop colormap [G]
    stops = [(0.267004, 0.004874, 0.329415), (0.282634, 0.408305, 0.663236), (0.369214, 0.724701, 0.550856), (0.832055, 0.897605, 0.293494), (0.993248, 0.906157, 0.143936)]
    idx = t * 4.0
    i = int(idx)
    f = idx - i
    if i >= 4: i = 4
    c0 = stops[i]
    c1 = stops[min(i+1, 4)]
    return [c0[j] + (c1[j] - c0[j]) * f for j in range(3)]

def mix(a, b, t):
    return a + (b - a) * t

def smoothstep(edge0, edge1, x):
    t = clamp((x - edge0) / (edge1 - edge0), 0.0, 1.0)
    return t * t * (3.0 - 2.0 * t)

def clamp(x, a, b):
    return max(a, min(b, x))

def dot(a, b):
    return sum(a[j]*b[j] for j in range(len(a)))

def length(v):
    return sum(x*x for x in v) ** 0.5
```

### Parameters and Typical Values
*   `uLineWidth`: Base width of the edge highlight. Not given in the sources.
*   `uMaxLodValue`: Scaling factor for LOD values. Not given in the sources.
*   `uSelectedLodValueFocus` / `uSelectedLodValueContext`: Thresholds for focus and context LOD levels [S1, unofficial].
*   `uImportantLineBoostFactor`: Threshold to distinguish important lines. Not given in the sources.
*   `LOD_EPSILON`: Small constant (0.001) to prevent division by zero or invalid LOD states [S1, unofficial].

### Edge Cases and Pitfalls
*   **Precision:** The use of `1e-6` in scalar normalization prevents division by zero if `uScalarMax == uScalarMin` [S1, unofficial].
*   **Boundaries:** The `smoothstep` for coverage ensures smooth transitions at the edge boundaries, but `EPSILON` is clamped to 0.49 to prevent over-smoothing or artifacts [S1, unofficial].
*   **Degenerate Input:** If `vLineAttributes` are all below the importance threshold, no edges may be drawn, potentially reducing structural clarity [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per fragment, as it iterates over a fixed number of vertices (4) per cell [S1, unofficial].
*   **Memory Complexity:** $O(W \times H)$ for the PPLL buffers (Start Offset and Fragment Buffer) if used for fragment management, plus texture storage for cell scalars [S1, unofficial].
*   **Strengths:** Integrates seamlessly with volume rendering pipelines; allows for LOD-based detail control [S1, unofficial].
*   **Limitations:** Edge detection is local to the cell, which may miss global structural boundaries; performance depends on the cost of distance-to-line-segment calculations [S1, unofficial].

## Background
The method is derived from the "ClearView" rendering technique, specifically the "HexMeshUnified" fragment shader implementation, which integrates volume coloring with edge highlighting [S1, unofficial]. The implementation uses a Parallel Prefix List List (PPLL) structure for efficient fragment management, a technique common in high-performance graphics applications [S1, unofficial].

## Sources
[S1] Amory0709/geresim-spike, https://github.com/Amory0709/geresim-spike, fetched 2026-10-27 (unofficial)
