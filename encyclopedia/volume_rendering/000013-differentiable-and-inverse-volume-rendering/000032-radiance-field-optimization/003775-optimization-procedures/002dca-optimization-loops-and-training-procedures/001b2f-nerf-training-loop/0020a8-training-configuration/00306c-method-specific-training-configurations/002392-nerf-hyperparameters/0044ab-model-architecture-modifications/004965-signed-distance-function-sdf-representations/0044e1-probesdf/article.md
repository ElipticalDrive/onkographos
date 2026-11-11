# ProbeSDF

Also called: Light Field Probes For Neural Surface Reconstruction

## Overview
ProbeSDF is a neural surface reconstruction method that decouples spatial and angular light field components using two volumetric grids of different resolutions to achieve real-time rendering and fast training [S1]. It solves the problem of view-dependent appearance modeling in Signed Distance Function (SDF) representations by positing that angular components (lighting/environment) are predominantly extrinsic and locally invariant to parallax, allowing them to be encoded at a lower spatial resolution than intrinsic spatial texture [S1]. Unlike Plenoxels or 3D Gaussian Splatting which use many parameters per voxel for color, ProbeSDF uses a lighter parametrization (as few as four coefficients per voxel) combined with spherical harmonics encoding of pre-MLP features [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Multi-view images, camera poses, and an initial sparse voxel grid structure.
*   **Outputs:** A reconstructed 3D surface (SDF field), surface normals, and rendered images with view-dependent appearance.

**Data Structures:**
1.  **SDF Grid:** Implicit representation of 3D shape. Stores SDF values and derived surface normal vectors $\mathbf{n}$ [S1].
2.  **Spatial Feature Grid:** High-resolution sparse voxel grid. Features are encoded via planar factorization within tiles [S1].
3.  **Angular Feature Grid (Probes):** Coarser grid storing light field probes represented as spherical harmonics coefficients [S1].
    *   **Tile Layout:** The space is divided into $16^3$ tiles.
    *   **Spatial Features ($F_s$):** Each tile contains $3 \times 16 \times 16 \times n_s$ coefficients, stored as three orthogonal planes ($F_x, F_y, F_z$) of resolution $16 \times 16$ [S1].
    *   **Angular Features ($F_a$):** Each tile has 8 corner probes. Each probe stores $l^2 \times n_a$ spherical harmonic coefficients [S1].
    *   **Memory Sharing:** Probes at shared corners of adjacent tiles are duplicated in memory but maintain shared values to benefit from hardware-accelerated interpolation [S1].

**Parameters:**
*   $n_s$: Number of spatial coefficients per probe dimension. Typical value: 4 or 8 [S1].
*   $n_a$: Number of angular features. Typical value: 4 or 8 [S1].
*   $l$: Spherical harmonic order. Up to 16 coefficients ($l=4$). Typical value: 4 [S1].
*   Per-camera bias vectors: Optional MLP bias vectors trained per camera to handle inconsistent shadows or large exposure changes (used for DTU and BMVS) [S1].

**Formulas:**
The outgoing radiance $C$ is computed by a small MLP that decodes spatial features, angular features, and grazing angle terms [S1]:

$$ C = MLP(\mathbf{F}_s(\mathbf{x}), \mathbf{F}_a(\mathbf{x}, \mathbf{r}), (1 - \mathbf{n} \cdot \mathbf{v})^0, \dots, (1 - \mathbf{n} \cdot \mathbf{v})^5, \dots) $$

Where:
*   $\mathbf{F}_s(\mathbf{x})$ is the spatial feature vector at position $\mathbf{x}$ [S1].
*   $\mathbf{F}_a(\mathbf{x}, \mathbf{r})$ is the angular feature vector at position $\mathbf{x}$ evaluated at the reflected vector $\mathbf{r}$ [S1].
*   $\mathbf{n}$ is the surface normal [S1].
*   $\mathbf{v}$ is the view direction [S1].
*   The terms $(1 - \mathbf{n} \cdot \mathbf{v})^k$ approximate the Fresnel term to account for increased surface reflectivity at grazing angles [S1].

**Spatial Feature Computation:**
Spatial features are computed via planar factorization to reduce dimensionality [S1]:

$$ \mathbf{F}_s = \mathbf{F}_x \otimes \mathbf{F}_y \otimes \mathbf{F}_z $$

Where $\mathbf{F}_x, \mathbf{F}_y, \mathbf{F}_z$ are the three orthogonal planes specific to the tile [S1].

**Angular Feature Computation:**
Angular features $\mathbf{F}_a$ are obtained by tri-linearly interpolating light field probes (spherical harmonics) from the coarser grid, evaluated at the reflected vector $\mathbf{r}$ [S1].

**Algorithm Steps:**
1.  **Initialize:** Create a sparse voxel grid for the SDF field and a coarser grid for angular probes. Initialize SDF values and probe coefficients [S1].
2.  **Ray Marching:** For each pixel, cast a ray and sample points along the ray.
3.  **SDF Evaluation:** Query the SDF grid at the sample point $\mathbf{x}$ to get the signed distance and derive the surface normal $\mathbf{n}$ [S1].
4.  **Feature Lookup:**
    *   Identify the $16^3$ tile containing $\mathbf{x}$.
    *   Compute $\mathbf{F}_s$ by interpolating the three orthogonal planes ($F_x, F_y, F_z$) within the tile [S1].
    *   Compute $\mathbf{F}_a$ by tri-linearly interpolating the 8 corner probes of the tile and evaluating the spherical harmonics at the reflected vector $\mathbf{r}$ [S1].
5.  **Appearance Decoding:**
    *   Calculate grazing angle terms: $g_k = (1 - \mathbf{n} \cdot \mathbf{v})^k$ for $k=0 \dots 5$ [S1].
    *   Concatenate $\mathbf{F}_s$, $\mathbf{F}_a$, and $g_k$ terms.
    *   Pass through the MLP to predict color $C$ [S1].
6.  **Rendering:** Integrate the color and opacity along the ray using volume rendering [S1].
7.  **Optimization:** Update SDF values, probe coefficients, and MLP weights using gradient descent to minimize the difference between rendered and ground truth images [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def evaluate_probe_sdf(x, v, sdf_grid, spatial_grid, probe_grid, l, ns, na):
    """
    Evaluates the ProbeSDF appearance model at point x for view direction v.
    
    Args:
        x: 3D position vector (float3)
        v: View direction vector (float3)
        sdf_grid: SDF grid object with query method
        spatial_grid: Spatial feature grid object with tile query method
        probe_grid: Angular probe grid object with tri-linear interpolation method
        l: Spherical harmonic order (e.g., 4)
        ns: Number of spatial coefficients per dimension
        na: Number of angular features per coefficient
        
    Returns:
        color: Predicted radiance (float3)
    """
    # 1. Get SDF and Normal
    sdf_val, normal = sdf_grid.query(x)
    # Ensure normal is normalized
    normal = normalize(normal)
    
    # 2. Compute Reflected Vector r
    # r = 2 * (n . v) * n - v ? No, standard reflection is r = 2(n.v)n - v for incoming light.
    # The paper says "evaluated at the reflected vector r". 
    # In the context of view-dependent appearance, r is typically the reflection of the view vector 
    # or the half-vector. The paper uses n.v in the Fresnel approximation.
    # Let's assume r is the reflection of the view direction v about the normal n.
    # r = 2 * dot(n, v) * n - v
    dot_nv = dot(normal, v)
    r = 2 * dot_nv * normal - v
    r = normalize(r)
    
    # 3. Compute Spatial Features Fs
    # Find the tile containing x
    tile_id = spatial_grid.get_tile_id(x)
    # Interpolate from the 3 planes Fx, Fy, Fz within the tile
    # The planes are 16x16. x is relative to tile origin.
    local_x = x - spatial_grid.get_tile_origin(tile_id)
    Fs = spatial_grid.interpolate_planes(tile_id, local_x, ns)
    
    # 4. Compute Angular Features Fa
    # Tri-linearly interpolate the 8 corner probes of the tile
    # Each probe has l^2 * na coefficients
    Fa = probe_grid.interpolate_probes(tile_id, x, l, na, r)
    
    # 5. Compute Grazing Angle Terms (Fresnel approximation)
    # Terms: (1 - n . v)^0 ... (1 - n . v)^5
    grazing_base = 1.0 - dot_nv
    grazing_terms = [pow(grazing_base, k) for k in range(6)]
    
    # 6. MLP Decoding
    # MLP: 2 hidden layers, 32 neurons, ReLU, sigmoid output
    input_vector = concatenate(Fs, Fa, grazing_terms)
    color = tiny_mlp_decode(input_vector)
    
    return color

def tiny_mlp_decode(input_vector):
    """
    Tiny MLP: 2 hidden layers, 32 neurons, ReLU, sigmoid output.
    """
    # Layer 1
    h1 = relu(dot(input_vector, W1) + b1)
    # Layer 2
    h2 = relu(dot(h1, W2) + b2)
    # Output
    color = sigmoid(dot(h2, W3) + b3)
    return color
```

**Edge Cases and Pitfalls:**
*   **Lighting Extrapolation:** Lighting encoded locally degrades extrapolation capabilities, causing shadow artifacts when looking from above or below on datasets with limited camera overlap along latitude (e.g., ActorsHQ legs) [S1].
*   **Local Minima:** Shape-radiance ambiguities can lead to local minima because light transport is not explicitly simulated [S1].
*   **Relighting:** Relighting and material editing are not supported [S1].

## Operating characteristics
*   **Time Complexity:** Training converges in minutes (e.g., ~1 min for MVMannequins, ~150s for DTU) [S1]. Rendering is real-time (200-400 Hz) [S1].
*   **Memory Complexity:** Probes require 1/6th of the memory taken by spatial features when $l=4$ and $n_s=n_a$ [S1]. Model sizes range from 30MB (MVMannequins) to 232MB (ActorsHQ full res) [S1].
*   **Performance:**
    *   **MVMannequins:** PSNR 36.81-36.90, Chamfer 1.03-1.04mm [S1].
    *   **ActorsHQ:** PSNR 37.48 (r/1), 36.62 (r/4) [S1].
    *   **DTU:** PSNR 37.03-37.74, Chamfer 0.68-0.71mm [S1].
    *   **BlendedMVS:** PSNR 35.19-35.89, Chamfer 2.22-2.37mm [S1].
*   **Strengths:** Fast training, real-time rendering, low memory footprint compared to Plenoxels (27 params/voxel) or 3D Gaussian Splatting (48 params/Gaussian) [S1].
*   **Limitations:** Targets objects with mild specularities; cone tracing methods are better for strong specularities but require >10 GPU-hours [S1]. Not suitable for relighting [S1].

## Background
ProbeSDF was introduced in the paper "ProbeSDF: Light Field Probes For Neural Surface Reconstruction" [S1]. It builds on the idea of using light field probes, a technique popular in explicit rendering, and adapts it to neural surface reconstruction using SDFs [S1]. The method leverages the observation that angular components of the light field are less sensitive to parallax than spatial components, allowing for a coarser representation [S1]. It is evaluated on datasets like DTU, BlendedMVS, MVMannequins, and ActorsHQ [S1].

## Sources
[S1] ProbeSDF: Light Field Probes For Neural Surface Reconstruction, https://doi.org/10.1109/CVPR52734.2025.01030, fetched 2026-10-26
