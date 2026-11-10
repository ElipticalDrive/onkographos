# 1D Lookup Table with Interpolation

Also called: HU -> mu mapping, Piecewise Linear HU to Attenuation Transfer Function, Piecewise Linear Transfer Function

## Overview
This method maps scalar voxel values (typically Hounsfield Units in medical imaging) to optical properties—specifically attenuation coefficients, color, and opacity—using a piecewise linear function defined by user-specified keypoints [S1, unofficial]. It solves the problem of visual clutter by allowing users to isolate regions of interest (e.g., bone, soft tissue) while rendering background data (e.g., air) transparent [S1, unofficial]. Unlike multi-dimensional transfer functions that incorporate gradient magnitude or other derivatives to highlight boundaries [S2, unofficial][S5, unofficial], the 1D lookup table relies solely on the scalar density/value, making it the simplest classification method but prone to ambiguity when different materials share similar scalar intensities [S8, unofficial][S12]. It builds on [Classification and transfer functions](38-classification-and-transfer-functions.md) by providing a computationally efficient, pre-computable mapping suitable for GPU texture sampling or shader evaluation.

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset $V$ where each voxel holds a scalar value $v$ (e.g., HU). A set of $K$ keypoints $P = \{(x_i, y_i)\}_{i=0}^{K-1}$, where $x_i$ is the input domain value (e.g., HU) and $y_i$ is the output property value (e.g., attenuation $\mu$, opacity $\alpha$, or color component).
*   **Output:** A transformed volume $V'$ where each voxel value is mapped to the corresponding optical property. In the specific medical imaging context, the output is an attenuation map $\mu_{tf}$ [S1, unofficial].

**Data Structures:**
*   **Keypoints Tensor:** A 2D tensor of shape $(K, 2)$, where $K$ is the number of keypoints. Column 0 stores the input domain values ($x$), and Column 1 stores the output range values ($y$) [S1, unofficial].
*   **Lookup Table (LUT):** Alternatively, the piecewise linear function can be pre-computed into a 1D texture or array of size $N$ (where $N$ is the bit-depth of the input, e.g., 4096 for 12-bit data), storing the mapped RGBA values for direct indexing [S3, unofficial][S6, unofficial].

**Steps:**
1.  **Define Keypoints:** Specify $K$ keypoints representing transitions between materials. For medical imaging, typical keypoints include air, lung/soft-tissue, soft tissue, cortical bone, and dense bone/metal [S1, unofficial].
2.  **Normalize Input Domain:** Map the input domain values to a normalized range $[0, 1]$ to facilitate interpolation. For Hounsfield Units (HU), the normalization formula is:
    $$ x_p = \frac{v + 3500}{7000} $$
    where $v$ is the raw HU value, offset is 3500, and scale is 7000 [S1, unofficial]. This maps the range $[-3500, 3500]$ to $[0, 1]$. Note: The source example uses a max HU of 3071, but the normalization formula uses 3500 as the upper bound offset/scale reference [S1, unofficial].
