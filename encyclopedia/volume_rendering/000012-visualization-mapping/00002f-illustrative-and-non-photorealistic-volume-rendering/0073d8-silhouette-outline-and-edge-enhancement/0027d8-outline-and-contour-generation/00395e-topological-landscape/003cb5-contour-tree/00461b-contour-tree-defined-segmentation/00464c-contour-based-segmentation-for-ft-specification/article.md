# Contour-Based Segmentation for FT Specification

Also called: Ropinski et al. technique

## Overview
This method uses user-drawn contours to separate portions of a volume before transfer function (FT) specification, allowing manual parameter definition on the isolated region [S1, unofficial]. It sets itself apart from sibling methods by using direct geometric separation via 2D sketches rather than topological tree traversal or neural network training [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D volume dataset, a 2D slice or projection view of the volume, and user-drawn 2D contours [S1, unofficial].
- Output: A transfer function (FT) defined for the region enclosed by the contours [S1, unofficial].

**Data Structures:**
- Volume: 3D grid of scalar values [S1, unofficial].
- Contours: 2D closed curves defined on a specific slice or projection plane [S1, unofficial].
- FT: Mapping from scalar value to opacity and color [S1, unofficial].

**Steps:**
1. User draws contours to separate portions of the volume [S1, unofficial].
2. The desired region is isolated [S1, unofficial].
3. User manually specifies the FT for the isolated region [S1, unofficial].

**Formulas:**
Not given in the sources.

**Parameters:**
Not given in the sources.

**Edge Cases and Pitfalls:**
Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def contour_based_ft_specification(volume, slice_view, user_contours):
    # Step 1: Create 2D mask from contours
    mask_2d = create_2d_mask_from_contours(slice_view, user_contours)
    # Step 2: Extrude 2D mask to 3D volume
    mask_3d = extrude_mask_to_volume(mask_2d, volume)
    # Step 3: Isolate region
    isolated_region = multiply_volume_by_mask(volume, mask_3d)
    # Step 4: Manual FT specification
    ft = manual_ft_specification(isolated_region)
    return ft

def create_2d_mask_from_contours(slice_view, contours):
    height, width = len(slice_view), len(slice_view[0])
    mask_2d = [[False] * width for _ in range(height)]
    for contour in contours:
        fill_contour_interior(mask_2d, contour)
    return mask_2d

def fill_contour_interior(mask, contour):
    height, width = len(mask), len(mask[0])
    for y in range(height):
        for x in range(width):
            if is_point_inside_polygon(x, y, contour):
                mask[y][x] = True

def is_point_inside_polygon(px, py, contour):
    # [G] Standard ray-casting algorithm to determine interior pixels
    intersections = 0
    n = len(contour)
    for i in range(n):
        x1, y1 = contour[i]
        x2, y2 = contour[(i + 1) % n]
        if ((y1 > py) != (y2 > py)) and (px < (x2 - x1) * (py - y1) / (y2 - y1) + x1):
            intersections += 1
    return intersections % 2 == 1

def extrude_mask_to_volume(mask_2d, volume):
    # [G] Extrude 2D mask along the slice normal (assumed z-axis)
    depth = len(volume[0][0])
    height, width = len(mask_2d), len(mask_2d[0])
    mask_3d = [[[False] * depth for _ in range(width)] for _ in range(height)]
    for z in range(depth):
        for y in range(height):
            for x in range(width):
                mask_3d[y][x][z] = mask_2d[y][x]
    return mask_3d

def multiply_volume_by_mask(volume, mask):
    # [G] Element-wise multiplication to zero out voxels outside the mask
    return [[[volume[y][x][z] if mask[y][x][z] else 0.0 for z in range(len(volume[0][0]))] for x in range(len(volume[0]))] for y in range(len(volume))]

def manual_ft_specification(isolated_region):
    # [G] Interactive UI step where user adjusts opacity/color parameters for active voxels
    return {"opacity": 1.0, "color": [1.0, 1.0, 1.0]}
```

## Subtopics

### [Neural Network Sketch-Based Specification](11743-neural-network-sketch-based-specification.md)
A technique using a neural network to adjust the transfer function (FT) based on a sketch drawn on a volume visualization, training the network to emphasize marked regions. It implements multidimensional transfer functions as back-propagation neural networks to decide voxel visibility, where training is supervised by user-selected voxels on 2D slices. It sets itself apart from sibling methods by replacing manual parameter tuning or geometric contour separation with a learned mapping from visual sketches to multidimensional visibility decisions.

## Operating characteristics
Not given in the sources.

## Background
This technique was introduced by Ropinski et al. in 2008 [S1, unofficial]. It builds on earlier sketch-based specification work by Tzeng, Lum, and Ma (2003) [S1, unofficial].

## Sources
[S1] A volume exploration approach based on clustering and dimensional reduction to support the definition of multidimensional transfer functions, http://hdl.handle.net/11449/153618, fetched 2026-10-26 (unofficial)
