# Adaptive Differentiable Grids for Cryo-ET

Also called: Adaptive density grid optimization, Adaptive density grids based on an octree structure, Adaptive differentiable grids, Coordinate-based representation with adaptive octree

## Overview
Adaptive Differentiable Grids (ADG) is a learning-based representation for cryo-electron tomography (cryo-ET) reconstruction that replaces the fully connected MLPs of traditional neural fields with an adaptive octree structure, where each node stores a learnable 3D density grid [S1]. It solves the challenge of reconstructing high-noise, low-signal-to-noise ratio (SNR) cryo-ET data by combining differentiable image formation with regularization terms (Total Variation, boundary consistency, and cross-nodes non-local constraints) to separate noise from signal while preserving fine features [S1]. Unlike coordinate-based neural networks which suffer from long training times and slow rendering, ADG uses trilinear interpolation for faster querying and allows better detail recovery in complex regions while denoising uniform areas [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Noisy 2D projection images $b$ (preprocessed with motion and intensity correction), acquisition geometry (tilt angles), and a Region of Interest (ROI) mask.
*   **Outputs:** A continuous 3D density field $\hat{x}$ reconstructed within the ROI, obtained by querying the optimized octree structure.

**Data Structures:**
1.  **Octree:** A hierarchical tree structure where each node represents a sub-volume of the ROI.
    *   **Node State:** Each node is either *enabled* (active), *disabled* (outside ROI or empty), or *leaf* (contains a density grid).
    *   **Density Grid:** Each enabled leaf node stores a 3D tensor of learnable density values. The grid dimensions are $N_x \times N_y \times N_z$. In the source implementation, $N_x = N_y = N_z$ for all nodes [S1].
2.  **Ray Sampling State:** For each ray, a list of 3D sample positions is generated based on the current octree structure.

**Numbered Steps:**

1.  **Initialization:**
    *   Initialize the octree to cover the ROI.
    *   Disable all nodes outside the ROI [S1].
    *   Initialize the density values in the enabled nodes (e.g., to zero or small random values) [G].

2.  **Octree Update (Coarse Phase):**
    *   Use down-sampled projections to update the octree structure [S1].
    *   Uniformly sample each octant (node) and compute the standard deviation (SD) of the densities inside each node [S1].
    *   If the SD is significant, the node has a higher chance of being divided into eight child octants [S1].
    *   Solve a mixed-integer program to divide, merge, or keep octants, ensuring the total number of nodes remains below a fixed limit [S1].
    *   *Note:* This step prioritizes regions with more features and accelerates computation [S1].

3.  **Freeze Structure and Optimize Grids (Fine Phase):**
    *   Freeze the octree structure (no further division/merging) [S1].
    *   Optimize the density grid values in each enabled node using the original noisy projections [S1].
    *   Apply a binary mask $M$ to exclude rays intersecting fiducial markers or padded gray regions at the borders [S1].

4.  **Ray Sampling:**
    *   For each ray, determine the intersection with the current octree nodes [S1].
    *   For each octree node $oc$ crossed by the ray, calculate the number of samples $N_{oc}$ using stratified random sampling:
        $$ N_{oc} = N_{max} \frac{loc}{doc} $$
        where $N_{max}$ is a hyperparameter (maximum samples per node), $loc$ is the length of the ray segment inside node $oc$, and $doc$ is the diagonal length of node $oc$ [S1].
    *   Generate $N_{oc}$ 3D positions along the ray segment within node $oc$ [S1].

5.  **Density Query:**
    *   For a query point $p$, identify the containing octree node [S1].
    *   Perform trilinear interpolation from the eight vertices of the density grid stored in that node to obtain the density $f_\phi(p)$ [S1].
    *   Formula:
        $$ f_\phi(p) = \text{trilinear\_interp}(p, \text{Grid}_{node(p)}) $$
        where $\text{Grid}_{node(p)}$ is the $N_x \times N_y \times N_z$ tensor of the node containing $p$ [S1].

6.  **Loss Computation:**
    *   **Data Fidelity Loss ($L_{data}$):**
        Discretize the image formation model $b_i = A_i x_i + n_i$, where $A_i$ is the Radon transform along ray $i$, $x_i$ is the vector of sampled densities, and $b_i$ is the measured intensity [S1].
        $$ L_{data}(\phi) = \frac{1}{N} \sum_{i=1}^{N} || M (A f_\phi(p_i) - b_i) ||_2^2 $$
        where $N$ is the batch size (number of sampling points), $M$ is the binary mask excluding fiducial markers, and $f_\phi(p_i)$ is the density at sample point $p_i$ [S1].
    *   **Total Variation Loss ($L_{tv}$):**
        Spatial regularizer to smooth the reconstruction [S1]. (Formula not explicitly detailed in the provided text, but cited as a standard prior) [S1].
    *   **Cross-Nodes Non-Local Constraint Loss ($L_{cnlc}$):**
        To reduce noise and enforce consistency between different nodes:
        1.  Randomly select two enabled nodes $m$ and $n$ inside the ROI at each iteration [S1].
        2.  Sample densities $f_\phi(p_i^m)$ from node $m$ and $f_\phi(p_j^n)$ from node $n$ [S1].
        3.  Compute the loss:
            $$ L_{cnlc} = \sum || \text{softmax}(f_\phi(p_i^m)^T f_\phi(p_j^n)) f_\phi(p_j^n) - f_\phi(p_i^m) ||_2^2 $$
            The softmax operation estimates similarities between density vectors in different nodes [S1].
    *   **Boundary Consistency Loss ($L_{bc}$):**
        Minimizes density differences between neighboring grids at their shared boundaries to handle discontinuity artifacts caused by independent optimization of octree nodes [S1].
    *   **Total Loss:**
        $$ L_{total} = L_{data} + k_{tv} L_{tv} + k_{cnlc} L_{cnlc} + k_{bc} L_{bc} $$
        where $k_{tv}, k_{cnlc}, k_{bc}$ are weighting parameters [S1].

7.  **Optimization Loop:**
    *   Compute gradients of $L_{total}$ with respect to the density grid values using backpropagation through the differentiable image formation model and trilinear interpolation [S1].
    *   Update the density grid values using an optimizer (e.g., Adam) [G].
    *   Repeat steps 2-6 until convergence.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def trilinear_interp(x, y, z, grid):
    """
    Performs trilinear interpolation on a 3D grid.
    grid: numpy array of shape (Nx, Ny, Nz)
    x, y, z: float coordinates normalized to [0, 1] relative to the grid bounds
    Returns: interpolated density value
    """
    # Scale to grid indices
    ix = x * (grid.shape[0] - 1)
    iy = y * (grid.shape[1] - 1)
    iz = z * (grid.shape[2] - 1)
    
    # Floor and ceil indices
    x0, x1 = int(np.floor(ix)), int(np.ceil(ix))
    y0, y1 = int(np.floor(iy)), int(np.ceil(iy))
    z0, z1 = int(np.floor(iz)), int(np.ceil(iz))
    
    # Clamp indices to grid bounds
    x0 = np.clip(x0, 0, grid.shape[0] - 1)
    x1 = np.clip(x1, 0, grid.shape[0] - 1)
    y0 = np.clip(y0, 0, grid.shape[1] - 1)
    y1 = np.clip(y1, 0, grid.shape[1] - 1)
    z0 = np.clip(z0, 0, grid.shape[2] - 1)
    z1 = np.clip(z1, 0, grid.shape[2] - 1)
    
    # Interpolation weights
    wx = ix - x0
    wy = iy - y0
    wz = iz - z0
    
    # Trilinear interpolation
    c00 = grid[x0, y0, z0] * (1 - wx) + grid[x1, y0, z0] * wx
    c01 = grid[x0, y1, z0] * (1 - wx) + grid[x1, y1, z0] * wx
    c10 = grid[x0, y0, z1] * (1 - wx) + grid[x1, y0, z1] * wx
    c11 = grid[x0, y1, z1] * (1 - wx) + grid[x1, y1, z1] * wx
    
    v0 = c00 * (1 - wy) + c01 * wy
    v1 = c10 * (1 - wy) + c11 * wy
    
    return v0 * (1 - wz) + v1 * wz

def compute_radon_transform(samples, densities, ray_direction, origin):
    """
    Approximates the Radon transform (line integral) for a set of samples along a ray.
    samples: list of 3D points along the ray
    densities: array of density values at those points
    ray_direction: unit vector of the ray
    origin: starting point of the ray
    Returns: projected intensity
    """
    # Calculate distances between consecutive samples
    distances = []
    for i in range(len(samples) - 1):
        p1 = np.array(samples[i])
        p2 = np.array(samples[i+1])
        dist = np.linalg.norm(p2 - p1)
        distances.append(dist)
    
    # Simple Riemann sum approximation of the integral
    # Note: The source defines A_i as the Radon transform operation.
    # In practice, this involves integrating density along the ray path.
    # For implementation, we sum density * step_size.
    # Assuming uniform sampling for simplicity in this placeholder, 
    # but source uses stratified sampling.
    total_integral = 0.0
    for i in range(len(densities)):
        # Step size can be approximated by distance to next sample or half-distance
        step = distances[i] if i < len(distances) else distances[-1]
        total_integral += densities[i] * step
        
    return total_integral

def update_octree_structure(octree, projections_downsampled):
    """
    Updates the octree structure based on density variance.
    """
    for node in octree.enabled_nodes:
        # Sample densities uniformly within the node's grid
        densities = sample_uniformly(node.grid)
        sd = np.std(densities)
        
        # If SD is significant, mark for division
        if sd > threshold:
            node.mark_for_division()
            
    # Solve mixed-integer program to divide, merge, or keep nodes
    # ensuring total nodes < max_nodes
    octree.solve_mip()

def train_adg(projections, mask, roi, max_iterations):
    """
    Main training loop for Adaptive Differentiable Grids.
    """
    # 1. Initialize Octree
    octree = Octree(roi)
    octree.disable_outside_nodes()
    
    # 2. Coarse Phase: Update Octree Structure
    # Use down-sampled projections
    projections_coarse = downsample(projections)
    for _ in range(coarse_iterations):
        update_octree_structure(octree, projections_coarse)
        
    # 3. Fine Phase: Optimize Density Grids
    # Freeze octree structure
    octree.freeze_structure()
    
    for iteration in range(max_iterations):
        # Ray Sampling
        rays = generate_rays(projections)
        all_samples = []
        all_densities = []
        
        for ray in rays:
            # Determine intersection with octree nodes
            intersections = octree.intersect_ray(ray)
            
            ray_samples = []
            ray_densities = []
            
            for node, start_point, end_point in intersections:
                # Calculate number of samples for this node
                loc = np.linalg.norm(end_point - start_point)
                doc = node.diagonal_length
                N_oc = max(1, int(N_max * loc / doc))
                
                # Stratified random sampling along the segment
                points = stratified_sample(start_point, end_point, N_oc)
                
                # Query densities
                for p in points:
                    density = trilinear_interp(*p, node.grid)
                    ray_samples.append(p)
                    ray_densities.append(density)
                    
            all_samples.extend(ray_samples)
            all_densities.extend(ray_densities)
            
        # Compute Data Fidelity Loss
        # A is the Radon transform operator
        # b is the measured intensity
        # M is the mask
        projected = compute_radon_transform(all_samples, all_densities, ...)
        loss_data = 0.5 * np.sum(mask * (projected - projections) ** 2) / len(all_samples)
        
        # Compute Regularization Losses
        loss_tv = compute_tv_loss(octree)
        loss_bc = compute_boundary_consistency_loss(octree)
        
        # Compute CNLC Loss
        # Randomly select two enabled nodes
        node_m, node_n = random_select_two_enabled_nodes(octree)
        samples_m = random_sample_from_grid(node_m.grid)
        samples_n = random_sample_from_grid(node_n.grid)
        
        densities_m = [trilinear_interp(*s, node_m.grid) for s in samples_m]
        densities_n = [trilinear_interp(*s, node_n.grid) for s in samples_n]
        
        # L_cnlc formula
        # L_cnlc = sum || softmax(f_m^T f_n) f_n - f_m ||^2
        # Note: The source formula implies vector operations. 
        # Assuming densities are scalars per point, this might be a typo in source 
        # or implies feature vectors. Given "density grids", densities are scalars.
        # If densities are scalars, softmax of scalar product is just sigmoid-like.
        # We implement the formula as written, assuming f is scalar.
        # If f is scalar, f^T f is just product.
        
        cnlc_loss = 0.0
        for i in range(len(samples_m)):
            # Assuming single density value per point
            f_m = densities_m[i]
            f_n = densities_n[i] # Assuming paired sampling or similar indices
            
            # Softmax of scalar product? 
            # Source: softmax(f_phi(p_i^m)^T f_phi(p_j^n))
            # If these are scalars, softmax is exp(x)/sum(exp). 
            # With one term, it's 1. This suggests f might be a vector or the formula 
            # implies a sum over neighbors. 
            # However, source says "sample densities f_phi(p)". 
            # We will implement the scalar version as a placeholder for the exact vector logic 
            # which is not fully detailed in the text provided.
            # [G] Assuming the source implies a non-local operation on feature vectors, 
            # but only density is mentioned. We stick to the scalar density interpretation 
            # and note the ambiguity.
            
            # Placeholder for CNLC calculation based on scalar densities
            # This part is ambiguous in the source text regarding vector vs scalar.
            # We will skip detailed implementation of CNLC as the source 
            # does not provide the full vector definition or how p_i and p_j are paired.
            pass 
            
        loss_cnlc = 0.0 # Placeholder
        
        loss_total = loss_data + k_tv * loss_tv + k_cnlc * loss_cnlc + k_bc * loss_bc
        
        # Backpropagate and update grid values
        gradients = compute_gradients(loss_total, octree)
        update_grids(octree, gradients)
        
    return octree
```

**Parameters:**
*   $N_x, N_y, N_z$: Grid size in x, y, z dimensions for each density grid. Typically $N_x = N_y = N_z$ [S1].
*   $N_{max}$: Hyperparameter for the maximum number of samples per node in ray sampling [S1].
*   $k_{tv}, k_{cnlc}, k_{bc}$: Weighting parameters for Total Variation, Cross-Nodes Non-Local Constraint, and Boundary Consistency losses, respectively [S1].

**Edge Cases and Pitfalls:**
*   **Noise:** Raw cryo-ET data follows a Poisson noise model, but preprocessing (motion/intensity correction) makes it approximately Gaussian [S1]. The method assumes Gaussian noise for simplicity [S1].
*   **Discontinuities:** Independent optimization of octree nodes causes discontinuity artifacts at grid edges. This is mitigated by the boundary consistency loss $L_{bc}$ [S1].
*   **Fiducial Markers:** Fiducial markers and padded gray regions interfere with reconstruction. A binary mask $M$ is applied to the data-fidelity loss to exclude rays intersecting these regions [S1].
*   **Inadequacy of Standard Neural Fields:** Standard MLP-based neural fields either learn noise as signal or oversmooth the signal, losing main features. ADG addresses this by using adaptive grids and specific regularizations [S1].

## Operating characteristics
*   **Time Complexity:** Faster than traditional MLP-based neural fields because querying density uses trilinear interpolation instead of evaluating a neural network at each sample point [S1]. The octree structure accelerates computation by prioritizing regions with features [S1].
*   **Memory Complexity:** Lower memory requirement than classical tomographic reconstruction techniques as it does not store the entire 3D volume during optimization, only the active octree nodes and their density grids [S1].
*   **Performance:** Evaluated on datasets EMPIAR 10643, EMPIAR 10453, and EMPIAR 11462 [S1]. Outperforms SART+TV, Kniesel et al., and NeAT in reconstruction quality and denoising [S1].
*   **Strengths:** Better detail preservation in complex regions, effective denoising in uniform regions, faster optimization and rendering than MLP-based methods [S1].
*   **Limitations:** Requires careful tuning of regularization weights ($k_{tv}, k_{cnlc}, k_{bc}$) and octree update constraints [S1]. The mixed-integer program for octree updates may be computationally intensive [S1].

## Background
*   **Origin:** Proposed by Wang et al. in the paper "Adaptive differentiable grids for cryo-electron tomography reconstruction and denoising" [S1].
*   **History:** Inspired by recent advances in neural representations (NeRFs) and adaptive octree structures (ACORN, NeAT) [S1]. Addresses specific challenges in cryo-ET: missing-wedge acquisition, sample misalignment, motion, large data processing, and low SNR [S1].
*   **Usage:** Used for cryo-electron tomography reconstruction and denoising [S1].

## Sources
[S1] Adaptive differentiable grids for cryo-electron tomography reconstruction and denoising, https://doi.org/10.1093/bioadv/vbad131, fetched 2026-10-27