3.  **Extract Normalized Keypoints:** Create 1D tensors $x_p$ and $y_p$ of shape $(1, K)$ containing the normalized input values and the corresponding output values (attenuation, opacity, or color) for each keypoint [S1, unofficial].
4.  **Interpolate:** Apply piecewise linear interpolation to the volume. For each voxel value $v$:
    *   Normalize $v$ to $x_p(v)$.
    *   Find the interval $[x_{p,i}, x_{p,i+1}]$ such that $x_{p,i} \le x_p(v) \le x_{p,i+1}$.
    *   Compute the interpolated value $y$ using linear interpolation:
        $$ y = y_i + \frac{(y_{i+1} - y_i)(x_p(v) - x_{p,i})}{x_{p,i+1} - x_{p,i}} $$
    *   Assign $y$ to the output volume at the corresponding voxel location [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def piecewise_linear_hu_to_mu(vol: torch.Tensor, keypoints: torch.Tensor) -> torch.Tensor:
    """
    Maps Hounsfield Unit (HU) values in a volume to attenuation coefficients (mu)
    using a piecewise linear function defined by keypoints.
    
    Args:
        vol: 3D tensor of shape (D, H, W) containing HU values.
        keypoints: 2D tensor of shape (K, 2). 
                   Column 0: HU values (x).
                   Column 1: Attenuation values (y).
    
    Returns:
        mu_tf: 3D tensor of shape (D, H, W) containing mapped attenuation values.
    """
    # 1. Normalize HU values to [0, 1]
    # Formula: (HU + 3500) / 7000
    # Clamp to [0, 1] to handle out-of-range values if necessary
    vol_normalized = torch.clamp((vol + 3500.0) / 7000.0, 0.0, 1.0)
    
    # 2. Extract normalized input (xp) and output (yp) values from keypoints
    # keypoints shape: (K, 2)
    xp = keypoints[:, 0]
    yp = keypoints[:, 1]
    
    # Normalize xp to [0, 1] as well for consistent interpolation
    # Assuming xp is already in HU range, we apply the same normalization
    # However, the source implies xp is the normalized version of the keypoint HU values
    # If keypoints are given in raw HU, normalize them:
    xp_normalized = torch.clamp((xp + 3500.0) / 7000.0, 0.0, 1.0)
    
    # 3. Perform piecewise linear interpolation
    # Using torch.nn.functional.interpolate or manual implementation
    # Here we use a manual approach for clarity and dependency-free implementation
    
    D, H, W = vol.shape
    mu_tf = torch.zeros_like(vol)
    
    # Reshape for vectorized operation
    vol_flat = vol_normalized.view(-1)
    xp_flat = xp_normalized.view(-1)
    yp_flat = yp.view(-1) # yp is already in the target range (e.g., mu)
    
    # Find indices where vol_flat falls between keypoints
    # Add boundary conditions: values below min keypoint use min keypoint's y
    # values above max keypoint use max keypoint's y
    
    # Create extended keypoints for boundary handling
    # If vol_flat < xp_normalized[0], use yp[0]
    # If vol_flat > xp_normalized[-1], use yp[-1]
    
    # Mask for values within range
    mask_low = vol_flat < xp_normalized[0]
    mask_high = vol_flat > xp_normalized[-1]
    mask_mid = ~mask_low & ~mask_high
    
    # Initialize output
    mu_flat = torch.zeros_like(vol_flat)
    
    # Handle low boundary
    mu_flat[mask_low] = yp_flat[0]
    
    # Handle high boundary
    mu_flat[mask_high] = yp_flat[-1]
    
    # Handle mid range
    if mask_mid.any():
        # Find the index i such that xp_normalized[i] <= vol_flat <= xp_normalized[i+1]
        # searchsorted returns indices where vol_flat could be inserted while maintaining order
        # We use 'right' side insertion to handle exact matches consistently
        indices = torch.searchsorted(xp_normalized, vol_flat[mask_mid], side='right') - 1
        
        # Clamp indices to valid range [0, K-2]
        indices = torch.clamp(indices, 0, len(xp_normalized) - 2)
        
        x0 = xp_normalized[indices]
        x1 = xp_normalized[indices + 1]
        y0 = yp_flat[indices]
        y1 = yp_flat[indices + 1]
        
        x = vol_flat[mask_mid]
        
        # Linear interpolation formula
        # y = y0 + (y1 - y0) * (x - x0) / (x1 - x0)
        # Avoid division by zero if x0 == x1
        denom = x1 - x0
        denom = torch.where(denom == 0, 1.0, denom)
        
        mu_flat[mask_mid] = y0 + (y1 - y0) * (x - x0) / denom
        
    # Reshape back to 3D
    mu_tf = mu_flat.view(D, H, W)
    
    return mu_tf
```

**Parameters:**
*   **Keypoints ($K$):** The number of control points defining the transfer function. Typical values depend on the number of distinct materials to be visualized (e.g., 5 for air, lung, soft tissue, bone, metal) [S1, unofficial].
*   **Normalization Offset/Scale:** For HU data, offset = 3500, scale = 7000 [S1, unofficial].
*   **Max Opacity:** In general 1D TFs, a maximum opacity value $\alpha_{max} \in [0, 1]$ can be applied to scale the output opacity [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Out-of-Bounds Values:** Voxel values outside the range of the defined keypoints must be handled. The standard approach is clamping to the nearest endpoint value [S1, unofficial].
*   **Identical RGB Triplets:** If used for color mapping, using identical RGB triplets for different target values causes surfaces to appear with the same color, leading to ambiguity [S2, unofficial].
*   **Color Crossover:** RGB triplets must be distinct enough to prevent color crossover caused by surface highlighting variations [S2, unofficial].
*   **Label Band-Crossing:** If labels are present, linear interpolation across label boundaries produces values that fall into incorrect anatomical classes. In such cases, nearest-neighbor interpolation should be used instead [S3, unofficial].

## Subtopics

### [Transfer Function](1272-transfer-function.md)
A Transfer Function (TF) maps raw scalar voxel values from a 3D volume dataset to optical properties—specifically color (RGB) and opacity ($\alpha$)—to classify and visualize regions of interest. It solves the problem of visual clutter by allowing users to assign high opacity to desired structures (e.g., bone, vessels) and low opacity or transparency to irrelevant background data (e.g., air, soft tissue). Unlike multi-dimensional transfer functions that incorporate gradient magnitude or other derivatives to highlight boundaries, the 1D TF relies solely on the scalar density/value, making it the simplest classification method but prone to ambiguity when different materials share similar scalar intensities.

### [Cubic Spline Transfer Function](1811-cubic-spline-transfer-function.md)
A 1D transfer function that maps a scalar field value in $[0, 1]$ to density and attenuation color using cubic spline interpolation of control points. It builds on [1D Lookup Table with Interpolation](2528-1d-lookup-table-with-interpolation.md) by providing $C^1$ continuous derivatives for smoother visual transitions compared to linear methods.

### [Linear Look-up Table Classification](2636-linear-look-up-table-classification.md)
Linear look-up table classification maps voxel intensity values to material composition percentages using a pre-calculated linear table. It solves the problem of assigning physical properties to discrete voxel data by treating classification as a nearest-neighbor decision based on density peaks. Unlike [Cubic Spline Transfer Function](1811-cubic-spline-transfer-function.md) or [Piecewise Linear Interpolation (Lookup Table)](2643-piecewise-linear-interpolation-lookup-table.md) which allow arbitrary opacity curves, this method enforces a strict linear ordering of materials, making it computationally simple but restricted to scenarios where materials are distinct and adjacent in density space.

### [Piecewise Linear Interpolation (Lookup Table)](2643-piecewise-linear-interpolation-lookup-table.md)
Piecewise linear interpolation maps scalar volume data values to visual parameters (color and opacity) by constructing a continuous curve through a set of user-defined control points. It solves the problem of defining a transfer function (TF) that is intuitive to edit and computationally efficient to evaluate during ray casting. It differs from [Linear Look-up Table Classification](2636-linear-look-up-table-classification.md) by explicitly computing intermediate values via interpolation rather than assigning discrete classes, and from [Smoothstep Interpolation](3287-smoothstep-interpolation.md) by lacking derivative continuity at the knots.

### [Smoothstep Interpolation](3287-smoothstep-interpolation.md)
Smoothstep interpolation provides a smooth, continuous transition between two values by applying a cubic Hermite spline to a normalized input. It is used in volume rendering classification to map scalar values to transfer function properties (such as opacity or color) with $C^1$ continuity, avoiding the discontinuities inherent in linear interpolation. It differs from [Linear Look-up Table Classification](2636-linear-look-up-table-classification.md) by eliminating sharp corners in the transfer function, and from [Cubic Spline Transfer Function](1811-cubic-spline-transfer-function.md) by being a fixed, parameter-free kernel rather than a piecewise function defined by control points.

### [VTK-Based Transfer Function Application](3725-vtk-based-transfer-function-application.md)
This method applies color and opacity transfer functions to scalar volume data using VTK's `vtkLookupTable` for color mapping and `vtkPiecewiseFunction` for opacity. It provides a pre-computed, table-based approach that decouples the transfer function definition from the rendering pipeline, allowing for efficient retrieval of RGBA components during ray casting. It differs from direct mathematical evaluation methods by relying on VTK's internal interpolation and caching mechanisms, and from surface rendering techniques by operating on volumetric scalar data rather than generating polygonal meshes.

### [Lookup Table Implementation](5347-lookup-table-implementation.md)
Lookup Table Implementation refers to the family of procedures that prepare, validate, and upload transfer function data to the GPU for volume rendering. It solves the problem of efficiently mapping scalar voxel values to RGBA properties by ensuring the data is in a format suitable for hardware acceleration (textures or SSBOs) and shader access. It is distinguished from its siblings by focusing on the **infrastructure** of the transfer function (memory allocation, monotonicity checks, binding) rather than the **definition** of the function itself (e.g., [1D Transfer Function Generation and Sampling](2689-1d-transfer-function-generation-and-sampling.md)) or its **application** during ray marching (e.g., [Hardcoded Transfer Function](1098-hardcoded-transfer-function.md)).

### [2D Transfer Function with Gradient Magnitude](5491-2d-transfer-function-with-gradient-magnitude.md)
This method extends the 1D lookup table by adding gradient magnitude as a second dimension, mapping a voxel's scalar value and its local gradient magnitude simultaneously to color and opacity. It solves the ambiguity of 1D transfer functions where different materials share similar intensity values but differ in boundary sharpness, allowing for finer control and sophisticated visualization of material boundaries. Unlike independent 1D functions that map scalar-to-color and scalar/gradient-to-opacity separately, this approach uses a single 2D domain to disambiguate regions that fall within similar ranges for both parameters simultaneously.

### [3D Transfer Function](5777-3d-transfer-function.md)
A 3D Transfer Function maps volume rendering samples to visual properties using three input dimensions, typically scalar data value, gradient magnitude, and a tertiary attribute such as the second directional derivative, occlusion, or statistical measures. It solves the problem of disambiguating complex boundary configurations where material boundaries overlap in two dimensions of the transfer function domain but differ in the third. Unlike 1D transfer functions which map only scalar value, or 2D transfer functions which map value and gradient magnitude, 3D TFs provide higher specificity for material boundaries, allowing the isolation of features that are indistinguishable in lower-dimensional projections.

### [Generic Operators](10489-generic-operators.md)
This family defines a set of primitive image processing and geometric operators used to transform voxel data (density, gradient, or binary masks) before or during the classification and rendering pipeline. Unlike the parent [1D Lookup Table with Interpolation](2528-1d-lookup-table-with-interpolation.md) which maps scalar values to color/opacity, these operators modify the spatial structure or intensity values of the volume itself to enhance surface detection, reduce artifacts, or prepare data for specific rendering techniques. They are distinguished from siblings by operating on the scene geometry or raw voxel intensities rather than defining the final visual appearance mapping.

### Target Value Configuration
User configuration of up to five target data values, their associated RGB colors, allowable value deviation windows, and maximum opacity [S2, unofficial]. The number of targets is limited to five [S2, unofficial]. RGB triplets are expressed as three real numbers ranging from 0.0 to 1.0 and must be distinct for each target to avoid color ambiguity and crossover [S2, unofficial]. Using identical RGB triplets for different target values causes surfaces to appear with the same color, leading to ambiguity [S2, unofficial]. RGB triplets must be distinct enough to prevent color crossover caused by surface highlighting variations [S2, unofficial].

### Label-aware Interpolation Switching
Switches volume interpolation type to nearest-neighbor when labels are present to prevent band-crossing artifacts, otherwise uses linear interpolation [S3, unofficial]. The steps are: 1. Check if labels are present via `hasLabels()`. 2. If labels exist, call `volume.getProperty().setInterpolationTypeToNearest()`. 3. Otherwise, call `volume.getProperty().setInterpolationTypeToLinear()` [S3, unofficial]. Linear interpolation across label band boundaries produces values that fall into incorrect anatomical classes, breaking label-scoped transfer functions [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels, assuming $K$ (number of keypoints) is small and constant. Interpolation per voxel is $O(1)$ if using a pre-computed LUT or binary search over keypoints.
*   **Memory Complexity:** $O(K)$ for storing keypoints or $O(M)$ for storing a pre-computed LUT of size $M$ (e.g., 4096 entries) [S3, unofficial][S6, unofficial].
*   **Performance:** Fast and suitable for real-time rendering on GPUs due to simple arithmetic operations and potential for texture-based lookup [S3, unofficial][S6, unofficial].
*   **Strengths:** Simple to implement and edit; computationally efficient; intuitive for users to define material ranges [S1, unofficial][S5, unofficial][S6, unofficial].
*   **Limitations:** Prone to ambiguity when different materials share similar scalar intensities; cannot distinguish materials based on boundary characteristics or other derived features [S8, unofficial][S12].
*   **When to Use:** When the volume data has distinct scalar ranges for different materials and high-frequency boundary details are not critical for classification.
*   **When Not to Use:** When materials have overlapping intensity ranges but different boundary sharpness (use 2D TF with gradient magnitude) or when smooth, derivative-continuous transitions are required (use Cubic Spline).

## Background
The method originates from early volume rendering techniques where simple piecewise linear functions were used to map scalar values to optical properties [S1, unofficial][S2, unofficial]. It is widely used in medical imaging (e.g., CT scans) to map Hounsfield Units to attenuation coefficients or opacity [S1, unofficial]. The approach is foundational in visualization toolkits like VTK, which provide built-in support for piecewise linear transfer functions [S1, unofficial].

## Sources
[S1] pcarnah/flashdrr, https://github.com/pcarnah/flashdrr, fetched 2026-10-26 (unofficial)
[S2] DTIC ADA202661: Volumetric Rendering Techniques for the Display of Three-Dimensional Aerodynamic Flow Field Data, https://archive.org/details/DTIC_ADA202661, fetched 2026-10-26 (unofficial)
[S3] kevinkunkel98/reinforcement-learning-volume-rendering-transfer-function, https://github.com/kevinkunkel98/reinforcement-learning-volume-rendering-transfer-function, fetched 2026-10-27 (unofficial)
