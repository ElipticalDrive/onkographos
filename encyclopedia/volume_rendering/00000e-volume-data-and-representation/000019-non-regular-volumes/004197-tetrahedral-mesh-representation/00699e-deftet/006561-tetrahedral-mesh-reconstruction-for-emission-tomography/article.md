# Tetrahedral Mesh Reconstruction for Emission Tomography

Also called: Adaptive Tetrahedral Mesh Reconstruction, Mesh-adaptive Emission Tomography

## Overview
A method for image reconstruction on tetrahedral meshes optimized for emission computed tomography (ECT) with parallel beam geometry, where image intensities are defined at mesh nodes and linearly interpolated inside each tetrahedron [S1]. It differs from standard voxel-based reconstruction by using a deformable mesh that adapts node density to intensity gradients, thereby reducing noise in uniform regions while preserving boundary details [S1]. It builds on [DEFTET](6415-deftet.md) by providing the specific reconstruction pipeline and system matrix formulation for ECT applications.

## How it works
**Inputs:**
*   Tomographic projection data (sinograms).
*   Initial point cloud defining the domain (typically a finely-spaced uniform cloud).
*   System parameters: parallel beam geometry, attenuation coefficients, detector geometric response.

**Outputs:**
*   Optimized tetrahedral mesh geometry (node positions and connectivity).
*   Reconstructed image intensities at mesh nodes.

**Data Structures:**
*   **Mesh:** Defined by a set of nodes $V$ and a set of tetrahedra $T$. Each tetrahedron $t \in T$ is defined by 4 nodes.
*   **Intensity:** Scalar value $I_v$ associated with each node $v \in V$.
*   **System Matrix:** $A$, where element $A_{ij}$ represents the contribution of node $j$ to projection bin $i$.

**Method Steps:**

1.  **Initial Reconstruction:**
    Reconstruct a noisy image on the initial finely-spaced uniform point cloud. This provides an initial estimate of the intensity distribution [S1].

2.  **Mesh Optimization (Two-Stage Process):**
    Adaptively transform the mesh geometry to optimize it for the specific patient data [S1].
    *   **Node Elimination:** Remove nodes located in regions of constant intensity to reduce computational load and noise [S1].
    *   **Node Merging:** Merge nodes along sharp boundaries to maintain geometric integrity [S1].
    *   **Node Motion:** Move nodes in the direction of the mean local intensity gradient. This increases node density in boundary regions where high spatial frequency information exists [S1].

3.  **System Matrix Calculation:**
    For the optimized mesh geometry, calculate the system matrix $A$ using an exact analytical formula [S1]. The calculation must include:
    *   Attenuation correction.
    *   Detector geometric response [S1].
    *   The contribution of linear interpolation within each tetrahedron to the parallel beam projections [S1].

4.  **Final Reconstruction:**
    Use the optimized mesh and the calculated system matrix $A$ to perform Maximum Likelihood-Expectation Maximization (ML-EM) reconstruction to solve for the node intensities $I_v$ [S1].

