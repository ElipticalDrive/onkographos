# Clipping plane for volume suppression

Also called: Clip Plane, Clipping Plane, Clipping Plane Widget, Cut Planes, Cutaway view generation, Frame type widget, Volume Clipping, Volume Cropping with Clipping Planes

## Overview
A clipping plane for volume suppression is an interaction technique that uses one or more arbitrarily oriented planes to cut away selected portions of a volume dataset, thereby uncovering internal details and generating cutaway views [S1, S5, unofficial]. Unlike [Cropping](1648-cropping.md), which restricts visibility to a rectilinear slab defined by planes parallel to the volume axes, clipping planes operate at arbitrary angles and can define a planar slice of finite thickness [S3, S5, unofficial]. This method is distinct from [Clipping](1444-clipping.md) in its specific application to volumetric data nodes where the interior of the clipped region retains data, often requiring capping of intersected geometry in medical contexts [S2].

## How it works

### Inputs and Outputs
**Inputs:**
*   Volume dataset (scalar or vector field).
*   Clipping plane definition: A normal vector $\mathbf{n} = (A, B, C)$ and position parameters defining the plane equation(s).
*   Optional: Falloff parameter ($D_{\text{fallo}}$) for smooth edges [S3, unofficial].
*   Optional: Region of interest (ROI) mask for superimposed data [S9, unofficial].

**Outputs:**
*   Rendered image where voxels outside the clipping region have suppressed opacity (typically zero) [S3, unofficial].
*   Interactive widget state reporting orientation, position, and mouse-click coordinates on the plane surface [S6].

### Data Structures

**Clipping Plane Definition (World Space):**
A plane is defined by the equation $Ax + By + Cz - D = 0$. For a cut plane of thickness, two parallel planes are used:
1.  $Ax + By + Cz - D_{\min} = 0$
2.  $Ax + By + Cz - D_{\max} = 0$

Where $D_{\min}$ and $D_{\max}$ measure the distance from the origin to the respective faces, and $D_{\min} \neq D_{\max}$ [S3, unofficial]. The thickness is $D_{\max} - D_{\min}$ [S3, unofficial].

**Widget State (for Interaction):**
*   **Orientation:** Represented by a normal vector or rotation matrix derived from the image's index-to-world transformation [S4, unofficial][S6].
*   **Position:** Defined by the distance from the volume origin along the normal [S2].
*   **Handles:** Corner balls for orientation modification and edge bars for position modification [S6].

**MPRPlane Node Structure (X3D Standard):**
*   `enabled` (SFBool, default TRUE)
*   `metadata` (SFNode, default NULL)
*   `normal` (SFVec3, default 0 0 1): Defines the plane's normal vector [S2].
*   `pos` (SFFloat, default 0.0, range [0,1]): Establishes the plane's position from the volume origin in the normal direction [S2].

**ijkPlanes Layout (for Coordinate Conversion):**
An array or tuple of six values: `[iMin, iMax, jMin, jMax, kMin, kMax]` representing the bounds of a cropping box in voxel indices [S4, unofficial].

### Steps

1.  **Initialization:**
    *   Copy the image data description to the widget to initialize its coordinate system [S4, unofficial].
    *   If converting from axis-aligned cropping bounds (`ijkPlanes`) to arbitrary planes:
        a. Extract `iMin, iMax, jMin, jMax, kMin, kMax` from `ijkPlanes` [S4, unofficial].
        b. Calculate the origin as `indexToWorld([iMin, jMin, kMin])` and the corner as `indexToWorld([iMax, jMax, kMax])` [S4, unofficial].
        c. Extract the rotation matrix from the image's index-to-world matrix [S4, unofficial].
        d. Rotate the standard basis vectors $(1,0,0), (-1,0,0), (0,1,0), (0,-1,0), (0,0,1), (0,0,-1)$ using the extracted rotation [S4, unofficial].
        e. Create `vtkPlane` (or equivalent) instances for each of the 6 faces (X min/max, Y min/max, Z min/max) using the rotated normals and the corresponding origin or corner [S4, unofficial].

2.  **Interaction Update:**
    *   Listen for modifications to the widget state's cropping planes [S4, unofficial].
    *   When modified, retrieve the planes and update the renderer's/mapper's clipping planes [S4, unofficial].
    *   The widget reports its orientation and position to the volume widget, which handles the actual clipping during drawing [S6].

