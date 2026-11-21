# Polygon-based Cropping
Also called: CutLineInside, CutLineOutside, CutingImagedata, Voxel-wise 2D Polygon Clipping

## Overview
Polygon-based cropping is a voxel-wise volume rendering technique that masks voxels based on a user-defined 2D polygon drawn in the display (screen) coordinate space. It solves the problem of interactive, free-form region-of-interest selection by converting screen-space constraints into volume-space visibility masks. It differs from planar clipping methods by supporting arbitrary convex or non-convex polygonal shapes rather than just axis-aligned or arbitrary planar cuts, and it operates by explicitly iterating over voxels to test their projected positions against the polygon boundary [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A `vtkImageData` object containing the volume scalar data, a `vtkRenderer` for coordinate transformation, a `QPolygonF` defining the 2D clipping boundary in screen coordinates, and an integer `type` flag determining the clipping mode.
- Output: The modified `vtkImageData` where voxels outside the valid region are set to zero (transparent), effectively cropping the volume.

**Data Structures:**
- `vtkImageData`: Stores voxel data. The implementation assumes the scalar type is `unsigned char` [S1, unofficial]. If the type differs, pointer casting must be adjusted [S1, unofficial].
- `QPolygonF`: A 2D polygon defined by a sequence of `QPointF` vertices.
- `vtkCoordinate`: Used to transform world coordinates to display coordinates.

**Steps:**
1. Retrieve volume metadata:
   - Get dimensions: `img_dims = image_data->GetDimensions()`
   - Get spacing: `img_spacing = image_data->GetSpacing()`
   - Get origin: `img_origian = image_data->GetOrigin()` [S1, unofficial]
2. Iterate over all voxel coordinates $(i, j, k)$ within the volume bounds. Note: Direct indexing is required; `vtkImageIterator` cannot retrieve spatial positions [S1, unofficial].
3. Compute the world position of the current voxel:
   $$
   \text{world\_pos} = [i \cdot \text{img\_spacing}[0] + \text{img\_origian}[0], \quad j \cdot \text{img\_spacing}[1] + \text{img\_origian}[1], \quad k \cdot \text{img\_spacing}[2] + \text{img\_origian}[2]]
   $$
   [S1, unofficial]
4. Transform the world position to display coordinates:
   - Use `coordinate->GetComputedDoubleDisplayValue(renderer)` to obtain `display_pos` [S1, unofficial].
5. Check if `display_pos` lies inside the user-drawn polygon:
   - Call `polygon.containsPoint(QPointF(display_pos[0], display_pos[1]), Qt::OddEvenFill)` [S1, unofficial].
6. Apply clipping based on `type`:
   - If `type == RenderWidget::CutLineInside` and the point is inside the polygon, keep the voxel value. If outside, set `*pPixel = 0`.
   - If `type == RenderWidget::CutLineOutside` and the point is outside the polygon, set `*pPixel = 0`. If inside, keep the voxel value.
   [S1, unofficial]
7. After modification, update the volume and re-add it to the renderer:
   - `volume->Update()`
   - `renderer->RemoveVolume(volume)`
   - `renderer->AddVolume(volume)`
   [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def polygon_based_cropping(image_data, volume, renderer, polygon, clip_type):
    # Retrieve metadata
    img_dims = image_data.GetDimensions()
    img_spacing = image_data.GetSpacing()
    img_origian = image_data.GetOrigin()
    
    # Create coordinate converter
    coordinate = vtkCoordinate()
    coordinate.SetCoordinateSystemToWorld()
    coordinate.SetRenderer(renderer)
    
    # Iterate over all voxels
    for k in range(img_dims[2]):
        for j in range(img_dims[1]):
            for i in range(img_dims[0]):
                # Compute world position
                world_pos = [
                    i * img_spacing[0] + img_origian[0],
                    j * img_spacing[1] + img_origian[1],
                    k * img_spacing[2] + img_origian[2]
                ]
                
                # Transform to display coordinates
                # Note: GetComputedDoubleDisplayValue expects world coordinates
                coordinate.SetCoordinate(world_pos[0], world_pos[1], world_pos[2])
                display_pos = coordinate.GetComputedDoubleDisplayValue(renderer)
                
                # Check if point is inside polygon
                # Qt::OddEvenFill is used for winding rule
                contains_point = polygon.containsPoint(
                    QPointF(display_pos[0], display_pos[1]), 
                    Qt.OddEvenFill
                )
                
                # Get pointer to voxel data
                # Assumes unsigned char type [S1, unofficial]
                pixel_ptr = image_data.GetScalarPointer(i, j, k)
                
                # Apply clipping logic
                if clip_type == RenderWidget.CutLineInside:
                    if not contains_point:
                        pixel_ptr[0] = 0
                elif clip_type == RenderWidget.CutLineOutside:
                    if contains_point:
                        pixel_ptr[0] = 0
                        
    # Update volume and renderer
    volume.Update()
    renderer.RemoveVolume(volume)
    renderer.AddVolume(volume)
```

**Parameters:**
- `clip_type`: Enum or integer flag. `CutLineInside` (keep inside polygon), `CutLineOutside` (keep outside polygon) [S1, unofficial].
- `fill_rule`: `Qt::OddEvenFill` is used for point-in-polygon testing [S1, unofficial].

**Edge Cases and Pitfalls:**
- The implementation assumes the bounding box origin starts at `(0,0,0)`. Production code must validate actual bounds [S1, unofficial].
- Do not use `vtkImageIterator` as it cannot retrieve spatial positions; direct indexing is required [S1, unofficial].
- The scalar type is assumed to be `unsigned char`. If the scalar type differs, the pointer cast and assignment must be adjusted [S1, unofficial].

## Subtopics

### [Convex VOI Clipping via Weighted Sweep Graph](15939-convex-voi-clipping-via-weighted-sweep-graph.md)
A CPU-based algorithm that intersects parallel sampling planes against a convex VOI using a Weighted Sweep Graph to exploit coherence between planes, reducing GPU load by transmitting only valid portions. Choose this for large volumes where reducing ray casting overhead is critical and the VOI is convex.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the total number of voxels in the volume, as every voxel is visited and transformed.
- **Memory Complexity:** $O(1)$ auxiliary space, as it modifies the volume in place.
- **Performance:** The method is CPU-bound due to the per-voxel coordinate transformation and polygon containment test. It is suitable for interactive editing but may be slow for very large volumes compared to GPU-based clipping.
- **Strengths:** Supports arbitrary 2D polygonal shapes in screen space, allowing for intuitive free-form cropping.
- **Limitations:** Requires full volume iteration; not suitable for real-time rendering of large datasets without optimization. Assumes specific data types and coordinate systems.
- **When to use:** Interactive volume editing where the user draws a custom 2D shape to define the region of interest.
- **When not to use:** Real-time rendering of large volumes where performance is critical; use GPU-based clipping or the Convex VOI Clipping method instead.

## Background
The method is described in the open-source project `kpengk/VolumeClipping` [S1, unofficial]. It is part of a larger system for interactive volume visualization using VTK and Qt. The approach leverages VTK's coordinate transformation capabilities and Qt's polygon geometry classes to implement a simple, voxel-wise clipping mechanism.

## Sources
[S1] kpengk/VolumeClipping, https://github.com/kpengk/VolumeClipping, fetched 2026-10-26 (unofficial)
