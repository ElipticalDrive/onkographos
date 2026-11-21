# Selection and Manipulation
## Overview
Selection and Manipulation enables users to identify specific subsets of volumetric data or graphical primitives for modification, highlighting, or analysis, and to interactively alter their geometric or visual properties. It distinguishes itself from sibling methods like Cropping or Clipping by focusing on semantic or spatial selection of data elements (e.g., splats, cells, widgets) rather than global view frustum or plane-based visibility culling, allowing for targeted editing and complex interaction states.
## Shared principles
The family operates on the principle that user input defines a selection criterion, which is then mapped to a subset of the underlying data structure. This subset is either visually highlighted (feedback) or used as a target for subsequent manipulation commands. The core mechanism involves:
1.  **Input Mapping:** Translating user gestures (mouse, hand, brush) into a selection domain (screen space, world space, or data index space).
2.  **Intersection/Containment Test:** Determining which data elements satisfy the selection criterion (e.g., spatial overlap, Hilbert code range, or occlusion).
3.  **State Management:** Maintaining the selection state, which may be integrated with the scene, decoupled, or pinned, influencing how the selection behaves during view transformations.
## Subtopics

### [Brushing and Linking with Hilbert Codes](10912-brushing-and-linking-with-hilbert-codes.md)
This method enables interactive selection in volume rendering by mapping Regions of Interest (ROIs) identified in a 1D data view to corresponding spatial cells in a 3D view. It is distinguished from siblings by representing selection boundaries as 1D Hilbert code intervals rather than 3D geometric primitives, solving the problem of efficiently linking disparate visualizations without transmitting large lists of cell indices. Choose this when working with structured or AMR data where 1D data views (like Dynamic Volume Lines) are available and compact selection transmission is required.

### [3D Widget Interaction Model](11643-3d-widget-interaction-model.md)
The 3D Widget Interaction Model provides a framework for creating interactive graphical objects (widgets) that exist within a 3D scene and can be directly manipulated by the user. It is distinguished from siblings by associating observers with a `vtkRenderWindow` to maintain a scene representation, enabling direct manipulation of data structures such as streamlines or clipping volumes. Choose this when a robust, event-driven framework for creating and manipulating 3D graphical handles is needed.

### Direct Selection and Deletion
**Definition:** An approach where users instantly highlight all splats that intersect with the brush during selection, contrasting with indirect approaches that compute intersections after brushing [S1].
**Steps:**
*   Selection/deletion brushes may be either circular (screen space) or spherical (world space) [S1].
*   Users may choose selections based on splat centers or their border [S1].
*   In case of circular brushes and splat-border intersection tests, apply an approximation that checks whether the center of the brush circle is inside an ellipse that is enlarged by the radius of the brush [S1].
*   Sort all splats by their deletion flag, putting the deleted splats towards the end of the buffer [S1].
*   Unmap and release physical allocations at the buffer's end in steps of the GPU's granularity [S1].
**Pitfalls:** Using radix sort is not required; developers may implement this functionality with GPU-friendly stream compaction [S1].

### Selection Interaction States
**Definition:** Integrated: Selection acts as fully belonging to the 3D scene. Movement depends on distance to origin; can move out of viewport or be occluded [S2]. Decoupled: Selection fully decoupled from scene. Independently manipulated, not affected by viewing transformation. Useful for fixed orientation [S2]. Pinned: Hybrid mode. Object pinned to current image space position. On-screen location static, but affected by rotations (same relative rotation as viewpoint) [S2].

## Background
The concepts of selection and manipulation in volume rendering and 3D visualization have evolved from simple ray-casting techniques to complex, state-aware interaction models. Early work on image plane selection and ray cursors established the foundation for 3D interaction [S3, unofficial]. The integration of selection with data structures like Hilbert codes allows for efficient linking between 1D and 3D views [S1]. The 3D Widget Interaction Model, popularized by libraries like VTK, provides a standardized way to handle complex user interactions in 3D scenes [S1]. Direct selection techniques, particularly for emerging media like Gaussian Splatting, emphasize immediate visual feedback to enhance user experience in interactive editing tasks [S1].
## Sources
[S1] Splatshop: Efficiently Editing Large Gaussian Splat Models, https://doi.org/10.1111/cgf.70214, fetched 2026-10-31
[S2] VolumeShop: An Interactive System for Direct Volume Illustration, https://doi.org/10.1109/vis.2005.135, fetched 2026-10-26
[S3] Interaction with volumetric displays, http://hdl.handle.net/1807/16733, fetched 2026-10-26 (unofficial)
