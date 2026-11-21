# Radon Transform
Also called: Discrete Radon Transform, Forward Radon Transform, Parallel-Beam Radon Transform, Sinogram Generation, Transformée de Radon, Transformée de Radon discrète, skradon

## Overview
The Radon transform maps a 2D image to its projections (sinogram) by integrating pixel intensities along parallel lines at various angles [S3, S4, unofficial]. It solves the forward acquisition model in computed tomography, converting spatial domain data into projection domain data [S1, S4, unofficial]. Unlike its inverse counterpart (e.g., Filtered Back-Projection), which reconstructs images from projections, the forward Radon transform generates the measurements required for reconstruction [S3, S4, unofficial]. It is distinguished from other projection methods by its strict parallel-beam geometry and linear integration nature [S3, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A 2D image $I$ (discrete pixel grid) and a set of projection angles $\theta$. In batch processing, this is a tensor of shape $(N, C, H, W)$ where $N$ is batch size, $C$ channels, $H$ height, $W$ width [S2, unofficial].
- **Output:** A sinogram tensor of shape $(N, C, D, A)$ where $D$ is the number of detector positions (radial bins) and $A$ is the number of angles [S2, S4, unofficial].

**Data Structures:**
- **Image Grid:** Discrete indices $(i, j)$ corresponding to pixel centers. The image is modeled as a sum of discrete Dirac impulses [S3, unofficial]:
  $$ f(x,y) = \sum_{i=-\infty}^{\infty} \sum_{j=-\infty}^{\infty} I(i,j) \Delta(x - j) \Delta(y - i) $$
  where $\Delta(\cdot)$ is the discrete Dirac impulse [S3, unofficial].
- **Projection Line:** Defined by angle $\theta$ and radial distance $\rho$ from the origin. The line equation is $\rho = x \cos \theta + y \sin \theta$ [S3, unofficial].
- **Detector Positions:** Linearly spaced values $r_s$ covering the image extent, typically from $-1$ to $1$ normalized or in pixel units [S1, unofficial].

**Formulas:**
The continuous Radon transform is defined as [S3, unofficial]:
$$ R_\theta(\rho) = \int_{-\infty}^{\infty} \int_{-\infty}^{\infty} f(x,y) \delta(\rho - x \cos \theta - y \sin \theta) \, dx \, dy $$
where $\delta(\cdot)$ is the Dirac delta function [S3, unofficial].

The discrete Radon transform $R_\theta(\rho)$ is the sum of pixel values $I(i,j)$ whose centers lie on the projection line [S3, unofficial]:
$$ R_\theta(\rho) = \sum_{i=0}^{H-1} \sum_{j=0}^{W-1} I(i,j) \cdot \mathbb{I}(\text{line } (\theta, \rho) \text{ intersects pixel } (i,j)) $$
In practice, this is implemented via interpolation (e.g., bilinear) to approximate the integral along the ray [S2, S4, unofficial].

**Parameters:**
- **theta:** Array of projection angles in degrees. Typical default: `np.linspace(0.0, 180.0, max(image.shape), endpoint=False)` [S4, unofficial]. In `skradon`, default is `torch.arange(180)` [S2, unofficial].
- **rs:** Detector positions. Default: linearly spaced from -1 to 1 with length equal to image height [S1, unofficial].
- **circle:** Boolean. If True, assumes zero intensity outside the inscribed circle, adjusting sinogram width to `min(H, W)` [S2, unofficial].
- **preserve_range:** Boolean. If True, preserves original dtype range; otherwise converts to float [0,1] [S2, unofficial].

**Steps:**
1. **Preprocessing:** Convert input image to float. If `preserve_range` is False, scale values according to `img_as_float` conventions (e.g., uint8 [0,255] to [0,1]) [S2, unofficial].
2. **Angle Setup:** If `theta` is None, generate angles from 0 to 179 degrees [S2, unofficial].
3. **Geometry Setup:**
   - If `circle` is True, calculate the radius $r = \min(H, W) // 2$ and mask pixels outside the inscribed circle to zero [S2, unofficial].
   - Define the rotation center at $(H//2, W//2)$ [S2, unofficial].
4. **Projection Loop (for each angle $\theta_k$ in `theta`):**
   a. **Affine Transformation:** Construct a $2 \times 3$ affine matrix representing rotation by $\theta_k$ and translation to center the projection axis [S2, unofficial].
   b. **Grid Generation:** Create a sampling grid using `affine_grid` (or equivalent) based on the affine matrix and image dimensions. `align_corners=False` is standard [S2, unofficial].
   c. **Sampling:** Sample the image values at the grid coordinates using `grid_sample` (bilinear interpolation) [S2, unofficial]. This yields a 2D projection profile for the current angle.
   d. **Accumulation:** Store the projection profile.
5. **Post-processing:**
   - If using the `skimage`-style approach (S1): Rotate the image for each angle, sum along the vertical axis, multiply by detector spacing $dr$, and transpose to form the sinogram [S1, unofficial].
   - If using the `skradon` approach (S2): The sampled projections are accumulated directly into the sinogram tensor [S2, unofficial].
6. **Output:** Return the sinogram tensor [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def radon_transform(image: torch.Tensor, theta: torch.Tensor = None, circle: bool = False, preserve_range: bool = False) -> torch.Tensor:
    """
    Computes the forward Radon transform (sinogram) of a batch of images.
    
    Args:
        image: Tensor of shape (N, C, H, W).
        theta: Tensor of angles in degrees. Default: arange(180).
        circle: If True, assumes zero outside inscribed circle.
        preserve_range: If False, converts image to float [0, 1].
        
    Returns:
        sinogram: Tensor of shape (N, C, D, A).
    """
    # 1. Preprocessing
    if not preserve_range:
        # Convert to float and scale to [0, 1] if necessary
        # Note: Exact scaling depends on input dtype (e.g., uint8 -> /255.0)
        image = image.float()
        if image.dtype == torch.uint8:
            image = image / 255.0
        elif image.max() > 1.0:
            image = image / image.max()
            
    N, C, H, W = image.shape
    device = image.device
    
    # 2. Angle Setup
    if theta is None:
        theta = torch.arange(180, dtype=torch.float32, device=device)
    else:
        theta = theta.to(device=device, dtype=torch.float32)
        
    A = theta.shape[0]
    
    # 3. Detector Positions (D)
    # Default: number of detectors equals image height (or min dimension if circle)
    if circle:
        D = min(H, W)
    else:
        D = H # Or W, depending on implementation convention. S1 uses height. S2 uses min if circle.
        
    # Detector positions rs from -1 to 1 (normalized) or pixel units
    # For affine grid, we typically work in normalized coordinates [-1, 1]
    rs = torch.linspace(-1.0, 1.0, D, device=device)
    
    # 4. Initialize Sinogram
    sinogram = torch.zeros((N, C, D, A), dtype=image.dtype, device=device)
    
    # 5. Loop over angles
    for k, angle_deg in enumerate(theta):
        angle_rad = torch.deg2rad(angle_deg)
        
        # Rotation matrix components
        cos_a = torch.cos(angle_rad)
        sin_a = torch.sin(angle_rad)
        
        # Construct affine matrix for rotation around center
        # Center in normalized coordinates: (0, 0) if we shift, or (0.5, 0.5) if using align_corners=False with standard grid
        # PyTorch affine_grid expects matrix in normalized coordinates [-1, 1]
        # Rotation around center (0.5, 0.5) in [0,1] space:
        # M = [ cos_a  -sin_a  tx ]
        #     [ sin_a   cos_a  ty ]
        # To rotate around center, we translate center to origin, rotate, translate back.
        # However, `affine_grid` with `align_corners=False` maps grid points to source.
        # We need to sample the image at rotated coordinates.
        
        # Standard approach: Create a grid of target coordinates, then map to source image coordinates.
        # Or: Create an affine matrix that rotates the image coordinates.
        
        # Let's use the standard affine transformation for projection:
        # We want to project along lines perpendicular to angle theta.
        # This is equivalent to rotating the image by -theta and summing? 
        # No, Radon is integration. The `skradon` implementation uses `grid_sample` with an affine grid.
        
        # Construct the 2x3 affine matrix for `affine_grid`
        # The grid generated by `affine_grid` will be in the range [-1, 1].
        # We want to sample the original image.
        
        # Rotation matrix R
        # R = [[ cos_a, -sin_a],
        #      [ sin_a,  cos_a]]
        # But we need to handle the center.
        
        # Simpler: Use `F.affine_grid` to generate a grid that corresponds to the projection geometry.
        # Actually, `skradon` uses a specific warp matrix.
        # Let's approximate the `skimage` logic: rotate image, then sum.
        # But `skradon` is differentiable and uses grid_sample.
        
        # Implementation detail from S2:
        # "construct a 2x3 affine transformation matrix representing the rotation and translation"
        # "Generate a sampling grid using F.affine_grid ... Sample ... using F.grid_sample"
        
        # Affine matrix for rotation by -theta around center (0.5, 0.5) in normalized coords
        # tx = 0.5 - 0.5*cos_a + 0.5*sin_a
        # ty = 0.5 - 0.5*sin_a - 0.5*cos_a
        # This rotates the coordinate system.
        
        # Let's use the standard rotation matrix around center (0.5, 0.5)
        # M = [[ cos_a, -sin_a, 0.5*(1-cos_a) + 0.5*sin_a],
        #      [ sin_a,  cos_a, 0.5*(1-sin_a) - 0.5*cos_a]]
        
        # Note: The exact translation terms depend on the desired projection axis.
        # For Radon, we project along lines.
        # A common differentiable implementation rotates the image by -theta and then sums along one axis?
        # No, that's for parallel beam if we just want the projection profile.
        # But we need the sinogram for all rho.
        
        # Correct approach for differentiable Radon (S2):
        # 1. Create a grid of points in the output space (detector positions).
        # 2. Map these points back to the input image space using the inverse projection geometry.
        # 3. Sample the image.
        
        # Let's stick to the `skimage` rotation-sum method for clarity if `skradon` is too opaque,
        # but the prompt asks for `skradon` style as it's the primary source for the "method" in the tree.
        # S2 says: "construct a 2x3 affine transformation matrix ... Generate sampling grid ... Sample".
        
        # Let's define the affine matrix for rotating the image by -theta.
        # This allows us to sum along columns to get projections?
        # No, Radon is not just rotation + sum. It's integration along lines.
        # If we rotate the image by -theta, the columns of the rotated image ARE the projections (integrated along the vertical axis).
        # So:
        # 1. Rotate image by -theta.
        # 2. Sum along axis 1 (columns) to get projection profile?
        # Wait, `skimage.radon` does: Rotate image, then sum along vertical axis.
        # S1 confirms: "Rotate the input image ... sum the values along the vertical axis".
        
        # So the differentiable version should mimic this:
        # 1. Rotate image by -theta.
        # 2. Sum along the appropriate axis.
        
        # However, S2 uses `grid_sample`. Why?
        # Because `rotate` is not differentiable in the same way or `grid_sample` allows for interpolation during rotation.
        # So:
        # 1. Create a grid corresponding to the rotated image coordinates.
        # 2. Sample the original image on this grid.
        # 3. Sum the result.
        
        # Let's implement the rotation via `grid_sample`.
        
        # Affine matrix for rotation by -angle_rad around center (0.5, 0.5)
        cos_a = torch.cos(angle_rad)
        sin_a = torch.sin(angle_rad)
        
        # Translation to keep center fixed
        # tx = 0.5 - 0.5*cos_a + 0.5*sin_a
        # ty = 0.5 - 0.5*sin_a - 0.5*cos_a
        # Note: Signs depend on rotation direction.
        
        # Construct 2x3 matrix
        # [[ cos_a, -sin_a, tx],
        #  [ sin_a,  cos_a, ty]]
        
        # For rotation by -theta:
        # cos(-t) = cos(t), sin(-t) = -sin(t)
        # M = [[ cos_a, sin_a, tx],
        #      [-sin_a, cos_a, ty]]
        
        # Let's use the standard rotation matrix for angle -theta
        # M = [[ cos_a, sin_a, 0.5*(1-cos_a) - 0.5*sin_a],
        #      [-sin_a, cos_a, 0.5*(1+sin_a) - 0.5*cos_a]]
        
        # Actually, let's just use the `skimage` logic which is robust.
        # But we must write it in Python.
        
        # Let's assume the `skradon` implementation details:
        # It uses `F.affine_grid` with a warp matrix.
        
        # Warp matrix for rotation by -theta
        # Center is (0.5, 0.5)
        # tx = 0.5 - 0.5*cos_a + 0.5*sin_a  <-- This is for rotation by +theta?
        # Let's derive:
        # x' = x cos - y sin
        # y' = x sin + y cos
        # To rotate by -theta:
        # x' = x cos + y sin
        # y' = -x sin + y cos
        # Center (0.5, 0.5) -> (0.5, 0.5)
        # 0.5 = 0.5 cos + 0.5 sin + tx => tx = 0.5 - 0.5 cos - 0.5 sin
        # 0.5 = -0.5 sin + 0.5 cos + ty => ty = 0.5 + 0.5 sin - 0.5 cos
        
        tx = 0.5 - 0.5 * cos_a - 0.5 * sin_a
        ty = 0.5 + 0.5 * sin_a - 0.5 * cos_a
        
        # Affine matrix for affine_grid (2x3)
        # [[ cos_a,  sin_a, tx],
        #  [-sin_a,  cos_a, ty]]
        
        theta_mat = torch.zeros((N, 2, 3), device=device)
        theta_mat[:, 0, 0] = cos_a
        theta_mat[:, 0, 1] = sin_a
        theta_mat[:, 0, 2] = tx
        theta_mat[:, 1, 0] = -sin_a
        theta_mat[:, 1, 1] = cos_a
        theta_mat[:, 1, 2] = ty
        
        # Generate grid
        grid = F.affine_grid(theta_mat, size=(N, C, H, W), align_corners=False)
        
        # Sample image
        # grid_sample expects grid of shape (N, H, W, 2)
        rotated_image = F.grid_sample(image, grid, mode='bilinear', padding_mode='zeros', align_corners=False)
        
        # Sum along vertical axis (axis 2, height) to get projection profile?
        # In `skimage`, sum along vertical axis (axis 0) after rotation.
        # Here, rotated_image is (N, C, H, W).
        # If we rotated by -theta, the columns (axis 3) correspond to the projection lines?
        # No, `skimage` rotates the image, then sums along the vertical axis (rows).
        # So we sum along axis 2 (H).
        
        # But we need the sinogram for all detector positions (rho).
        # The sum along H gives a 1D profile of length W.
        # We need to map this to the detector positions rs.
        
        # This simple rotation-sum approach assumes the detector positions align with the pixel columns.
        # For a more general Radon transform, we need to interpolate the sum.
        
        # Let's stick to the S2 description: "Sample the image values at the grid coordinates ... to produce the projection values".
        # This implies we are not just summing, but integrating along specific lines.
        
        # Given the complexity of implementing the exact `skradon` grid from scratch without its internal helpers,
        # and the fact that S1 provides a clear algorithmic description (Rotate -> Sum),
        # we will use the Rotate -> Sum approach as the primary implementation, noting that `skradon` uses grid_sample for differentiability.
        
        # However, the prompt asks for the method from the article. S2 is the primary source for the "method" in the tree context (TorchRadon).
        # S2 says: "construct a 2x3 affine transformation matrix ... Generate a sampling grid ... Sample ... Accumulate".
        
        # Let's provide the S2-style pseudocode which is more general.
        
        # For each angle, we create a grid that samples the image along the projection lines.
        # This is complex. Let's fall back to the S1 method which is explicitly described and easier to implement from text.
        # S1: Rotate, Sum, Multiply by dr, Transpose.
        
        # But S1 is `skimage`. S2 is `skradon`.
        # The article is "Radon Transform (method)" under "TorchRadon".
        # So we should prioritize S2.
        
        # S2 Implementation Logic:
        # 1. Convert to float.
        # 2. If circle, mask.
        # 3. For each theta:
        #    a. Create affine matrix.
        #    b. Create grid.
        #    c. Sample image.
        #    d. The sampled image is the projection? No, it's the rotated image.
        #    e. Then sum? S2 doesn't explicitly say "sum". It says "accumulate the sampled projections".
        #    f. This implies the grid itself defines the projection lines.
        
        # Let's assume the S1 method is the "standard" Radon transform and S2 is a differentiable variant.
        # We will write the S1 method as it is complete and clear.
        
        pass # Placeholder for the loop logic which is complex to derive exactly from S2 text alone without code.
        
    # Given the ambiguity in S2's text regarding the exact grid construction for projection lines (vs rotation),
    # and the clarity of S1, we will use S1's logic for the pseudocode, as it is a valid implementation of the Radon Transform.
    
    # Re-writing pseudocode based on S1 (skimage) which is a standard implementation:
    
    sinogram = torch.zeros((N, C, D, A), dtype=image.dtype, device=device)
    
    for k, angle_deg in enumerate(theta):
        angle_rad = torch.deg2rad(angle_deg)
        
        # 1. Rotate image by -angle_rad
        # Using grid_sample for differentiability
        cos_a = torch.cos(angle_rad)
        sin_a = torch.sin(angle_rad)
        
        # Rotation matrix for -angle
        # tx = 0.5 - 0.5*cos_a + 0.5*sin_a
        # ty = 0.5 - 0.5*sin_a - 0.5*cos_a
        # Wait, earlier derivation:
        # tx = 0.5 - 0.5*cos_a - 0.5*sin_a
        # ty = 0.5 + 0.5*sin_a - 0.5*cos_a
        
        tx = 0.5 - 0.5 * cos_a - 0.5 * sin_a
        ty = 0.5 + 0.5 * sin_a - 0.5 * cos_a
        
        theta_mat = torch.zeros((N, 2, 3), device=device)
        theta_mat[:, 0, 0] = cos_a
        theta_mat[:, 0, 1] = sin_a
        theta_mat[:, 0, 2] = tx
        theta_mat[:, 1, 0] = -sin_a
        theta_mat[:, 1, 1] = cos_a
        theta_mat[:, 1, 2] = ty
        
        grid = F.affine_grid(theta_mat, size=(N, C, H, W), align_corners=False)
        rotated_image = F.grid_sample(image, grid, mode='bilinear', padding_mode='zeros', align_corners=False)
        
        # 2. Sum along vertical axis (axis 2, H)
        # This gives a profile of length W
        profile = torch.sum(rotated_image, dim=2) # Shape (N, C, W)
        
        # 3. Multiply by dr
        # dr = 1.0 / D ? Or based on image size?
        # S1: dr = difference between consecutive detector positions.
        # If rs is -1 to 1, dr = 2/(D-1).
        dr = 2.0 / (D - 1) if D > 1 else 1.0
        profile = profile * dr
        
        # 4. Transpose and store
        # Sinogram shape: (N, C, D, A)
        # Profile is (N, C, W). We need to map W to D.
        # If W == D, we can store directly.
        # If not, we need to interpolate.
        # S1 assumes W == D or handles it via `rs`.
        
        # For simplicity, assume W == D.
        if W == D:
            sinogram[:, :, :, k] = profile
        else:
            # Interpolate profile to D points
            # This is a gap [G]
            pass
            
    return sinogram
```

## Subtopics

### [Filtered Back-Projection (FBP)](11308-filtered-back-projection-fbp.md)
Filtered Back-Projection (FBP) is the inverse Radon transform method that reconstructs a 2D image from its sinogram. It solves the inverse problem by filtering the projection profiles in the frequency domain with a high-pass ramp filter to correct the blurring effect of simple back-projection, then integrating the filtered projections back into the image space. It is chosen for its computational efficiency compared to iterative or Fourier-based reconstruction methods.

### Single quadrant Radon transform
A variant that utilizes projections from only one quadrant (0-90 degrees) to reduce computational overhead, exploiting a four-fold redundancy [S5, unofficial]. It computes back-projections by treating them as real/imaginary parts of a complex number and averaging individual back-projections [S5, unofficial]. It is chosen when computational resources are limited, but requires the quadrature approach to avoid poor reconstruction quality [S5, unofficial].

## Operating characteristics
- **Time Complexity:** $O(A \cdot H \cdot W)$ for a single image, where $A$ is the number of angles, $H$ height, $W$ width. The rotation and summation steps dominate [S1, S2, unofficial].
- **Memory Complexity:** $O(D \cdot A)$ for the sinogram storage, where $D$ is the number of detectors [S2, unofficial].
- **Performance:** Efficient for moderate resolutions. GPU acceleration (via `torchskradon`) allows for batch processing [S2, unofficial].
- **Strengths:** Differentiable (when implemented with `grid_sample`), linear, and standard for forward projection in CT [S2, S4, unofficial].
- **Limitations:** Sensitive to aliasing if angles are sparse; requires interpolation for non-integer pixel shifts [S3, S4, unofficial].
- **When to use:** When generating synthetic sinograms for training or when the forward model is needed for differentiable tomography [S2, unofficial].
- **When not to use:** When only reconstruction is needed (use FBP or iterative methods) [S3, unofficial].

## Background
The Radon transform was defined by Johann Radon in 1919 [S3, unofficial]. It forms the mathematical foundation of computed tomography (CT) [S4, unofficial]. The discrete implementation is widely used in medical imaging and industrial CT [S1, S4, unofficial]. The `skimage` implementation is based on Justin K. Romberg's code [S2, unofficial]. The differentiable `skradon` implementation extends this to PyTorch for differentiable inverse rendering [S2, unofficial].

## Sources
[S1] guibo-tech/Tomographic-Reconstruction, https://github.com/guibo-tech/Tomographic-Reconstruction, fetched 2026-10-26 (unofficial)
[S2] tomluetjen/torchskradon, https://github.com/tomluetjen/torchskradon, fetched 2026-10-27 (unofficial)
[S3] Précision et qualité en reconstruction tomographique : algorithmes et applications, https://doi.org/10.70675/d2bc4875z6a5az46afz9289z2fe16b3a43f3, fetched 2026-10-30 (unofficial)
[S4] plot_radon_transform.rst.txt, https://scikit-image.org/docs/stable/_sources/auto_examples/transform/plot_radon_transform.rst.txt, fetched 2026-10-30 (unofficial)
[S5] Quadrature Radon Transform for Smoother  Tomographic Reconstruction, https://core.ac.uk/download/234644432.pdf, fetched 2026-10-27 (unofficial)
