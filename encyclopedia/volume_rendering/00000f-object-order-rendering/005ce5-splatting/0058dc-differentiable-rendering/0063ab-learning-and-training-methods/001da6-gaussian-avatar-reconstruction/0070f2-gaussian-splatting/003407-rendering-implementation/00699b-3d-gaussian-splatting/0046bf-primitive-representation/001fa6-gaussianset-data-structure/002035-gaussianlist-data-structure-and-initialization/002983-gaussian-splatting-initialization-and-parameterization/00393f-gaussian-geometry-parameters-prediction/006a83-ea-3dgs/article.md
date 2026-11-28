# EA-3DGS

Also called: Enhanced 3DGS

## Overview
EA-3DGS is a method for large-scale 3D scene representation that addresses the instability and memory inefficiency of standard 3D Gaussian Splatting (3DGS) by using an adaptive tetrahedral mesh for initialization, structure-aware pruning/densification for stability, and vector quantization for compression [S1, unofficial]. It sets itself apart from partitioning-based large-scale methods (e.g., CityGaussian, DoGaussian) by enabling single-GPU training without complex synchronization or merging of multiple nodes, thereby preserving global information like lighting [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Multi-view images of a large-scale scene.
*   **Output:** A compressed set of 3D Gaussian primitives (centers, covariances, opacities, spherical harmonics coefficients) ready for differentiable rendering.

**Data Structures:**
1.  **Adaptive Tetrahedral Mesh:** Generated from a sparse point cloud (via COLMAP) using Delaunay triangulation. It consists of tetrahedra $T$ and triangular faces $V$.
2.  **Gaussian Set:** A fixed set of $N = k \cdot n$ Gaussians, where $n$ is the number of mesh faces and $k$ is the number of Gaussians per face.
    *   Each Gaussian $i$ has parameters: position $\mu_i$, opacity $\alpha_i$, covariance $\Sigma_i$, and color $c_i$ (represented by spherical harmonics).
    *   Position $\mu_i$ is not stored directly but defined by trainable barycentric coordinates $(\alpha_{1,i}, \alpha_{2,i}, \alpha_{3,i})$ relative to the vertices of the mesh face $V$ it belongs to.
3.  **Codebook:** A small set of representative vectors used for Vector Quantization (VQ) of Gaussian attributes to reduce memory usage.

**Steps:**

1.  **Mesh Generation:**
    *   Reconstruct a sparse point cloud from input multi-view images using COLMAP [S1, unofficial].
    *   Perform Delaunay triangulation on the point cloud to generate an adaptive tetrahedral mesh covering the scene volume [S1, unofficial].

2.  **Gaussian Initialization:**
    *   For each triangular face $V$ of the tetrahedral mesh with vertices $v_1, v_2, v_3$:
        *   Initialize $k$ Gaussian components.
        *   For each Gaussian $j$ on face $V$, define its center position $P$ as a convex combination of the vertex positions:
            $$ P(\alpha_1, \alpha_2, \alpha_3) = \alpha_1 v_1 + \alpha_2 v_2 + \alpha_3 v_3 $$
            where $\alpha_1, \alpha_2, \alpha_3$ are trainable parameters constrained by $\alpha_1 + \alpha_2 + \alpha_3 = 1$ [S1, unofficial].
        *   Initialize other parameters (opacity, covariance, color) similar to standard 3DGS, ensuring $\Sigma$ is decomposed into scaling $S$ and rotation $R$ such that $\Sigma = RSS^T R^T$ for positive semi-definiteness [S1, unofficial].
    *   The total number of Gaussians is fixed at $N = k \cdot n$ [S1, unofficial].

3.  **Rendering:**
    *   For each pixel, project 3D Gaussians to 2D. The projected 2D covariance matrix $\Sigma'$ is calculated as:
        $$ \Sigma' = J W \Sigma W^T J^T $$
        where $W$ is the view transformation matrix, $\Sigma$ is the 3D covariance matrix, and $J$ is the Jacobian of the affine approximation of the projective transformation [S1, unofficial].
    *   Compute the 3D Gaussian function value $G(\chi)$ at position $\chi$ (centered at mean $\mu$):
        $$ G(\chi) = e^{-\frac{1}{2} \chi^T \Sigma^{-1} \chi} $$
    *   Calculate effective opacity $\alpha'_i$ as the product of learned opacity $\alpha_i$ and a function of $\Sigma$ and spherical harmonics [S1, unofficial].
    *   Render pixel color $C$ using alpha compositing over a depth-sorted list of overlapping Gaussians $N$:
        $$ C = \sum_{i \in N} c_i \alpha'_i \prod_{j'=1}^{i-1} (1 - \alpha'_{j'}) $$
        where $c_i$ is the learned color and $\alpha'_i$ is the effective opacity [S1, unofficial].

