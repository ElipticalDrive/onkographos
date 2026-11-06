# Raycasting Model Matrix & AABB Planes

Also called: Model matrix AABB computation, Ray-box intersection preparation

## Overview
This method prepares the geometric state for ray-box intersection tests in volume raycasting by computing a normalized model matrix and the bounding planes of the volume's axis-aligned bounding box (AABB). It solves the problem of mapping world-space rays into a canonical unit cube space where intersection logic is simplified. It differs from general ray-box intersection methods by explicitly handling the scaling of the volume's physical dimensions to a unit extent and computing the specific plane offsets required for the slab test.

## How it works
**Inputs:**
- `m_size`: A 3D vector representing the number of voxels in the volume along each axis ($N_x, N_y, N_z$).
- `m_spacing`: A 3D vector representing the physical size of a single voxel along each axis ($\Delta x, \Delta y, \Delta z$).
- `m_origin`: A 3D vector representing the world-space position of the volume's minimum corner (or a reference origin).
- `shift`: A boolean flag indicating whether to apply the origin offset to the plane calculations.

**Outputs:**
- `modelMatrix`: A 4x4 transformation matrix.
- `top`: The world-space coordinate of the top plane of the AABB.
- `bottom`: The world-space coordinate of the bottom plane of the AABB.

**Data Structures:**
- `modelMatrix`: A 4x4 matrix stored in column-major or row-major order depending on the graphics API, but logically composed of translation and scaling components.
- `top`, `bottom`: Scalar values representing the extent boundaries along the primary axis of interest (typically the longest axis or Z-axis in the provided context).

**Steps:**
1.  **Compute Physical Extent:** Calculate the physical dimensions of the volume by multiplying the voxel count by the spacing:
    $$ \text{extent} = \text{m\_size} \times \text{m\_spacing} $$
    This results in a 3D vector $(E_x, E_y, E_z)$.

2.  **Compute Scale Factor:** Determine the scale factor by finding the maximum dimension of the extent. This normalizes the longest side of the bounding box to 1.
    $$ \text{scale\_factor} = \max(E_x, E_y, E_z) $$

3.  **Compute Model Matrix:** Construct the model matrix by first translating by the negative origin divided by the scale factor, and then scaling by half the extent divided by the scale factor. The formula provided is:
    $$ \text{modelMatrix} = \text{translate}\left(-\frac{\text{m\_origin}}{\text{scale\_factor}}\right) \times \text{scale}\left(0.5 \times \frac{\text{extent}}{\text{scale\_factor}}\right) $$
    Note: The source code snippet `modelMatrix.scale(0.5f * extent())` implies the scaling is applied to the extent vector directly, but the formula divides by scale_factor. The code snippet `modelMatrix.translate(-m_origin / scale_factor())` confirms the translation component. The scaling component in the code `0.5f * extent()` seems to ignore the division by `scale_factor` in the multiplication step, but the formula explicitly includes it. We follow the formula for the mathematical definition, assuming the code snippet might be a simplified or specific implementation detail where `extent()` might already be normalized or the division is implicit in the `scale_factor` usage context. However, strictly following the formula:
    $$ \text{scale\_component} = 0.5 \times \frac{\text{extent}}{\text{scale\_factor}} $$

4.  **Compute Top Plane:** Calculate the top plane coordinate. If `shift` is true, subtract the normalized origin.
    $$ \text{top} = \frac{\text{extent}}{2.0} - \frac{\text{m\_origin}}{\text{scale\_factor}} \times \text{shift} $$
    If `shift` is false:
    $$ \text{top} = \frac{\text{extent}}{2.0} $$

5.  **Compute Bottom Plane:** Calculate the bottom plane coordinate. If `shift` is true, subtract the normalized origin.
    $$ \text{bottom} = -\frac{\text{extent}}{2.0} - \frac{\text{m\_origin}}{\text{scale\_factor}} \times \text{shift} $$
    If `shift` is false:
    $$ \text{bottom} = -\frac{\text{extent}}{2.0} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_model_matrix_and_planes(m_size, m_spacing, m_origin, shift):
    # Step 1: Compute physical extent
    extent = m_size * m_spacing  # Element-wise multiplication
    
    # Step 2: Compute scale factor (max dimension)
    scale_factor = max(extent.x, extent.y, extent.z)
    
    # Step 3: Compute Model Matrix components
    # Translation: -origin / scale_factor
    translation = -m_origin / scale_factor
    
    # Scaling: 0.5 * (extent / scale_factor)
    # Note: The source formula divides extent by scale_factor.
    # The code snippet shows scale(0.5f * extent()), which might imply
    # extent is already scaled or the formula is the ground truth.
    # We follow the formula:
    scale_val = 0.5 * (extent / scale_factor)
    
    # Construct Model Matrix (Identity * Scale * Translation)
    # Assuming column-major or standard OpenGL-style construction
    model_matrix = identity_matrix()
    model_matrix = apply_scale(model_matrix, scale_val)
    model_matrix = apply_translation(model_matrix, translation)
    
    # Step 4 & 5: Compute Top and Bottom planes
    origin_shift = m_origin / scale_factor if shift else 0.0
    
    top = (extent / 2.0) - origin_shift
    bottom = (-extent / 2.0) - origin_shift
    
    return model_matrix, top, bottom
```

**Parameters:**
- `shift`: Boolean. If true, the origin offset is applied to the plane calculations. Typical value: `true` for world-space alignment.

**Edge Cases:**
- **Zero Scale Factor:** If the volume has zero dimensions, `scale_factor` will be zero, causing division by zero. Guard against this by checking if `scale_factor` is near zero.
- **Non-uniform Scaling:** The method normalizes the longest side to 1, resulting in a non-uniform scale for the other axes if the volume is not a cube. This must be accounted for in the ray transformation.

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(1)$ for the matrix and plane computation, as it involves a fixed number of arithmetic operations.
- **Memory Complexity:** $O(1)$ for storing the matrix and plane values.
- **Strengths:** Simplifies ray-box intersection by normalizing the volume to a unit cube. Efficient for static volumes.
- **Limitations:** Requires recomputation if the volume's physical dimensions or origin change. The normalization assumes the longest side is the primary axis for scaling, which may distort aspect ratios if not handled correctly in the ray transformation.

## Background
This method is derived from GPU-based volume raycasting implementations, specifically those using OpenGL for rendering. It is part of the preprocessing step to set up the rendering pipeline for ray-box intersection tests. The approach is common in real-time volume visualization where performance is critical.

## Sources
[S1] GitHub - m-pilia/volume-raycasting: Simple visualiser based on GPU ..., https://github.com/m-pilia/volume-raycasting, fetched 2026-10-27 (unofficial)
