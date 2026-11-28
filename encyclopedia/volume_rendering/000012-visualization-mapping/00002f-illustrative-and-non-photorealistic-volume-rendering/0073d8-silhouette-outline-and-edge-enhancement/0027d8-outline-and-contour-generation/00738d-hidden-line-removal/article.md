# Hidden Line Removal

Also called: Wireframe Occlusion Culling

## Overview
Hidden Line Removal is a rendering technique used in wireframe representations to hide line segments that are occluded by solid objects in the scene [S1, S2]. It distinguishes itself from sibling methods like [Object Contour NPR](787-object-contour-npr.md) and [Image Space Outline Generation](788-image-space-outline-generation.md) by operating on explicit geometric primitives (lines/edges) rather than extracting contours from volumetric data or image-space depth buffers [S1, S2].

## How it works
**Inputs and Outputs:**
- Input: A set of wireframe line segments representing the edges of 3D objects.
- Input: A depth buffer (z-buffer) or ray-casting intersection data representing the solid geometry of the scene.
- Output: A subset of the input line segments that are visible to the camera.

**Data Structures:**
- `LineSegment`: A structure containing two 3D points ($P_1, P_2$) defining the segment endpoints.
- `DepthBuffer`: A 2D array of depth values ($z$) corresponding to pixel coordinates, or a list of intersected primitives for ray-based approaches.

**Steps:**
1. **Rasterization/Projection:** Project the 3D line segments onto the 2D image plane.
2. **Occlusion Test:** For each line segment, determine if any part of it lies behind a solid object.
   - *Rasterization Approach:* Render the solid geometry to a depth buffer. Then, for each line segment, sample the depth buffer at the segment's pixel locations. If the segment's depth is greater than the depth buffer value at any sampled point, that portion is occluded.
   - *Ray Casting Approach:* For each line segment, cast rays from the camera through the segment's pixels. If a ray intersects a solid object closer than the segment, the segment is occluded.
3. **Clipping:** Clip the line segments to remove occluded portions. Segments entirely occluded are discarded.
4. **Rendering:** Render the remaining visible line segments.

**Formulas:**
- Depth comparison: $z_{seg} > z_{buffer}(x, y)$ implies occlusion at pixel $(x, y)$.
- Line parameterization: $L(t) = P_1 + t(P_2 - P_1)$ for $t \in [0, 1]$.

**Parameters:**
- *Not given in the sources.*

**Edge Cases and Pitfalls:**
- *Precision:* Depth buffer precision issues may cause z-fighting, leading to incorrect occlusion decisions for lines very close to surfaces [G].
- *Boundaries:* Lines exactly on the boundary of occlusion may flicker if depth values are unstable [G].
- *Degenerate Input:* Zero-length line segments should be ignored [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hidden_line_removal(lines, depth_buffer, image_width, image_height):
    visible_lines = []
    for line in lines:
        p1, p2 = line.p1, line.p2
        # Project p1 and p2 to screen space
        s1 = project_to_screen(p1)
        s2 = project_to_screen(p2)
        
        # Check if line is behind the camera
        if s1.z <= 0 or s2.z <= 0:
            continue
            
        # Sample depth buffer along the line
        occluded = False
        num_samples = max(abs(s2.x - s1.x), abs(s2.y - s1.y))
        if num_samples == 0:
            num_samples = 1
            
        for i in range(num_samples + 1):
            t = i / num_samples
            sx = s1.x + t * (s2.x - s1.x)
            sy = s1.y + t * (s2.y - s1.y)
            sz = s1.z + t * (s2.z - s1.z)
            
            # Check bounds
            if 0 <= sx < image_width and 0 <= sy < image_height:
                buffer_z = depth_buffer[int(sy)][int(sx)]
                # If line is behind the surface, it is occluded
                if sz > buffer_z:
                    occluded = True
                    break
        
        if not occluded:
            visible_lines.append(line)
            
    return visible_lines
```

## Subtopics

### [Object Contour NPR](787-object-contour-npr.md)
Extracts contours based on object boundaries; choose when you need stylized edges from volumetric data.

### [Image Space Outline Generation](788-image-space-outline-generation.md)
Uses depth/normal discontinuities; choose for fast, resolution-independent outlines.

### [Silhouette Rendering](7959-silhouette-rendering.md)
Renders only the outer boundary; choose for clear object separation.

### [Suggestive Contour Extraction](13101-suggestive-contour-extraction.md)
Highlights concavities; choose for enhanced shape perception.

## Operating characteristics
- **Time Complexity:** $O(L \cdot P)$ where $L$ is the number of line segments and $P$ is the number of pixels sampled per segment.
- **Memory Complexity:** $O(W \cdot H)$ for the depth buffer, where $W$ and $H$ are image dimensions.
- **Strengths:** Provides clear visibility of internal structures in wireframe models; simple to implement with standard rasterization.
- **Limitations:** Computationally expensive for high-resolution scenes with many line segments; requires accurate depth information.
- **When to use:** When rendering wireframe models where internal occlusion is critical for understanding structure.
- **When not to use:** When performance is critical and wireframe detail is not essential; when using solid rendering.

## Background
Hidden Line Removal is a standard feature in 3D visualization software, notably documented in ParaView [S1, S2]. It has been a part of the ParaView user interface since at least version 5.8.1 [S1] and remains in version 6.2.0 [S2]. It is often used in conjunction with parallel projection for technical drawings [S1, S2].

## Sources
[S1] 4. Displaying data — ParaView Documentation 5.8.1 documentation, https://docs.paraview.org/en/v5.9.1/UsersGuide/displayingData.html, fetched 2026-10-29
[S2] 5. Displaying data — ParaView Documentation 6.2.0 documentation, https://docs.paraview.org/en/latest/UsersGuide/displayingData.html, fetched 2026-10-29
