# Ghosting and Cutaway Techniques

Also called: Cutaway views, Ghosting

## Overview
Ghosting and Cutaway Techniques are illustrative volume rendering methods that resolve occlusion by selectively suppressing or removing foreground objects to reveal interior structures of interest [S1, book][S2]. Unlike standard clipping which discards context, these methods maintain spatial awareness through opacity modulation (ghosting) or explicit visual cues [S1, book][S2]. They distinguish themselves from sibling methods like [Geometric Buffers](781-geometric-buffers-g-buffers.md) by focusing on logical object decomposition (Selection, Ghost, Background) rather than purely geometric buffer manipulation, and from [Context-Preserving Volume Rendering](801-context-preserving-volume-rendering.md) by using discrete object separation rather than continuous opacity modulation based on shading intensity [S1, book][S2].

## Shared principles
The family operates on a three-object logical model: **Selection** (the object of interest), **Ghost** (the original position of the selection, often moved), and **Background** (the occluding environment) [S1, book][S2].

1.  **Object Decomposition**: The volume is conceptually split into these three sets. The Selection is the primary focus. The Ghost represents the "before" state or alternative position. The Background contains all other data [S1, book].
2.  **Occlusion Handling**:
    *   **Cutaways**: The Background is partially or fully removed in regions occluding the Selection or Ghost [S1, book].
    *   **Ghosting**: Instead of full removal, opacity of the Background is reduced in occluding regions to preserve edge cues and spatial context [S1, book].
3.  **Visual Conventions**: To clarify the relationship between Ghost and Selection, the system may render auxiliary geometry:
    *   **Boxes**: Bounding boxes for orientation [S2].
    *   **Arrows**: Indicating translation/movement [S2].
    *   **Fans**: Connecting shapes indicating alternative depictions [S2].
4.  **Rendering Strategy**: Efficient implementation often avoids costly per-voxel importance compositing by using static importance functions or depth-buffer-based occlusion culling across multiple rendering passes [S1, book].

## Subtopics

### [Feature Halos](799-feature-halos.md)
Feature Halos reinforces depth perception by darkening and increasing opacity in regions just outside strong volumetric features, obscuring background elements. It distinguishes itself by generating these effects directly during the illumination pass using local spatial properties (gradient orientation relative to view and neighbors), rather than relying on post-processing or explicit surface extraction. The method is inherently view-dependent, ensuring features do not cast halos on themselves.

### [Context-Preserving Volume Rendering](801-context-preserving-volume-rendering.md)
Context-Preserving Volume Rendering (CPVR) selectively reduces opacity in less important data regions to visualize interior structures while preserving exterior context. It solves the problem of context loss inherent in conventional clipping techniques by modulating sample opacity based on shading intensity, gradient magnitude, distance to the eye point, and previously accumulated opacity. Unlike clipping planes which remove context information, CPVR retains it by making homogeneous areas (high lighting intensity) transparent, allowing users to inspect the interior of an object without hiding occluded objects.

### [Exact Projection Full Opacity](1882-exact-projection-full-opacity.md)
Exact Projection Full Opacity is a non-photorealistic volume rendering technique that simplifies visual output by identifying a single primary front object and optionally one significant background object, blending them based on density and distance thresholds rather than performing full front-to-back accumulation. It differs from standard front-to-back compositing by skipping the accumulation of intermediate samples, storing only the first significant object (primary) and potentially one background object, then blending them directly to create a stylized, illustrative result.

### [Multi-object volume rendering with ghosting](6421-multi-object-volume-rendering-with-ghosting.md)
Multi-object volume rendering with ghosting is a technique for visualizing inter-penetrating volumetric objects by decomposing a scalar field into three distinct logical objects: selection, ghost, and background, allowing independent control of appearance and spatial transformation. It solves the problem of occlusion in complex scenes by selectively reducing opacity in occluding regions to preserve features such as edges, aiding mental reconstruction of structures and spatial location. Unlike [Cutaways and Ghosting](6479-cutaways-and-ghosting.md) which may remove occluding regions completely, this method allows smooth control from no ghosting (opacity not reduced) to full cutaway (occluding structures completely suppressed). It sets itself apart from single-pass multi-object compositing by using three separate rendering passes to avoid computational overhead and dynamic branching penalties.

### [Cutaways and Ghosting](6479-cutaways-and-ghosting.md)
Cutaways remove occluding objects to reveal structures of interest, using a simplified three-object setup (Background, Ghost, Selection) with static importance functions to skip costly importance compositing.

#### Cutaways
*   **Definition**: A technique where the occluding object is cut out to reveal the structure of interest, determined by an importance function [S1, book].
*   **Step**: In VolumeShop's simplified three-object setup, the importance function is statically defined to skip costly importance compositing, allowing efficient implementation [S1, book]. Cutaways are performed only on the background and can be independently defined for ghost and selection [S1, book].

#### Visual Conventions
*   **Definition**:
    *   **Boxes**: 3D bounding boxes providing cues on position and orientation. Useful during design but distracting during presentation [S2].
    *   **Arrows**: Depict translation between ghost and selection (original to current position). Used to suggest movement or insertion [S2].
    *   **Fans**: A connected pair of shapes (rectangles or circles) indicating a more detailed or alternative depiction. Constructed by connecting screen-space bounding rectangles of ghost and selection [S2].
*   **Step**: Control arrow curvature using screen-space depth difference to avoid very short arrows when ghost and selection project to nearby positions, creating bent arrows [S2].

## Background
The concepts of cutaways and ghosting originate from illustrative practices in information visualization and technical drawing. Viola et al. introduced importance-driven volume rendering as a general framework for determining which objects to cut [S1, book]. VolumeShop later provided an interactive system for direct volume illustration, implementing these concepts with a simplified three-object model to enable efficient real-time interaction [S1, book][S2]. These techniques are primarily used in medical visualization, engineering design review, and assembly instruction where understanding internal structure and spatial relationships of multiple components is critical [S1, book][S2].

## Sources
[S1] Illustrative Focus+Context Approaches in Interactive Volume Visualization, https://doi.org/10.4230/dfu.sciviz.2010.136, fetched 2026-10-26 (book)
[S2] VolumeShop: An Interactive System for Direct Volume Illustration, https://doi.org/10.1109/vis.2005.135, fetched 2026-10-26
