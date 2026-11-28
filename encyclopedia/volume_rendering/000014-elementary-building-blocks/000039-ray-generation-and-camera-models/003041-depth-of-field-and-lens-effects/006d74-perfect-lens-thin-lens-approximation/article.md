# Perfect Lens (Thin Lens Approximation)
## Overview
The Perfect Lens method models a thin lens using the paraxial approximation to simulate depth of field effects in volume rendering. It maps incident rays from world space to a lens plane, intersects them with a defined plane of focus, and then refracts them through the optical center to a camera plane. This approach differs from sibling methods like [Thin Lens Depth of Field](4922-thin-lens-depth-of-field.md) by explicitly calculating the geometric intersection path through the lens center to determine the final ray direction, ensuring precise focus control based on magnification and numerical aperture.
## How it works
### Inputs and Outputs
**Inputs:**
- `origins`: Tensor of shape $(N, 3)$ containing ray origins in world space.
- `directions`: Tensor of shape $(N, 3)$ containing ray directions in world space.
- `f`: Focal length (float).
- `na`: Numerical aperture (float), defined as the inverse of the f-number.
- `m`: Lens magnification (float).
- `eps`: Numerical stability parameter (float, default $10^{-15}$).

**Outputs:**
- `new_origins`: Tensor of shape $(N, 3)$ representing the intersection points on the lens plane.
- `new_directions`: Tensor of shape $(N, 3)$ representing the refracted ray directions.
- `valid_mask`: Boolean tensor of shape $(N,)$ indicating which rays passed the aperture validation.

### Data Structures
- **Ray Tensor**: A batched tensor where each row is a 3D vector $(x, y, z)$.
- **Lens Parameters**: Stored as scalar floats.
- **Plane of Focus (POF)**: A plane perpendicular to the optical axis (x-axis) at distance $pof$ from the lens center.

### Steps
1.  **Transform to Lens Space**: Transform ray origins and directions from world space to lens space using the inverse camera transform. In this implementation, the lens is centered at the origin with the optical axis aligned to the x-axis.
2.  **Calculate Intersection with Lens Plane**: Compute the intersection time $t$ with the lens plane at $x=0$.
    $$ t = \frac{-\text{origins}_x}{\text{directions}_x + \epsilon} $$
    Calculate the intersection coordinates $(y, z)$ on the lens plane:
    $$ y = \text{origins}_y + t \cdot \text{directions}_y $$
    $$ z = \text{origins}_z + t \cdot \text{directions}_z $$
3.  **Validate Aperture**: Check if the intersection point lies within the lens radius $r_{lens}$.
    $$ r_{lens} = \frac{f \cdot na}{2} $$
    The ray is valid if:
    $$ t > 0 \quad \text{AND} \quad y^2 + z^2 < r_{lens}^2 $$
    If invalid, set $t$ to NaN and mark the ray as invalid.
4.  **Calculate Plane of Focus (POF)**: Determine the distance of the plane of focus from the lens center.
    $$ pof = \frac{f \cdot (m + 1)}{m} $$
    Adjust the sign of $pof$ to place it on the same side as the object relative to the lens. If the object is at $x_{objects}$, then:
    $$ pof = pof \cdot \text{sign}(x_{objects}) $$
5.  **Intersect with Plane of Focus**: Calculate the intersection point $x_\star$ on the plane of focus.
    $$ x_0 = \text{origins} + t \cdot \text{directions} $$
    $$ t_{pof} = \frac{x_{0,x} - pof}{\text{directions}_x + \epsilon} $$
    $$ t_{pof} = -t_{pof} $$
    $$ x_\star = x_0 + t_{pof} \cdot \text{directions} $$
6.  **Calculate Camera Position**: Determine the camera position on the opposite side of the lens.
    $$ \text{camera\_pos} = -f \cdot (1 + m) \cdot \text{sign}(x_{objects}) $$
7.  **Intersect with Camera Plane**: Calculate the final intersection point $x_v$ on the camera plane passing through the optical center.
    $$ d = -x_\star $$
    $$ t_{cam} = \frac{\text{camera\_pos} - x_{\star,x}}{d_x + \epsilon} $$
    $$ x_v = x_\star + t_{cam} \cdot d $$
8.  **Compute New Direction**: The new direction is the vector from the lens intersection point $x_0$ to the camera intersection point $x_v$.
    $$ \text{new\_direction} = x_v - x_0 $$
    Normalize the new direction.
