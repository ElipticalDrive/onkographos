# Selection and Manipulation


## Overview
This family covers techniques for identifying specific subsets of volumetric data or rendering primitives for modification, deletion, or annotation, and for manipulating the spatial relationship between the selection and the scene. It solves the problem of precise user intent translation in 3D space, where traditional 2D screen-space interactions must be mapped to complex 3D structures. It is distinguished from [Cropping](1648-cropping.md) by its focus on discrete object or primitive-level targeting rather than global volume partitioning, and from [3D Cursor](1655-3d-cursor.md) by its broader scope including occlusion-based and state-based selection modes.

## Shared principles
The core mechanism involves mapping user input to a set of selection criteria $S$ within the data domain $D$. Unlike global clipping, selection operates on a subset of elements (e.g., splats, voxels, or mesh primitives). The implementation must handle the coordinate transformation between user input space (screen or hand) and world space, often requiring ray casting or spatial indexing. A key challenge is the feedback loop: the visual representation of the selection must update in real-time to confirm the user's intent, which may involve temporary highlighting or state flags before final application (e.g., deletion or duplication).

## Subtopics

### Direct Selection and Deletion
Direct selection and deletion allows users to instantly highlight and remove specific rendering primitives, such as splats, during the brushing operation. This approach contrasts with indirect methods where intersections are computed only after the brush stroke is complete.

*   **Definition**: An approach where users instantly highlight all splats that intersect with the brush during selection, contrasting with indirect approaches that compute intersections after brushing [S1].
*   **Brush Geometry**: Selection/deletion brushes may be either circular (screen space) or spherical (world space) [S1]. Users may choose selections based on splat centers or their border [S1].
*   **Intersection Approximation**: In case of circular brushes and splat-border intersection tests, apply an approximation that checks whether the center of the brush circle is inside an ellipse that is enlarged by the radius of the brush [S1].
*   **Deletion Implementation**: Sort all splats by their deletion flag, putting the deleted splats towards the end of the buffer [S1]. Unmap and release physical allocations at the buffer's end in steps of the GPU's granularity [S1].
*   **Implementation Note**: Using radix sort is not required; developers may implement this functionality with GPU-friendly stream compaction [S1].

### Selection Interaction States
Selection interaction states define how a selected object or annotation behaves relative to the scene's viewing transformation. This is particularly relevant for annotations and illustrative elements in volume rendering.

*   **Integrated**: The selection acts as fully belonging to the three-dimensional scene. This is intuitive, but has certain drawbacks. For example, when the viewpoint is rotated, the selection’s movement is dependent on its distance to the origin. It can easily move out of the viewport or can be occluded by other objects [S2].
*   **Decoupled**: The opposite to the integrated approach is to fully decouple the selection from the scene. It can be independently manipulated and is not affected by the viewing transformation. This is, for instance, useful when it is required to depict an object at a specific orientation regardless of the viewing transformation [S2].
*   **Pinned**: A useful hybrid between the two modes above is to allow the object to be pinned to its current position in image space. Its on-screen location remains static, but it is still affected by rotations. A rotation of the viewpoint causes the same relative rotation of the object. For example, this can be used to generate a special view which always shows the part of an object facing away from the viewer in the background object [S2].

### Image Plane Selection
Image plane selection techniques project 3D objects onto a 2D image plane to facilitate selection, often extending to immersive 3D environments using hand-tracking or stereoscopic displays.

*   **Definition**: A selection technique where 3D objects are projected onto a 2D image plane, and a cursor defines a perpendicular ray to select the first intersected object [S3, unofficial]. A selection technique where users select objects that visually lie behind a hand-attached cursor, defined by the vector joining the user's eye and hand [S3, unofficial].
*   **Stereoscopic Extension**: For stereoscopic displays, define the image plane as the viewpoint from only a single eye to create a one-eyed cursor [S3, unofficial].
*   **Aperture-based Variant**: Cast cone from eye through finger. Distance between finger and eye controls cone divergence angle. Use device orientation for disambiguation when multiple objects intersect [S3, unofficial].
*   **Head Crusher**: Select objects lying between index finger and thumb [S3, unofficial].
*   **Sticky Finger**: Select objects occluded by the tip of the index finger [S3, unofficial].
*   **Lifting Palm**: Flatten hand and position palm underneath object to select it [S3, unofficial].
*   **Framing Hands**: Use both hands to define a bounding box around object using thumbs and index fingers [S3, unofficial].

## Background
Selection and manipulation in volume rendering evolved from early 2D image editing and 3D modeling interfaces. The concept of "direct" vs. "indirect" selection in splat-based rendering was formalized in recent works like Splatshop [S1], addressing the need for immediate visual feedback in VR and large-scale data editing. The classification of selection states (Integrated, Decoupled, Pinned) originated from systems like VolumeShop [S2], designed to handle annotations and illustrative elements in scientific visualization. Image plane and occlusion-based selection techniques trace back to early VR interaction research, including the "one-eyed cursor" for stereoscopic displays [S3, unofficial] and hand-based occlusion techniques like "Head Crusher" and "Sticky Finger" [S3, unofficial]. These methods address the ambiguity of 3D selection by leveraging 2D projection or physical occlusion cues.

## Sources
[S1] Splatshop: Efficiently Editing Large Gaussian Splat Models, https://doi.org/10.1111/cgf.70214, fetched 2026-10-31
[S2] VolumeShop: An Interactive System for Direct Volume Illustration, https://doi.org/10.1109/vis.2005.135, fetched 2026-10-26
[S3] Interaction with volumetric displays, http://hdl.handle.net/1807/16733, fetched 2026-10-26 (unofficial)
