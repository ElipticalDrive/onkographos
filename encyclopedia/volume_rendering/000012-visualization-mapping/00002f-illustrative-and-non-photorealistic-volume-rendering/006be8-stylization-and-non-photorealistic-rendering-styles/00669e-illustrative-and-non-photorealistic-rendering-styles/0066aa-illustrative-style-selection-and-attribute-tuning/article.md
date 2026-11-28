# Illustrative Style Selection and Attribute Tuning

Also called: Object Classification for Illustrative Rendering, Style Selection Strategy

## Overview
This method provides a strategy for classifying anatomical objects into focus, context, and container categories to select appropriate rendering styles, shading, transparency, color, and silhouettes for expressive combined illustrations [S1, unofficial]. It solves the problem of visual clutter and ambiguity in complex volume-rendered scenes by establishing a visual hierarchy that guides the viewer's attention to high-interest structures while maintaining spatial context. It sets itself apart from sibling methods like [Volume Illustration](810-volume-illustration.md) by explicitly defining a tripartite classification system (focus/context/container) and mapping specific stylistic attributes (silhouette thickness, color saturation, shading type) to these roles rather than relying solely on geometric stylization or texture mapping.

## How it works
The method operates on a set of segmented objects extracted from medical scan data. It assumes the input includes object segmentation labels and optionally geometric properties (curvature, size). The output is a rendered image where each object is styled according to its assigned category.

### Data Structures
*   **Object List**: A collection of objects, each with properties:
    *   `category`: Enum {FOCUS, CONTEXT, CONTAINER}.
    *   `doi`: Degree of Interest (float, 0.0 to 1.0), where 1.0 is highest interest.
    *   `geometry_type`: Enum {POINT_BASED, POLYGONAL}.
    *   `curvature_variance`: Float (optional, for style selection).
    *   `nesting_level`: Integer (distance from viewer or hierarchical depth).

### Steps
1.  **Classify Objects**: Assign each object to one of three categories [S1, unofficial]:
    *   **Focus**: Objects of highest interest (e.g., internal organs like kidney, heart, colon).
    *   **Context**: Surrounding or neighboring objects that provide spatial reference.
    *   **Container**: Objects that enclose a subset of focus and context objects (e.g., skin). The current implementation typically defines only one container object.

2.  **Select Rendering Style**: Choose between point-based and polygonal-based rendering based on object properties [S1, unofficial]:
    *   Use **Point-Based Algorithm** if:
        *   Segmentation is provided.
        *   Object is of moderate or large size.
        *   Object does not exhibit extreme curvature variances (e.g., most internal organs).
        *   Object is a Focus object.
    *   Use **Polygonal-Based Rendering** if:
        *   Object is a Container.
        *   Object has sharp edges.
        *   Object is a relatively thin structure.
    *   *Note*: Point-based rendering avoids the triangulation step required by polygonal methods.

3.  **Select Shading Style**: Assign shading to enhance distinction between categories [S1, unofficial]:
    *   **Container**: Standard or discrete shading.
    *   **Context**: Cool-to-warm shaded. This style makes context objects clear and distinctive.
    *   **Focus**: Halftoning. This is particularly effective for large, innermost organs.
    *   *Constraint*: Do not apply the same shading style to both container and context objects, as they become indistinguishable.

4.  **Set Degree of Transparency**: Apply transparency based on category and Degree of Interest (doi) [S1, unofficial]:
    *   **Container**: Must be very translucent.
    *   **Focus**: Rendered almost or totally opaque.
    *   **Context and other non-container objects**: Apply the degree of interest concept (doi). Objects at the same nesting level are assigned transparencies inversely proportional to their doi.
    *   *Formula*: Let $T$ be transparency, $doi$ be degree of interest. For objects at the same nesting level, $T \propto \frac{1}{doi}$. Focus objects have $doi \approx 1.0$, thus $T \approx 0$ (opaque).

5.  **Set Color**: Assign colors to emphasize hierarchy [S1, unofficial]:
    *   **Container**: Light skin-like colors. This produces acceptable visual effects even if the container is not skin.
    *   **Focus**: Fully saturated, preferably warm colors (e.g., red or orange).
    *   **Context**: Cooler colors than focus objects.