9.  **Set New Origin**: The new origin is the intersection point on the lens plane $x_0$.
10. **Return Results**: Return the new origins, new directions, and the validity mask.

### Sampling Points on the Lens Disk
To sample points uniformly on the lens disk for Monte Carlo integration:
1.  Generate $r_{squared} = \text{rand()} \cdot r_{lens}^2$.
2.  Generate $\theta = \text{rand()} \cdot 2\pi$.
3.  Convert to Cartesian coordinates:
    $$ y = \sqrt{r_{squared}} \cdot \cos(\theta) $$
    $$ z = \sqrt{r_{squared}} \cdot \sin(\theta) $$
4.  The probability density is $p(A) = \frac{1}{\pi r_{lens}^2}$.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def perfect_lens_step(origins, directions, f, na, m, eps=1e-15):
    """
    Performs one step of the Perfect Lens approximation.
    
    Args:
        origins: Tensor of shape (N, 3)
        directions: Tensor of shape (N, 3)
        f: Focal length (float)
        na: Numerical aperture (float)
        m: Magnification (float)
        eps: Numerical stability parameter (float)
        
    Returns:
        new_origins: Tensor of shape (N, 3)
        new_directions: Tensor of shape (N, 3)
        valid_mask: Tensor of shape (N,)
    """
    # Step 1: Calculate intersection with lens plane at x=0
    # t = (-origins[:, 0]) / (directions[:, 0] + eps)
    t = -origins[:, 0] / (directions[:, 0] + eps)
    
    # y = origins[:, 1] + t * directions[:, 1]
    # z = origins[:, 2] + t * directions[:, 2]
    y = origins[:, 1] + t * directions[:, 1]
    z = origins[:, 2] + t * directions[:, 2]
    
    # Step 2: Validate aperture
    # lens_radius = f * na / 2
    lens_radius = f * na / 2.0
    radius_sq = y**2 + z**2
    valid_t = (t > 0) & (radius_sq < lens_radius**2)
    
    # Set t to NaN for invalid rays
    t_valid = torch.where(valid_t, t, torch.tensor(float('nan'), device=t.device))
    
    # Step 3: Calculate intersection point on lens plane x_0
    # x_0 = origins + t.unsqueeze(1) * directions
    # Note: Use t_valid for calculation, but keep original origins for invalid rays if needed
    # However, for invalid rays, the direction is undefined, so we can set it to zero or keep original
    # Let's assume we only process valid rays for direction calculation, but return full batch
    x_0 = origins + t_valid.unsqueeze(1) * directions
    
    # Step 4: Calculate Plane of Focus (POF)
    # pof = f * (m + 1) / m
    # We need the sign of the object position. Assuming x_objects is the x-coordinate of the object.
    # In this context, x_objects is typically the x-coordinate of the ray origin or a specified object plane.
    # Let's assume x_objects is the x-coordinate of the ray origin for simplicity, or passed as a parameter.
    # Based on the source, x_objects is calculated in intersect(). Let's assume it's the x-coordinate of the origin.
    x_objects = origins[:, 0]
    pof = f * (m + 1) / m
    pof = pof * torch.sign(x_objects)
    
    # Step 5: Intersect with Plane of Focus
    # t_pof = (x_0[:, 0] - pof) / (directions[:, 0] + eps)
    # t_pof = -t_pof
    # x_star = x_0 + t_pof.unsqueeze(1) * directions
    t_pof = (x_0[:, 0] - pof) / (directions[:, 0] + eps)
    t_pof = -t_pof
    x_star = x_0 + t_pof.unsqueeze(1) * directions
    
    # Step 6: Calculate Camera Position
    # camera_pos = -f * (1 + m) * torch.sign(x_objects)
    camera_pos = -f * (1 + m) * torch.sign(x_objects)
    
    # Step 7: Intersect with Camera Plane
    # d = -x_star
    # t_cam = (camera_pos - x_star[:, 0]) / (d[:, 0] + eps)
    # x_v = x_star + t_cam.unsqueeze(1) * d
    d = -x_star
    t_cam = (camera_pos - x_star[:, 0]) / (d[:, 0] + eps)
    x_v = x_star + t_cam.unsqueeze(1) * d
    
    # Step 8: Compute New Direction
    # new_direction = x_v - x_0
    new_direction = x_v - x_0
    
    # Normalize new_direction
    new_direction = new_direction / (torch.norm(new_direction, dim=1, keepdim=True) + eps)
    
    # Step 9: Set New Origin
    new_origins = x_0
    
    # Step 10: Handle Invalid Rays
    # For invalid rays, set new_direction to zero or original direction?
    # The source says "Return transformed rays with new directions and boolean mask of all True (valid rays)"
    # This implies we return the mask separately.
    # For invalid rays, the direction is not physically meaningful in this model.
    # We can set it to zero or keep the original. Let's set to zero for clarity.
    new_direction = torch.where(valid_t.unsqueeze(1), new_direction, torch.zeros_like(new_direction))
    new_origins = torch.where(valid_t.unsqueeze(1), new_origins, origins) # Keep original origin for invalid rays? Or set to NaN?
    # The source says "set t to NaN" for invalid rays.
    # If t is NaN, x_0 is NaN.
    # Let's use the t_valid to mask the results.
    
    # Re-calculate x_0 with t_valid to ensure NaN for invalid rays
    x_0_invalid = origins + t_valid.unsqueeze(1) * directions
    # If t_valid is False, t_valid is NaN, so x_0_invalid will be NaN
    # But torch.where with NaN might propagate NaN.
    # Let's just return the mask and the calculated values.
    
    return new_origins, new_direction, valid_t