3.  **Rendering (Sample Evaluation):**
    For each sample point $\mathbf{p}$ along a ray:
    a. Calculate the signed distance $d$ from $\mathbf{p}$ to the primary plane defined by normal $\mathbf{n}$ and offset $D$.
    b. Determine visibility based on the clipping mode:
        *   **Inside Cut Plane:** If the sample lies between $D_{\min}$ and $D_{\max}$, the opacity $\alpha$ is the value from the shading stage [S3, unofficial].
        *   **Outside Cut Plane:** If the sample is outside the cut plane, opacity is forced to zero [S3, unofficial].
        *   **Transition Region:** If the sample is within the falloff distance $D_{\text{fallo}}$ from the plane boundary, multiply the opacity by a correction factor that increases linearly from zero to one [S3, unofficial].
    c. **Superimposed Data (Optional):** If rendering physiological data within a known ROI (e.g., inside the lung), display the data only within this region and on the surface exposed by the cutting plane, typically as grayscale [S9, unofficial].

4.  **Querying (Optional):**
    *   Map a slice of the data to the arbitrary plane defined by the clip widget and blend it with the volume by a constant opacity value [S6].
    *   Report the spatial position of a mouse click on the clipping surface, providing an additional means of querying positions distinct from a 3D data probe [S6].

### Formulas

The two parallel faces of the cut plane are given by [S3, unofficial]:
$$ Ax + By + Cz - D_{\min} = 0 $$
$$ Ax + By + Cz - D_{\max} = 0 $$

The thickness of the cut plane is [S3, unofficial]:
$$ \text{Thickness} = D_{\max} - D_{\min} $$

The opacity correction factor in the transition region increases linearly from zero to one [S3, unofficial]. Let $\alpha_{\text{shade}}$ be the opacity from the shading stage. If the sample is within the falloff distance $D_{\text{fallo}}$ from the boundary, the final opacity $\alpha_{\text{final}}$ is adjusted. Outside the cut plane, $\alpha_{\text{final}} = 0$ [S3, unofficial].

### Parameters

*   **Single Clipping Plane:** A reasonable compromise between flexibility and usability [S1, unofficial]. VolumePro supports a single cut plane [S3, unofficial].
*   **Falloff Parameter ($D_{\text{fallo}}$):** Specifies the width of the transition from full opacity to none. Typical value: 5 voxels [S3, unofficial].
*   **MPRPlane `pos`:** Range [0, 1], representing normalized distance from origin [S2].

### Edge Cases and Pitfalls

*   **Geometry Capping:** In complex scenarios involving both geometry and volume data, clipped volume data always contains data inside the clipped region, whereas clipped polygonal objects could be empty. Medical applications expect intersected 3D geometry to be capped [S2].
*   **Precision:** Samples can be made visible only if they are outside the cut plane, or opacities near edges can fall off gradually for smooth edges [S3, unofficial].
*   **Coordinate Systems:** Converting between voxel indices and world space requires accurate extraction of the index-to-world rotation matrix [S4, unofficial].

## Subtopics

### [Clipping Cube and Cutting Plane Interaction](7062-clipping-cube-and-cutting-plane-interaction.md)
This method provides interactive, selective suppression of volume data using two complementary geometric primitives: an axis-aligned **Clipping Cube** and an arbitrary **Cutting Plane**. It solves the problem of isolating specific anatomical or structural regions (focus) while maintaining spatial context of the surrounding volume. It is distinguished from standard clipping by its support for non-uniform resizing of the cube via corner widgets and its integration with Non-Photorealistic Rendering (NPR) to render the "context" region as contours rather than transparent geometry, thereby reducing visual clutter.

### Note Cut
Cuts the sample after it has been played for x ticks [S7, unofficial].

### Spatial Domain Interaction
An interaction mode where a clipping plane slice is textured with map coordinates to help users understand the relationship between spatial and Transfer Function (TF) domains [S8].
*   **Steps:**
    1.  Rotate and translate the volume and place a clipping plane [S8].
    2.  Sample voxels on the clipping plane and retrieve their map coordinates [S8].
    3.  Map map coordinates to color channels: For Kohonen maps, x and y to Red and Green. For Spherical maps, x, y, and z to Red, Green, and Blue [S8].
    4.  Blend the textured slice with the rendered volume using a user-controlled opacity [S8].
    5.  Allow user to click on the clipping plane to set the Gaussian TF peak to the map coordinates of the pointed voxel [S8].
    6.  Display a white cross in the map representation corresponding to the mouse position on the clipping plane [S8].

### Superimposed Data Visualization with Cutting Planes
A visualization technique for superimposed anatomical and physiological data, displaying physiological data only within a known region of interest (e.g., inside the lung) and on a user-controlled cutting plane surface [S9, unofficial].
*   **Steps:**
    1.  Identify the region of interest (e.g., inside the lung) [S9, unofficial].
    2.  Display physiological data only within this region [S9, unofficial].
    3.  Render the data as grey-scale on the surface exposed by a user-controlled cutting plane [S9, unofficial].