6.  **Set Silhouette**: Render silhouettes to improve expressiveness and shape perception [S1, unofficial]:
    *   **Focus and Context**: Always render with outlined silhouettes.
    *   **Container**: May be rendered without silhouettes.
    *   **Silhouette Thickness**: Proportional to the degree of object importance. Focus objects receive thicker silhouettes.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_illustrative_style(object_list):
    """
    Applies illustrative style selection and attribute tuning to a list of objects.
    Assumes object_list contains objects with properties: category, doi, geometry, curvature_variance.
    """
    for obj in object_list:
        # Step 1: Classification is assumed to be pre-computed based on semantic labels
        # obj.category is one of: 'FOCUS', 'CONTEXT', 'CONTAINER'
        
        # Step 2: Select Rendering Style
        if obj.category == 'CONTAINER':
            obj.rendering_style = 'POLYGONAL'
        elif obj.category == 'FOCUS':
            # Point-based preferred for focus objects of moderate/large size without extreme curvature
            if obj.size >= THRESHOLD_MODERATE and obj.curvature_variance < THRESHOLD_EXTREME:
                obj.rendering_style = 'POINT_BASED'
            else:
                obj.rendering_style = 'POLYGONAL'
        else: # CONTEXT
            # Context objects often have sharp edges or are thin structures in this context
            if obj.has_sharp_edges or obj.is_thin:
                obj.rendering_style = 'POLYGONAL'
            else:
                obj.rendering_style = 'POINT_BASED' # Fallback, though context often benefits from polygonal for sharpness
        
        # Step 3: Select Shading Style
        if obj.category == 'CONTAINER':
            obj.shading_style = 'STANDARD'
        elif obj.category == 'CONTEXT':
            obj.shading_style = 'COOL_TO_WARM'
        elif obj.category == 'FOCUS':
            obj.shading_style = 'HALFTONING'
        
        # Step 4: Set Degree of Transparency
        if obj.category == 'CONTAINER':
            obj.transparency = 0.9 # Very translucent
        elif obj.category == 'FOCUS':
            obj.transparency = 0.0 # Almost or totally opaque
        else:
            # Inverse proportionality to doi for same nesting level
            # Note: In practice, this requires normalizing doi within the nesting level
            # For simplicity, assuming doi is normalized 0-1
            obj.transparency = 1.0 - obj.doi
        
        # Step 5: Set Color
        if obj.category == 'CONTAINER':
            obj.color = LIGHT_SKIN_COLOR
        elif obj.category == 'FOCUS':
            obj.color = WARM_SATURATED_COLOR # e.g., Red or Orange
        else: # CONTEXT
            obj.color = COOLER_COLOR
        
        # Step 6: Set Silhouette
        if obj.category in ['FOCUS', 'CONTEXT']:
            obj.has_silhouette = True
            # Thickness proportional to importance (doi)
            obj.silhouette_thickness = obj.doi * MAX_THICKNESS
        else: # CONTAINER
            obj.has_silhouette = False
            obj.silhouette_thickness = 0
```

### Parameters and Edge Cases
*   **Thresholds**: `THRESHOLD_MODERATE`, `THRESHOLD_EXTREME`, `MAX_THICKNESS` are implementation-specific constants not defined in the source. [G]
*   **Nesting Level**: The transparency formula requires objects to be grouped by nesting level. If nesting levels are not explicitly defined, a fallback strategy (e.g., global doi normalization) must be used. [G]
*   **Container Definition**: The source notes that only one container object is typically defined (skin). If multiple containers exist, the "very translucent" rule applies to all. [S1, unofficial]
*   **Curvature Variance**: Extreme curvature variances disqualify an object from point-based rendering, even if it is a focus object. This prevents artifacts on highly irregular surfaces. [S1, unofficial]

## Operating characteristics
*   **Time Complexity**: Depends on the rendering style. Point-based rendering avoids triangulation, potentially reducing preprocessing time for large datasets. Polygonal rendering requires mesh generation, which is more expensive. [S1, unofficial]
*   **Memory Complexity**: Point-based methods may require storing more points if high resolution is needed, while polygonal methods store mesh topology. [S1, unofficial]
*   **Strengths**: Provides a clear visual hierarchy. Enhances shape perception through silhouettes. Distinguishes between object types via color and shading. [S1, unofficial]
*   **Limitations**: Relies on accurate segmentation and classification. The "one container" assumption may not hold for all anatomical scenes. [S1, unofficial]
*   **When to Use**: Medical visualization where specific organs (focus) need to be highlighted against surrounding structures (context) and body surface (container). [S1, unofficial]
*   **When Not to Use**: Scenes without clear hierarchical relationships or where photorealism is preferred over illustrative clarity.

## Background
The method originates from work on expressive anatomical illustrations based on scanned patient data, presented in the context of computer and robot-assisted surgery [S1, unofficial]. It builds on the concept of Degree of Interest (doi) [32] for transparency control. The approach integrates point-based and polygonal rendering algorithms with various shading styles (standard, cool-to-warm, halftoning) to create a flexible illustration system. It was tested with diverse models extracted from segmentation data. [S1, unofficial]

## Sources
[S1] GMS | GMS Current Topics in Computer and Robot Assisted Surgery | Expressive anatomical illustrations based on scanned patient..., https://www.egms.de/static/de/journals/curac/2006-1/curac000009.shtml, fetched 2026-10-30 (unofficial)
