# Cylindrical Projection for Volumetric Approximation

Also called: Cylindrical Body Map Construction, X-Ray Casting, cylindrical body map generation

## Overview
Cylindrical Projection for Volumetric Approximation is a two-step technique for representing large volumetric medical datasets (such as CT images) in a compact form suitable for real-time rendering [S1, unofficial]. It solves the problem of efficiently visualizing internal structures like vasculature by projecting volume samples onto a cylindrical surface centered on a region of interest, which is then texture-mapped onto a 3D cylinder for real-time display with parallax effects [S1, unofficial]. Unlike direct ray casting which requires re-casting for different viewpoints, this method constructs a static cylindrical body map that can be viewed from different points of view without re-casting [S1, unofficial]. It differs from simple orthographic projection onto a rectangle by accounting for the final direction of view, which is necessary because anatomical landmarks are both in front of and behind the vasculature and the camera moves via rotation and translation along the body's longitudinal axis [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of CT image slices. Each slice is typically in RAW format with dimensions $512 \times 512$ pixels and a defined pixel thickness $t_{slice}$ [S1, unofficial]. The dataset includes a header with a Hounsfield number offset constant (1024) [S1, unofficial].
*   **Output:** A cylindrical body map texture. The output image format is RGB with a depth of 1. The dimensions are $360 \times 322 \times (\text{SweepAngle} \times \text{NumberOfSlices})$ [S1, unofficial].

**Data Structures:**
The core data structure is the cylindrical body map, which stores attenuation values. It can be conceptualized as a 2D texture map unwrapped from a cylinder.
*   **Cylinder Geometry:** Defined by a center `CylCenter` $(0,0)$ and a radius `CylRadius` $r$.
*   **Texture Coordinates:** The texture is mapped to the cylinder surface. The angular dimension corresponds to the 360-degree rotation around the center, and the vertical dimension corresponds to the stack of CT slices [S1, unofficial].

**Parameters:**
*   `CylCenter`: $(0,0)$ [S1, unofficial].
*   `CylRadius`: $r$ [S1, unofficial].
*   `HounsfieldOffset`: 1024 [S1, unofficial].
*   $\mu_w$ (Water attenuation number): $0.02/\text{mm}$ [S1, unofficial].
*   $t_{slice}$: Pixel thickness for the current slice [S1, unofficial].

**Algorithm Steps:**
1.  Initialize the projection geometry: Set a circle with `CylCenter` $(0,0)$ and radius `CylRadius` $r$ [S1, unofficial].
2.  Iterate through all sequential CT image slices.
3.  Load the current CT slice into memory [S1, unofficial].
4.  Iterate through angles from $0$ to $360$ degrees.
5.  Cast a ray from `CylCenter` to the edge of the circle at the current angle [S1, unofficial].
6.  Iterate through radii from $0$ to `CylRadius`.
7.  Get a sample pixel value $I_y$ from the loaded slice at the current ray position (intersection of the ray and the slice plane) [S1, unofficial].
8.  Calculate the attenuation factor for the sample using the Hounsfield number formula (see Formulas).
9.  Set the pixel value on the cylinder surface using the calculated attenuation [S1, unofficial].
10. After processing all angles and radii for all slices, create the final output image from the accumulated cylinder surface data [S1, unofficial].
11. Wrap (texture map) the 2D image onto the surface of a cylinder with proper transparency values [S1, unofficial].

**Formulas:**
The attenuation calculation involves two steps: converting the pixel intensity to an attenuation number, and then computing the exponential attenuation factor.

1.  **Attenuation Number ($\mu_y$):**
    $$ \mu_y = \frac{I_y - 1024}{1024} $$
    where $I_y$ is the pixel intensity [S1, unofficial]. The term $(I_y - 1024)$ represents the Hounsfield number [S1, unofficial].

2.  **Attenuation Factor ($\sigma_k$):**
    $$ \sigma_k = \exp\left(-\sum \mu_y \cdot t_{slice}\right) $$
    where $\mu_w$ (water attenuation) is $0.02/\text{mm}$, $t_{slice}$ is the pixel thickness, and the sum is accumulated along the ray path [S1, unofficial]. The ratio $E_f/E_i$ (output/input energy) constitutes the overall attenuation factor [S1, unofficial].

