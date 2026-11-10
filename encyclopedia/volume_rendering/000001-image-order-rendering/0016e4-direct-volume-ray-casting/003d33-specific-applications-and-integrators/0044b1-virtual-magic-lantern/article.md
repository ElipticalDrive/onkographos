# Virtual Magic Lantern

Also called: Magic Lantern metaphor

## Overview
The Virtual Magic Lantern (VML) is an interaction metaphor for volumetric data inspection that uses a virtual right circular cone to define a region of interest (ROI) [S1]. It solves the problem of simultaneously viewing a focal region with enhanced detail (via a secondary transfer function or shading style) and its surrounding context (via the primary transfer function) [S1]. Unlike the Virtual Magic Window (VMW), which restricts the ROI to the viewing direction, VML allows the cone to be positioned and oriented independently of the camera, providing richer contextual information at the boundary of the ROI, albeit with a performance penalty compared to classical ray casting [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A 3D volumetric dataset (texture), a primary transfer function (TF), a secondary transfer function (TF), cone geometry parameters (apex position, axis direction, aperture angle), and camera parameters.
*   **Outputs:** A rendered image where voxels inside the cone are colored using the secondary TF and voxels outside are colored using the primary TF.

**Data Structures:**
*   **Volume Texture:** 3D texture storing the scalar field values. In the source implementation, models are $512 \times 512 \times 512$ voxels [S1].
*   **Cone Geometry:** Defined by three parameters passed to the GPU:
    *   `apex`: The 3D position of the cone's tip (typically derived from a 3D pointer device like the Wanda) [S1].
    *   `axis`: The 3D direction vector of the cone's central axis (derived from the pointer device) [S1].
    *   `aperture`: The half-angle of the cone [S1].

**Algorithm Steps:**
1.  **Initialization:** Pass the primary and secondary transfer functions, the volume texture, and the cone geometry (apex, axis, aperture) to the GPU fragment shader [S1].
2.  **Ray Casting Loop:** For each pixel, cast a ray through the volume. Iterate along the ray at discrete sample positions [S1].
3.  **Point-in-Cone Test:** For each sample point $P$, determine if it lies inside the inspection cone.
    *   Let $A$ be the apex of the cone.
    *   Let $\vec{d}$ be the normalized axis direction of the cone.
    *   Let $\theta$ be the aperture angle.
    *   Calculate the vector $\vec{v} = P - A$.
    *   Calculate the angle $\alpha$ between $\vec{v}$ and $\vec{d}$ using the dot product: $\cos(\alpha) = \frac{\vec{v} \cdot \vec{d}}{|\vec{v}|}$.
    *   The point is inside the cone if $\alpha \le \theta$ (or equivalently $\cos(\alpha) \ge \cos(\theta)$) and the point is in the forward direction of the cone ($\vec{v} \cdot \vec{d} > 0$) [S1].
4.  **Color Calculation:**
    *   If the point is inside the cone, calculate the material color using the **inside** transfer function (secondary TF) [S1].
    *   If the point is outside the cone, calculate the material color using the **outside** transfer function (primary TF) [S1].
5.  **Shading and Compositing:** Apply Phong shading to the calculated material color and composite the result along the ray using standard volume ray casting accumulation [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader / Initialization
# Inputs: gl_FragCoord, VolumeTexture, Apex, Axis, Aperture, PrimaryTF, SecondaryTF

# Fragment Shader
void main() {
    // 1. Calculate ray origin and direction for the current pixel
    vec3 ray_origin = calculateRayOrigin(gl_FragCoord);
    vec3 ray_direction = calculateRayDirection(gl_FragCoord);
    
    // 2. Initialize accumulation variables
    vec4 accumulated_color = vec4(0.0);
    float accumulated_opacity = 0.0;
    vec3 position = ray_origin;
    
    // 3. Ray casting loop
    while (!endRayCasting(position, ray_origin, ray_direction)) {
        // Sample volume value
        float v = texture3D(VolumeTexture, position).a;
        
        // 4. Point-in-Cone Test
        // Vector from apex to current position
        vec3 v_vec = position - Apex;
        
        // Check if point is in front of the apex along the axis
        float dot_prod = dot(v_vec, Axis);
        bool is_forward = dot_prod > 0.0;
        
        // Calculate angle between v_vec and Axis
        float cos_alpha = dot(v_vec, Axis) / (length(v_vec) * length(Axis));
        // Clamp for numerical stability
        cos_alpha = clamp(cos_alpha, -1.0, 1.0);
        
        // Check if angle is within aperture
        float cos_theta = cos(Aperture);
        bool is_inside_angle = cos_alpha >= cos_theta;
        
        bool inside_cone = is_forward && is_inside_angle;
        
        // 5. Determine color based on inclusion
        vec4 material_color;
        if (inside_cone) {
            material_color = calculateInsideColor(v); // Secondary TF
        } else {
            material_color = calculateOutsideColor(v); // Primary TF
        }
        
        // 6. Apply Shading (Phong)
        // Note: calculatePhongShading modifies material_color based on position and normals
        material_color = calculatePhongShading(material_color, position);
        
        // 7. Composite
        // Standard front-to-back compositing
        float alpha = material_color.a;
        accumulated_color.rgb += (1.0 - accumulated_opacity) * alpha * material_color.rgb;
        accumulated_opacity += (1.0 - accumulated_opacity) * alpha;
        
        // Update position
        position += ray_direction * step_size;
    }
    
    gl_FragColor = accumulated_color;
}
```

**Parameters:**
*   **Aperture Angle:** Adjustable via a joystick on the Wanda device [S1].
*   **Apex Position:** Determined by the 3D pointer device (Wanda) [S1].
*   **Axis Direction:** Determined by the orientation of the Wanda device [S1].
*   **Resolution:** Tested on $512 \times 512 \times 512$ voxel models [S1].
*   **Window Size:** $900 \times 900$ pixels [S1].

**Edge Cases and Pitfalls:**
*   **Navigation Alignment:** When navigating through a model, the cone apex (Wanda position) may not point exactly in front of the user, causing the entire volume to be rendered with the primary transfer function if the cone misses the view frustum or the volume [S1].
    *   *Solution:* Generate the inspection cone with the apex at the observer position while keeping the direction guided by the hand [S1].
*   **Abrupt Region Changes:** The region pointed by the device may change more abruptly than expected if the pointed geometry is closer than supposed [S1].
    *   *Solution:* Place the apex of the cone at some distance back from the device [S1].
*   **Performance Penalty:** The inclusion test adds a rendering penalty compared to classical ray casting, but real-time framerates are maintained [S1].

## Subtopics

### [Virtual Magic Window](11345-virtual-magic-window.md)
A restricted implementation of VML where the virtual lantern aligns with the viewing direction, creating a window. It incurs no performance penalty compared to classical volume ray casting but provides less contextual information at the boundary of the ROI.

## Operating characteristics
*   **Time Complexity:** The method adds a point-in-cone test per sample point. The complexity is $O(N \cdot S)$, where $N$ is the number of pixels and $S$ is the number of samples per ray, plus the cost of the cone test [S1].
*   **Memory Complexity:** Requires storage for the volume texture and the cone parameters. No additional large data structures are required compared to classical ray casting [S1].
*   **Measured Performance:** Tested on a 3.00 GHz Intel Core 2 Duo PC, 8GB RAM, GeForce 8800 GTX (768MB RAM) [S1].
    *   *Feet Model (1 sample/voxel):* RC=32 fps, VML=17 fps, VMW=32 fps [S1].
    *   *Feet Model (2 samples/voxel):* RC=18.5 fps, VML=9.8 fps, VMW=18.5 fps [S1].
    *   *Manix Model (1 sample/voxel):* RC=18 fps, VML=13 fps, VMW=18 fps [S1].
    *   *Manix Model (2 samples/voxel):* RC=10 fps, VML=6.5 fps, VMW=10 fps [S1].
    *   *Head Model (1 sample/voxel):* RC=28 fps, VML=15.3 fps, VMW=28 fps [S1].
    *   *Head Model (2 samples/voxel):* RC=15 fps, VML=10 fps, VMW=15 fps [S1].
    *   *Rips Model (1 sample/voxel):* RC=32 fps, VML=14.8 fps, VMW=32 fps [S1].
    *   *Rips Model (2 samples/voxel):* RC=18 fps, VML=8.4 fps, VMW=18 fps [S1].
*   **Scaling:** Framerates slow down roughly 40% when the volume covers the whole viewport (close inspection) compared to viewing from outside [S1]. Disabling stereo rendering doubles framerates [S1].
*   **Strengths:** Intuitive interaction, provides simultaneous view of ROI and context, suitable for immersive VR setups (PowerWall) [S1].
*   **Limitations:** Performance penalty compared to classical ray casting and VMW; potential for abrupt region changes if not tuned [S1].
*   **When to use:** Immersive stereo systems for volumetric data inspection where contextual information is critical [S1].
*   **When not to use:** When maximum performance is required and contextual boundary information is less important (use VMW or classical RC) [S1].

## Background
The Virtual Magic Lantern was introduced by Andujar, Fairen, and Argelaguet in 2007 [S1]. It is tailored to facilitate volumetric data inspection, particularly in Virtual Reality setups with large screens (e.g., PowerWall) [S1]. The metaphor is inspired by the historical magic lantern, using a virtual illumination cone to highlight a focal region [S1]. It is implemented as a modification of GPU-based volume ray casting [S1].

## Sources
[S1] The virtual magic lantern, https://doi.org/10.1145/1643928.1643955, fetched 2026-10-28
