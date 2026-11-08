# DoubleNeRF

Also called: Double NERF

## Overview
DoubleNeRF is a method for synthesizing dynamic scenes by representing them as the superposition of two or more static neural radiance fields, specifically separating a static background from a moving object with a static shape [S1]. Unlike [NeRF++](6198-nerf.md), which separates scenes using a spherical boundary, DoubleNeRF separates content semantically (background vs. foreground object) without spherical constraints, and unlike [D-NeRF](5573-d-nerf.md) (not linked in tree but referenced in sources), it does not use time as an input or map canonical space to deformed space, assuming instead that the object's appearance is independent of its position relative to the background [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Two sets of oriented images: $A_b$ (background images, excluding the dynamic object) and $A_o$ (object images, with background masked out using alpha channels) [S1].
*   **Outputs:** A dynamic radiance field $n_d$ capable of rendering novel views of the dynamic scene [S1].

**Data Structures and Coordinate Systems:**
The method relies on two distinct coordinate systems:
1.  **Background Scene Coordinate System ($O_b X_b Y_b Z_b$):**
    *   Origin: Center of mass of the scene 3D model on the ground level [S1].
    *   $X_b$ axis: Directed collinear to the projection of the optical axis of the first camera in set $A_b$ [S1].
    *   $Z_b$ axis: Normal to the surface of the ground [S1].
    *   $Y_b$ axis: Completes the right-handed coordinate system [S1].
2.  **Object Coordinate System ($O_o X_o Y_o Z_o$):**
    *   Origin: Located on the projection of the center of mass of the object to the ground plane [S1].
    *   $X_o$ axis: Directed toward the positive direction of the construction axis of the object (e.g., forward motion) [S1].
    *   $Z_o$ axis: Normal to the surface of the ground [S1].
    *   $Y_o$ axis: Completes the right-handed coordinate system [S1].

**Method Steps:**
1.  **Estimate Background Field:** Train a neural radiance field $n_b$ using the image set $A_b$. This field operates in the background coordinate system $O_b X_b Y_b Z_b$ [S1].
2.  **Estimate Object Field:** Train a neural radiance field $n_o$ using the image set $A_o$. The training data for $n_o$ uses alpha channels to mask the background. A critical constraint is applied: for any point in $n_o$ not located on the object, the volume density is equal to 0 [S1]. This field operates in the object coordinate system $O_o X_o Y_o Z_o$ [S1].
3.  **Define Transformation:** Define the rigid body transformation from the background coordinate system to the object coordinate system using a rotation matrix $R_{bo}$ and a translation vector $T_{bo}$ [S1].
4.  **Superposition:** Compute the dynamic radiance field $n_d$ by summing the background field and the transformed object field [S1].

**Formulas:**
The dynamic radiance field $n_d$ at a scene point $(x, y, z)$ is defined as [S1]:

$$ n_d(x, y, z, \Theta, \phi) = n_b(x, y, z, \Theta, \phi) + n_o(x'_o, y'_o, z'_o, \Theta, \phi) $$

where $(x'_o, y'_o, z'_o)$ are the object coordinates transformed from the scene coordinates $(x, y, z)$. The transformation is given by [S1]:

$$ X'_o = R_{bo} \cdot X_o + T_{bo} $$

where $X_o = [x, y, z]^T$ is the point in the background coordinate system, $R_{bo}$ is the rotation matrix, and $T_{bo}$ is the translation vector [S1].

**Assumptions and Constraints:**
*   The surface brightness and reflections of the moving object are independent of the location of the object with respect to the background scene [S1].
*   The object has a static shape [S1].
*   The object field $n_o$ contributes zero density outside the object's geometry [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def double_nerf_render(camera_pose, scene_point):
    """
    Renders a single point in the scene using the DoubleNeRF superposition.
    
    Args:
        camera_pose: Camera extrinsics and intrinsics for the target view.
        scene_point: 3D point (x, y, z) in the background coordinate system.
        
    Returns:
        color: RGB color at the scene point.
        density: Volume density at the scene point.
    """
    # 1. Evaluate Background Field
    # n_b takes scene coordinates directly
    bg_color, bg_density = neural_radiance_field_b(scene_point, camera_pose)
    
    # 2. Transform Scene Point to Object Coordinates
    # X_o is the scene point in background coordinates
    X_o = scene_point
    # Apply rigid transformation from background to object frame
    X_prime_o = R_bo @ X_o + T_bo
    
    # 3. Evaluate Object Field
    # n_o takes transformed coordinates
    obj_color, obj_density = neural_radiance_field_o(X_prime_o, camera_pose)
    
    # 4. Superposition
    # The dynamic field is the sum of the two fields
    total_density = bg_density + obj_density
    total_color = bg_color + obj_color
    
    return total_color, total_density

def neural_radiance_field_b(point, camera_pose):
    """
    Standard NeRF forward pass for the background field.
    """
    # Implementation details of standard NeRF [S1]
    # Input: point (x,y,z), camera direction
    # Output: color (RGB), density (sigma)
    pass

def neural_radiance_field_o(point, camera_pose):
    """
    Standard NeRF forward pass for the object field.
    Constraint: density is 0 if point is not on object surface.
    """
    # Implementation details of standard NeRF [S1]
    # Input: point (x'_o, y'_o, z'_o), camera direction
    # Output: color (RGB), density (sigma)
    pass
```

## Operating characteristics
*   **Complexity:** The rendering complexity is effectively doubled compared to a single NeRF, as two neural network forward passes are required per sample (one for background, one for object) [S1].
*   **Performance:** Evaluated on the SematicVoxels dataset using city scenes as background and cars as foreground dynamic objects [S1]. The method achieves and surpasses state-of-the-art results in dynamic scene synthesis metrics including PSNR, SSIM, LPIPs, and FID [S1].
*   **Strengths:**
    *   Handles dynamic scenes with static-shape objects without requiring time as an input [S1].
    *   Avoids the spherical boundary artifacts and limitations of [NeRF++](6198-nerf.md) [S1].
    *   Separates background and foreground cleanly, allowing independent optimization [S1].
*   **Limitations:**
    *   Limited to scenes with a static background and objects with static shapes (no non-rigid deformation) [S1].
    *   Requires separate training data for background and object, with the object data needing background masking [S1].
    *   Assumes object appearance is invariant to its position relative to the background, which may not hold for complex lighting interactions (e.g., shadows cast by the object on the background) [S1].

## Background
DoubleNeRF was proposed in 2023 by authors associated with the ISPRS Archives [S1]. It was implemented using the TensorFlow library [S1]. The method was developed to address the limitation of standard NeRF, which cannot synthesize dynamic scenes [S1]. It builds on the concept of NeRF++ but diverges by using semantic separation rather than geometric (spherical) separation [S1]. It is distinct from D-NeRF, which uses time and canonical space deformation [S1].

## Sources
[S1] DOUBLE NERF: REPRESENTING DYNAMIC SCENES AS NEURAL RADIANCE FIELDS, https://doi.org/10.5194/isprs-archives-xlviii-2-w3-2023-115-2023, fetched 2026-10-27