### MPR Volume Rendering Style
An X3D node definition for Multi-Planar Reconstruction (MPR) volume rendering that allows users to declare arbitrary planes for visualization or filtering of regions in reconstructed planes [S2].
*   **Layout:**
    *   `MPRVolumeStyle` node inherits `X3DVolumeRenderStyleNode`. Fields: `SFBool enabled` (default TRUE), `SFNode metadata` (default NULL, type `X3DMetadataObject`), `SFNode transferFunction` (default NULL, types `X3DTexture2DNode` or `X3DTexture3DNode`), `MFNode planes` (default NULL, type `MPRPlane`) [S2].
    *   `MPRPlane` node inherits `X3DNode`. Fields: `SFBool enabled` (default TRUE), `SFNode metadata` (default NULL, type `X3DMetadataObject`), `SFVec3 normal` (default 0 0 1), `SFFloat pos` (default 0.0, range [0,1]). The normal attribute defines the plane's normal vector; the pos attribute establishes the plane's position from the volume origin in the normal direction [S2].

## Operating characteristics

*   **Time Complexity:** Rendering time depends on the number of samples per ray and the cost of evaluating the plane equation for each sample. The plane equation evaluation is $O(1)$ per sample.
*   **Memory Complexity:** Minimal additional memory for storing plane parameters ($O(1)$). Widget state requires storage for orientation and position.
*   **Performance:** Hardware-accelerated volume rendering can achieve interactive rates [S9, unofficial]. The use of a single clipping plane is a compromise between flexibility and usability [S1, unofficial].
*   **Strengths:** Uncover internal details, generate cutaway views, support arbitrary angles, smooth edges via falloff [S1, S3, S5, unofficial].
*   **Limitations:** Clipped volume data always contains data inside the clipped region, which may differ from polygonal clipping expectations [S2]. Complex scenarios may require capping [S2].
*   **When to use:** When internal structures need to be inspected without removing the entire outer shell, or when generating cross-sections at arbitrary angles [S1, S5, unofficial].
*   **When not to use:** When a rectilinear region of interest is sufficient (use Cropping) or when arbitrary polygon mesh clipping is required [S5, unofficial].

## Background

*   **Origin:** The concept of independently movable clipping planes for volume visualization was suggested by Chen et al. in 1988 [CMS88] and explicitly mentioned for volume suppression by Gobbetti and Kobbelt in 1996 [GK96] [S1, unofficial].
*   **Development:** VolumePro, a real-time ray-casting system, supported a single cut plane with configurable falloff [S3, unofficial]. The X3D standard added support for clipping planes in version 3.2, extending it to volumetric data nodes [S2].
*   **Usage:** Widely used in medical applications for inspecting anatomical structures, often combined with region-growing operations for segmentation [S1, unofficial][S2]. Also used in physics simulation to correlate visual output with results [S2].

## Sources

[S1] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
[S2] Community-driven extensions to the X3D volume rendering component, https://doi.org/10.1145/3055624.3075945, fetched 2026-11-02
[S3] The VolumePro Real-Time Ray-Casting System, https://www.merl.com/publications/docs/TR99-19.pdf, fetched 2026-10-26 (unofficial)
[S4] Muhammed-Magdy-Abdelhady/volume-rendering-web-application, https://github.com/Muhammed-Magdy-Abdelhady/volume-rendering-web-application, fetched 2026-10-27 (unofficial)
[S5] Hardware-Accelerated Volume Rendering, https://vcg.seas.harvard.edu/publications/20040101-hardware-accelerated-volume-rendering/paper, fetched 2026-10-27 (unofficial)
[S6] Interactive Volume Rendering Using Multi-Dimensional Transfer Functions, https://publications.sci.utah.edu/publications/jmk01/simian-paper.pdf, fetched 2026-10-27
[S7] AndreasVlcek/Voxel-Space-Engine-Comanche-Terrain-Rendering, https://github.com/AndreasVlcek/Voxel-Space-Engine-Comanche-Terrain-Rendering, fetched 2026-10-27 (unofficial)
[S8] Design of Multi-dimensional Transfer Functions Using Dimensional Reduction, https://doi.org/10.2312/vissym/eurovis07/131-138, fetched 2026-10-26
[S9] DTIC ADA208091: Interactive Visualization of 3D Medical Data, https://archive.org/details/DTIC_ADA208091, fetched 2026-10-26 (unofficial)