5.  **Dynamic Imaging (Optional):**
    For dynamic PET or SPECT imaging:
    *   Generate a quasi-static sinogram by summing projection data from multiple time frames [S1].
    *   Perform system matrix generation using this quasi-static sinogram [S1].
    *   Use this single system matrix to reconstruct individual time frame projections [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tetrahedral_mesh_reconstruction(projections, initial_point_cloud, attenuation_map, detector_response):
    """
    Reconstructs emission tomography data on an adaptive tetrahedral mesh.
    
    Args:
        projections: Array of sinogram data.
        initial_point_cloud: Array of 3D coordinates for the initial uniform cloud.
        attenuation_map: Attenuation coefficients for the domain.
        detector_response: Detector geometric response parameters.
        
    Returns:
        optimized_mesh: The final tetrahedral mesh structure.
        intensities: Array of intensity values at mesh nodes.
    """
    
    # Step 1: Initial Reconstruction on Uniform Cloud
    # Reconstruct a noisy image on the finely-spaced uniform cloud
    # This provides the initial intensity field for mesh adaptation
    initial_intensities = ml_em_reconstruction(projections, initial_point_cloud, attenuation_map, detector_response)
    
    # Step 2: Mesh Optimization
    # Create initial mesh from point cloud (e.g., Delaunay triangulation)
    mesh = create_tetrahedral_mesh(initial_point_cloud)
    
    # Two-stage adaptive transformation
    # Stage 1: Identify regions of constant intensity and boundaries
    intensity_gradients = compute_local_intensity_gradients(mesh, initial_intensities)
    
    # Remove nodes in constant intensity regions
    constant_regions = identify_constant_intensity_regions(mesh, initial_intensities, threshold=epsilon)
    mesh = remove_nodes(mesh, constant_regions)
    
    # Merge nodes along boundaries
    boundary_nodes = identify_boundary_nodes(mesh, intensity_gradients, threshold=gradient_threshold)
    mesh = merge_boundary_nodes(mesh, boundary_nodes)
    
    # Move nodes in the direction of the mean local intensity gradient
    # This provides higher node density in boundary regions
    for node in mesh.nodes:
        if node not in constant_regions and node not in boundary_nodes:
            gradient = intensity_gradients[node]
            node.position += learning_rate * gradient
            
    # Re-triangulate mesh to maintain tetrahedral connectivity after node movement
    mesh = retriangulate_mesh(mesh)
    
    # Step 3: System Matrix Calculation
    # Calculate system matrix A using exact analytical formula for parallel beam geometry
    # Include attenuation correction and detector geometric response
    system_matrix = calculate_system_matrix(mesh, projections.geometry, attenuation_map, detector_response)
    
    # Step 4: Final Reconstruction
    # Use ML-EM to reconstruct node intensities
    intensities = ml_em_reconstruction(projections, mesh, system_matrix, attenuation_map, detector_response)
    
    return mesh, intensities

def calculate_system_matrix(mesh, geometry, attenuation_map, detector_response):
    """
    Calculates the system matrix A where A_ij is the contribution of node j to projection bin i.
    Uses exact analytical formula for linear interpolation within tetrahedra.
    """
    num_projections = geometry.num_bins
    num_nodes = mesh.num_nodes
    A = zeros((num_projections, num_nodes))
    
    for i in range(num_projections):
        ray = geometry.get_ray(i)
        for j in range(num_nodes):
            node = mesh.nodes[j]
            # Calculate contribution based on linear interpolation within tetrahedra
            # intersected by the ray, including attenuation and detector response
            contribution = compute_tetrahedron_ray_intersection_contribution(
                node, mesh, ray, attenuation_map, detector_response
            )
            A[i, j] = contribution
            
    return A
```

**Parameters:**
*   **Gradient Threshold:** Determines what constitutes a "boundary" for node movement and merging. Not given in the sources.
*   **Constant Intensity Threshold:** Determines what constitutes a "constant intensity region" for node removal. Not given in the sources.
*   **Learning Rate:** Step size for node movement along the gradient. Not given in the sources.
*   **ML-EM Iterations:** Number of iterations for the reconstruction algorithm. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Degenerate Tetrahedra:** Node movement may create degenerate tetrahedra (zero volume). Re-triangulation is required to maintain mesh quality [S1].
*   **Boundary Preservation:** Node merging must preserve sharp boundaries to avoid blurring. The method specifies merging "along boundaries" but does not define the exact geometric criterion [S1].
*   **Quasi-static Assumption:** In dynamic imaging, the assumption that the sum of projections represents a valid static geometry may introduce errors if there is significant motion between frames. Not given in the sources.

## Operating characteristics
*   **Time Complexity:** The method involves iterative reconstruction (ML-EM) and mesh optimization. The system matrix calculation is $O(N_{proj} \times N_{nodes})$, where $N_{proj}$ is the number of projection bins and $N_{nodes}$ is the number of mesh nodes. The adaptive mesh optimization adds overhead but reduces $N_{nodes}$ in the final reconstruction. Not given in the sources.
*   **Memory Complexity:** Proportional to the number of mesh nodes and tetrahedra. The system matrix is sparse. Not given in the sources.
*   **Performance:** Evaluated on simulated projections of the NCAT phantom, dynamic SPECT phantom, patient studies, and dynamic microPET rat studies. Compared to standard voxel-based reconstruction for high and low signal-to-noise ratio datasets. Results demonstrate that tetrahedral mesh-based images represent the data effectively. Not given in the sources.
*   **Strengths:** Intrinsically three-dimensional, multi-level resolution property, reduced noise in uniform regions, preserved boundary details. Not given in the sources.
*   **Limitations:** Requires complex mesh optimization. System matrix calculation is computationally intensive. Not given in the sources.

## Background
*   **Origin:** The method was presented in a paper by [S1] in 2013.
*   **History:** It is a practical implementation of image reconstruction on tetrahedral meshes optimized for emission computed tomography. It builds on the concept of deformable meshes (DEFTET) but applies it specifically to ECT with parallel beam geometry.
*   **Applications:** PET and SPECT imaging, including dynamic studies.

## Sources
[S1] Practical implementation of tetrahedral mesh reconstruction in emission tomography, https://iopscience.iop.org/article/10.1088/0031-9155/58/9/3001/meta, fetched 2026-10-31