4.  **Pruning and Densification (Structure-Aware):**
    *   **Pruning:** Assess the global importance of each Gaussian based on training observations (e.g., contribution to loss or gradient magnitude). Prune Gaussians with low importance to reduce redundancy [S1, unofficial].
    *   **Densification:** To mitigate artifacts from excessive pruning (e.g., blurry regions), apply structure-aware densification. Specifically, increase Gaussian density in low-curvature regions of the mesh or scene [S1, unofficial]. This contrasts with standard 3DGS cloning/splitting which relies heavily on initial point cloud quality and often fails to grow points in sparse areas, leading to needle-like artifacts [S1, unofficial].

5.  **Compression (Vector Quantization):**
    *   Apply Vector Quantization (VQ) to compress Gaussian component parameters.
    *   Transform the large set of Gaussian attribute vectors into a smaller codebook of representative vectors.
    *   Quantize original vectors by assigning them to the nearest codebook entry, significantly reducing memory usage [S1, unofficial].

**Parameters:**
*   $k$: Number of Gaussian components initialized on each mesh face. Typical values are not explicitly given in the sources, but the total count is $k \cdot n$ [S1, unofficial].
*   Compression Ratio: EA-3DGS achieves 5.2x compression [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Sparse Initialization Artifacts:** Standard 3DGS adaptive density control fails to grow points in areas where the initial point cloud is sparse, resulting in blurry or needle-like artifacts. EA-3DGS mitigates this via mesh-based initialization and structure-aware densification [S1, unofficial].
*   **Global Information Loss:** Partitioning methods may lose global information like lighting during merging. EA-3DGS avoids this by using single-GPU training [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def generate_tetrahedral_mesh(sparse_point_cloud):
    """
    Generate adaptive tetrahedral mesh from sparse point cloud using Delaunay triangulation.
    Returns: list of tetrahedra, list of triangular faces with vertex indices.
    """
    # Implementation detail: Use a Delaunay triangulation library (e.g., scipy.spatial.Delaunay)
    # on the sparse_point_cloud.
    # Extract triangular faces from the tetrahedra.
    return tetrahedra, triangular_faces

def initialize_gaussians(triangular_faces, k):
    """
    Initialize Gaussian components on each mesh face.
    Args:
        triangular_faces: List of faces, each face is a list of 3 vertex positions [v1, v2, v3].
        k: Number of Gaussians per face.
    Returns:
        Gaussians: List of Gaussian objects.
    """
    gaussians = []
    for face in triangular_faces:
        v1, v2, v3 = face
        for _ in range(k):
            # Initialize trainable barycentric coordinates
            # In practice, these are learned parameters, initialized randomly or uniformly
            # subject to alpha1 + alpha2 + alpha3 = 1
            alphas = np.random.dirichlet([1, 1, 1])
            
            # Calculate position
            position = alphas[0] * v1 + alphas[1] * v2 + alphas[2] * v3
            
            # Initialize other parameters (opacity, covariance, color)
            # Covariance Sigma = R S S^T R^T
            # Color c (Spherical Harmonics coefficients)
            gaussian = {
                'alphas': alphas,
                'vertices': [v1, v2, v3],
                'position': position,
                'opacity': 0.5, # Initial guess
                'covariance': np.eye(3), # Initial guess
                'color': np.zeros(3) # Initial guess
            }
            gaussians.append(gaussian)
    return gaussians

def render_gaussians(gaussians, camera_params):
    """
    Render scene using alpha compositing.
    Args:
        gaussians: List of Gaussian objects.
        camera_params: View matrix W, projection parameters for Jacobian J.
    Returns:
        Image: Rendered pixel colors.
    """
    # Project 3D Gaussians to 2D
    projected_gaussians = []
    for g in gaussians:
        # Calculate projected 2D covariance Sigma' = J W Sigma W^T J^T
        # W is view transformation, J is Jacobian of affine approximation
        Sigma_prime = calculate_projected_covariance(g['covariance'], camera_params['W'], camera_params['J'])
        
        # Calculate effective opacity alpha'
        # alpha' = alpha * f(Sigma, SH)
        alpha_prime = g['opacity'] * calculate_opacity_factor(g['covariance'], g['color'])
        
        projected_gaussians.append({
            'position_2d': project_position(g['position'], camera_params),
            'covariance_2d': Sigma_prime,
            'opacity': alpha_prime,
            'color': g['color']
        })
    
    # Sort by depth
    projected_gaussians.sort(key=lambda x: x['depth'])
    
    # Alpha compositing
    image = np.zeros((height, width, 3))
    for pixel in pixels:
        color = 0.0
        accumulated_opacity = 0.0
        for g in projected_gaussians:
            # Check if Gaussian covers pixel
            if covers_pixel(g, pixel):
                # Calculate Gaussian value at pixel
                chi = pixel - g['position_2d']
                # G(chi) = exp(-0.5 * chi^T Sigma'^-1 chi)
                g_value = np.exp(-0.5 * chi.T @ np.linalg.inv(g['covariance_2d']) @ chi)
                
                contribution = g['color'] * g['opacity'] * g_value
                color += contribution * (1 - accumulated_opacity)
                accumulated_opacity += g['opacity'] * g_value * (1 - accumulated_opacity)
                
                if accumulated_opacity > 0.99:
                    break
        image[pixel.y, pixel.x] = color
    return image

def prune_and_densify(gaussians, training_data):
    """
    Prune low-importance Gaussians and densify low-curvature regions.
    Args:
        gaussians: List of Gaussian objects.
        training_data: List of training images/views.
    Returns:
        Updated list of Gaussians.
    """
    # Calculate importance score for each Gaussian
    scores = calculate_gaussian_scores(gaussians, training_data)
    
    # Prune Gaussians with low scores
    # Threshold is not specified in sources [G]
    pruned_gaussians = [g for g, s in zip(gaussians, scores) if s > threshold]
    
    # Structure-aware densification in low-curvature regions
    # Identify low-curvature regions on the mesh
    low_curvature_faces = identify_low_curvature_faces(mesh)
    
    # Add new Gaussians to these regions
    for face in low_curvature_faces:
        # Clone or split existing Gaussians in this region
        # Specific strategy for cloning/splitting is not detailed in sources [G]
        pass
        
    return pruned_gaussians

def vector_quantize(gaussians, codebook_size):
    """
    Compress Gaussian parameters using Vector Quantization.
    Args:
        gaussians: List of Gaussian objects.
        codebook_size: Number of representative vectors in codebook.
    Returns:
        Quantized Gaussians, Codebook.
    """
    # Extract all Gaussian attribute vectors
    all_vectors = extract_attributes(gaussians)
    
    # Train codebook using VQ algorithm (e.g., LBG or k-means)
    codebook = train_codebook(all_vectors, codebook_size)
    
    # Quantize each Gaussian attribute to nearest codebook entry
    quantized_gaussians = []
    for g in gaussians:
        # Find nearest codebook entry
        nearest_idx = np.argmin([np.linalg.norm(attr - cb) for cb in codebook])
        quantized_gaussians.append({
            **g,
            'codebook_index': nearest_idx,
            'quantized_attributes': codebook[nearest_idx]
        })
    return quantized_gaussians, codebook
```

## Operating characteristics
*   **Memory:** EA-3DGS achieves 5.2x compression compared to baseline 3DGS, reducing scene size from 1.8GB to 345MB [S1, unofficial].
*   **Performance:** PSNR drops from 25.19 (baseline) to 24.82 (EA-3DGS) after compression [S1, unofficial].
*   **Hardware:** Designed for single-GPU training, avoiding the complexity and potential information loss of multi-node/partitioned approaches [S1, unofficial].
*   **Strengths:** Stable training in large-scale scenes; avoids blurry/needle-like artifacts from sparse initialization; efficient memory usage via VQ; no need for complex partition synchronization.
*   **Limitations:** Fixed number of Gaussians ($k \cdot n$) may not adapt as dynamically as standard 3DGS cloning/splitting in all scenarios; relies on the quality of the initial tetrahedral mesh.

## Background
EA-3DGS was proposed to address the limitations of standard 3DGS in large-scale scenes, specifically the reliance on initial point cloud quality and the high memory cost of storing numerous Gaussian attributes [S1, unofficial]. It builds on the 3DGS rendering pipeline but introduces an adaptive tetrahedral mesh for initialization and structure-aware densification/pruning [S1, unofficial]. The method is particularly suited for outdoor scenes where large-scale representation and efficient compression are critical [S1, unofficial].

## Sources
[S1] EA-3DGS: Efficient and Adaptive 3D Gaussians with Highly Enhanced Quality for outdoor scenes, https://doi.org/10.48550/arXiv.2505.10787, fetched 2026-10-30 (unofficial)