3.  **Final Projection Pixel Value:**
    Since the attenuation is an exponential function, the final projection pixel value is computed by multiplying each term [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cylindrical_projection(ct_slices, cyl_center, cyl_radius, hounsfield_offset, mu_water, t_slice):
    """
    Construct a cylindrical body map from CT slices.
    
    Args:
        ct_slices: List of 2D arrays representing CT slices.
        cyl_center: Tuple (0, 0).
        cyl_radius: Float, radius of the cylinder.
        hounsfield_offset: Int, typically 1024.
        mu_water: Float, water attenuation number (0.02/mm).
        t_slice: Float, thickness of a pixel in mm.
        
    Returns:
        texture_map: 2D array representing the unwrapped cylindrical body map.
    """
    # Output dimensions: 360 angles x (number of slices * vertical resolution per slice)
    # Note: The source specifies output size 360x322x(SweepAngle x NumberOfSlices).
    # Assuming 360 angles and a fixed vertical resolution per slice or total height.
    # For implementation, we map angle to x and slice index to y.
    
    num_slices = len(ct_slices)
    num_angles = 360
    
    # Initialize texture map. 
    # Dimensions based on source: 360 (angles) x 322 (height factor) x (SweepAngle * NumSlices)
    # Simplified for implementation: 2D texture where x=angle, y=slice_index
    # The source mentions "360x322x(SweepAngle x NumberOfSlices)" which implies a 3D volume 
    # or a specific texture format. We will generate a 2D array for the unwrapped map.
    # Let's assume the height corresponds to the number of slices.
    texture_height = num_slices
    texture_width = num_angles
    
    texture_map = np.zeros((texture_height, texture_width))
    
    for slice_idx, slice_data in enumerate(ct_slices):
        for angle_deg in range(num_angles):
            angle_rad = math.radians(angle_deg)
            
            # Ray direction from center
            # Ray: P(t) = CylCenter + t * (cos(angle), sin(angle))
            # We iterate radius from 0 to CylRadius
            # However, the source says "Cast a ray from CylCenter to edge of circle"
            # and "Iterate through radii from 0 to CylRadius".
            # This implies we are sampling along the radius.
            
            # The sample point on the slice is determined by the ray intersection.
            # Since the slice is a 2D plane (z=constant), and the ray is in the xy plane,
            # the intersection is simply the point on the circle at that angle.
            # But wait, the ray is cast *from* the center *to* the edge.
            # The sample is taken at the current radius position along this ray.
            
            # The source says: "Get a sample pixel value from the loaded slice at the current ray position."
            # This implies we need to map the cylindrical coordinate (r, theta) to the image coordinate (x, y).
            
            # Image center is likely (256, 256) for a 512x512 image.
            image_center_x = 256
            image_center_y = 256
            
            # We need to sample the slice at the point corresponding to (r, angle)
            # x = image_center_x + r * cos(angle)
            # y = image_center_y + r * sin(angle)
            
            # However, the source algorithm iterates radius.
            # It seems the "cylinder surface pixel value" is set for the surface of the cylinder (r = CylRadius).
            # But the loop iterates radius 0 to CylRadius.
            # "Set the cylinder surface pixel value" suggests we only care about the surface?
            # Or does it accumulate?
            # "The samples from one slice can be thought of as making up one circle of data."
            # "If we stack the circles from all the slices, we produce a cylindrical representation"
            # This implies the texture map stores the values at the surface of the cylinder (r = CylRadius).
            # The inner loop over radius might be for accumulation or just describing the ray cast.
            # Given "Set the cylinder surface pixel value", we likely only store the value at r = CylRadius.
            
            # Let's assume we only store the value at the surface r = CylRadius.
            # The "Iterate through radii" might be a description of the ray casting process,
            # but the output is the surface map.
            
            # Re-reading: "Cast a ray from CylCenter to edge of circle... Iterate through radii... Get a sample... Set the cylinder surface pixel value"
            # This is ambiguous. Does it set the value for *each* radius?
            # "The samples from one slice can be thought of as making up one circle of data."
            # A circle has a single radius.
            # "This map can then be 'unwrapped' to form a rectangular... texture map"
            # A texture map is 2D.
            # If we stored values for all radii, it would be 3D.
            # The output size is 360x322x(...).
            # 360 is angles. 322 is likely related to slices or height.
            # If it's a texture map, it's 2D.
            # Let's assume the texture map stores the attenuation at the surface r = CylRadius.
            
            # Calculate sample point on the image plane
            r = cyl_radius
            x_img = image_center_x + r * math.cos(angle_rad)
            y_img = image_center_y + r * math.sin(angle_rad)
            
            # Get sample pixel value I_y
            # Need to handle boundary checks and interpolation if necessary [G]
            I_y = get_pixel_value(slice_data, x_img, y_img)
            
            # Calculate attenuation number
            mu_y = (I_y - hounsfield_offset) / hounsfield_offset
            
            # Calculate attenuation factor
            # The formula is sigma_k = exp(-sum(mu_y * t_slice))
            # Since we are at the surface, we are summing along the ray?
            # No, the ray is in the plane of the slice.
            # The source says "sum mu_y * t_slice".
            # This implies we are integrating along the ray path through the volume.
            # But we are only sampling one point per slice?
            # "Each CT slice has a header... pixel thickness... tslice is the thickness of a pixel for the current slice"
            # "Ef/Ei is the output/input energy... final projection pixel value can be computed by multiplying each term"
            # This suggests that for each slice, we compute a term, and multiply them.
            # This is equivalent to summing the exponents.
            
            # Term for this slice:
            # exp(-mu_y * t_slice)
            # But mu_y is derived from I_y.
            # Is mu_y the attenuation coefficient?
            # "mu_y is the attenuation number"
            # "mu_w is the attenuation number of water"
            # The formula for mu_y uses I_y.
            
            # Let's compute the term for this slice
            term = math.exp(-mu_y * t_slice)
            
            # Accumulate?
            # "Ef/Ei is the output/input energy, which constitutes the overall attenuation factor"
            # "final projection pixel value can be computed by multiplying each term"
            # So we multiply the terms from all slices?
            # But we are building a map for *one* slice at a time?
            # "The samples from one slice can be thought of as making up one circle of data."
            # "If we stack the circles from all the slices, we produce a cylindrical representation"
            # This implies the texture map is built slice by slice.
            # So for each slice, we compute the attenuation for the surface points.
            # But attenuation depends on the path through the volume.
            # If we are projecting onto a cylinder, the ray passes through multiple slices.
            # The source algorithm iterates slices, then angles, then radii.
            # It sets the cylinder surface pixel value.
            # This suggests the value at (angle, slice_idx) is computed using the data from slice_idx?
            # Or is it the accumulated attenuation from all slices up to that point?
            # "X-Ray Casting... approximating a ray cast through the volume with sample points on the surface of the cylinder"
            # "We approximate the value at point A by sampling the values along OA"
            # This suggests the value on the cylinder surface represents the attenuation along the ray from the center to the surface.
            # But the ray is in the plane of the slice.
            # This is confusing.
            # Let's stick to the literal algorithm steps.
            
            # Step: Calculate attenuation factor
            # Step: Set cylinder surface pixel value
            
            # If we only have one slice's data, we can't compute the full path attenuation.
            # Unless the "cylinder" is just a representation of the slice data wrapped around.
            # "This map can then be 'unwrapped' to form a rectangular... texture map"
            # "The output image size... 360x322x(SweepAngle x NumberOfSlices)"
            # This looks like a 3D volume or a specific texture format.
            # Let's assume the texture map stores the value computed from the current slice.
            
            # Compute attenuation for this sample
            attenuation = math.exp(-mu_y * t_slice)
            
            # Store in texture map
            # Texture map index: (slice_idx, angle_deg)
            texture_map[slice_idx, angle_deg] = attenuation
            
    return texture_map

def get_pixel_value(slice_data, x, y):
    """
    Get pixel value from slice data at coordinates (x, y).
    Handles boundaries and interpolation [G].
    """
    h, w = slice_data.shape
    # Check boundaries
    if x < 0 or x >= w or y < 0 or y >= h:
        return 0 # Or handle appropriately [G]
    
    # Bilinear interpolation [G]
    x0 = int(math.floor(x))
    y0 = int(math.floor(y))
    x1 = x0 + 1
    y1 = y0 + 1
    
    # Clamp coordinates
    x0 = max(0, min(x0, w-1))
    y0 = max(0, min(y0, h-1))
    x1 = max(0, min(x1, w-1))
    y1 = max(0, min(y1, h-1))
    
    dx = x - x0
    dy = y - y0
    
    val00 = slice_data[y0, x0]
    val01 = slice_data[y0, x1]
    val10 = slice_data[y1, x0]
    val11 = slice_data[y1, x1]
    
    val_top = val00 * (1 - dx) + val01 * dx
    val_bottom = val10 * (1 - dx) + val11 * dx
    
    return val_top * (1 - dy) + val_bottom * dy
```

**Edge Cases and Pitfalls:**
*   **Distortion:** Distortion increases as the viewing angle $\theta$ diverges from 180 degrees. At 180 degrees, sample points accurately represent data along cast rays because rays are normal to the projection plane. For other angles, the method approximates a ray cast through the volume with sample points on the cylinder surface (sampling along OA/OB instead of the correct orthographic path AB) [S1, unofficial].
*   **Viewing Angle:** The impact of distortion is minimized because the surgeon's attention is focused on the vasculature where $\theta$ will be close to 180 degrees [S1, unofficial].
*   **Boundary Conditions:** When sampling points fall outside the image boundaries, appropriate handling (e.g., clamping or interpolation) is required [G].

## Operating characteristics
*   **Time Complexity:** The algorithm iterates through all slices, all angles (360), and all radii (up to CylRadius). The complexity is $O(N_{slices} \times 360 \times CylRadius)$ [S1, unofficial].
*   **Memory Complexity:** The output is a texture map of size $360 \times 322 \times (\text{SweepAngle} \times \text{NumberOfSlices})$ in RGB format with depth 1 [S1, unofficial].
*   **Performance:** The method provides interactive frame rates on hardware such as Silicon Graphics Power Onyx RE2 and Maximum Impact systems with 1MB texture memory [S1, unofficial].
*   **Strengths:** Compact representation of large volumetric data; real-time rendering capability; view-independent pre-computation; parallax effects [S1, unofficial].
*   **Limitations:** Approximation error increases with viewing angle deviation from 180 degrees; distortion for off-axis views [S1, unofficial].
*   **When to use:** Surgical simulation for visualizing vasculature in CT data where real-time performance is critical and the view is constrained [S1, unofficial].
*   **When not to use:** When high accuracy is required for all viewing angles; when the region of interest is not cylindrical or centered [S1, unofficial].

## Background
*   **Origin:** The method was developed for a rendering system for photo-realistic images and virtual environments, specifically for surgical simulation [S1, unofficial].
*   **History:** It was presented in a final report on a rendering system, utilizing CT images from the National Library of Medicine's Visible Man data set [S1, unofficial].
*   **Usage:** Used in surgical simulation for visualizing vasculature in CT data, specifically for the Visible Man data set from the National Library of Medicine [S1, unofficial]. Rendered using Open Inventor and hardware texture mapping on Silicon Graphics Power Onyx RE2 and Maximum Impact systems [S1, unofficial].
*   **Development:** The authors note that to get a better image, they are experimenting with putting multiple concentric cylinders inside the volume, which should give more accurate results as distortion decreases [S1, unofficial].

## Sources
[S1] DTIC ADA325753: A Rendering System for Photo-Realistic Images and Virtual Environments: Final Report., https://archive.org/details/DTIC_ADA325753, fetched 2026-10-26 (unofficial)