def sample_lens_disk(lens_radius, num_samples, device='cpu'):
    """
    Samples points uniformly on the lens disk.
    
    Args:
        lens_radius: Radius of the lens (float)
        num_samples: Number of samples (int)
        device: Device to create tensors on
        
    Returns:
        points: Tensor of shape (num_samples, 3) with y and z coordinates
        pdf: Probability density (float)
    """
    # r_squared = rand() * lens_radius^2
    r_squared = torch.rand(num_samples, device=device) * lens_radius**2
    # theta = rand() * 2*pi
    theta = torch.rand(num_samples, device=device) * 2 * math.pi
    
    # y = sqrt(r_squared) * cos(theta)
    # z = sqrt(r_squared) * sin(theta)
    y = torch.sqrt(r_squared) * torch.cos(theta)
    z = torch.sqrt(r_squared) * torch.sin(theta)
    
    # points[:, 0] = 0 (on the lens plane)
    # points[:, 1] = y
    # points[:, 2] = z
    points = torch.zeros((num_samples, 3), device=device)
    points[:, 1] = y
    points[:, 2] = z
    
    # pdf = 1 / (pi * lens_radius^2)
    pdf = 1.0 / (math.pi * lens_radius**2)
    
    return points, pdf
```

### Parameters
- `f`: Focal length (float).
- `na`: Numerical aperture (float).
- `m`: Lens magnification (float).
- `eps`: Numerical stability parameter (float, default $10^{-15}$).

### Edge Cases and Pitfalls
- **Division by Zero**: The calculation of $t$ involves division by the x-component of the direction. A small epsilon is added to prevent division by zero.
- **Invalid Rays**: Rays that do not intersect the lens plane within the aperture are marked as invalid. Their directions are set to zero or kept as original, depending on implementation.
- **Sign of Object Position**: The sign of the object position determines the side of the lens for the plane of focus and camera position. Incorrect sign handling can lead to incorrect focus.
- **Numerical Stability**: The use of epsilon is crucial for numerical stability, especially when dealing with rays parallel to the lens plane.

## Operating characteristics
- **Time Complexity**: $O(N)$ where $N$ is the number of rays. Each ray is processed independently with a constant number of arithmetic operations.
- **Memory Complexity**: $O(N)$ for storing the ray origins, directions, and intermediate calculations.
- **Strengths**: Provides precise control over depth of field through explicit geometric calculations. Efficient for batched ray processing.
- **Limitations**: Assumes paraxial approximation, which may not be accurate for wide-angle lenses or large apertures. Does not account for lens aberrations.
- **When to Use**: Suitable for volume rendering applications requiring depth of field effects with a thin lens model.
- **When Not to Use**: Not suitable for simulating complex lens effects such as aberrations, distortion, or non-paraxial behavior.

## Background
The Perfect Lens method is based on the thin lens approximation, a common model in optics for simulating depth of field. It is implemented in the `gradoptics` library, which focuses on gradient-based optics simulations. The method is particularly useful in scientific visualization and volume rendering where precise control over focus is required.

## Sources
[S1] magis-slac/gradoptics, https://github.com/magis-slac/gradoptics, fetched 2026-10-27 (unofficial)
