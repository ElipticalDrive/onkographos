# Developability Regularizer

Also called: Stein et al. regularizer

## Overview
The Developability Regularizer is a mesh regularization term that encourages geometric flatness (developability) by minimizing the smallest eigenvalue of the vertex normal covariance matrix, weighted by vertex area [S1, unofficial]. It builds on [Mesh Reconstruction Evaluation](2909-mesh-reconstruction-evaluation.md) [S1, unofficial].

## How it works
The method computes a per-vertex contribution to the loss based on the local geometry of incident triangles [S1, unofficial]. It aggregates normal covariances and areas, then computes the minimum eigenvalue as a measure of non-developability [S1, unofficial].

**Inputs:**
- `tri_pos`: Tensor of shape $(T, 3, 3)$, containing the 3D positions of the three vertices for each of the $T$ triangles [S1, unofficial].
- `tris`: Tensor of shape $(T, 3)$, containing the vertex indices for each triangle [S1, unofficial].
- $V$: Number of unique vertices in the mesh [S1, unofficial].

**Outputs:**
- `loss`: Scalar tensor representing the total developability loss [S1, unofficial].

**Data Structures:**
- `vert_normal_covariance_sum`: Tensor of shape $(V, 9)$, initialized to zeros. Stores the flattened $3 \times 3$ normal covariance matrix for each vertex [S1, unofficial].
- `vert_area`: Tensor of shape $(V)$, initialized to zeros. Stores the weighted area contribution for each vertex [S1, unofficial].
- `vert_degree`: Tensor of shape $(V)$, initialized to zeros (int32). Stores the number of triangles incident to each vertex [S1, unofficial].

**Parameters:**
- `REL_EPS`: $1 \times 10^{-6}$. Added to the covariance matrix to ensure numerical stability during eigenvalue decomposition [S1, unofficial].
- `POS_EPS`: $1 \times 10^{-6}$. Used in normalization denominators to prevent division by zero [S1, unofficial].

**Steps:**

1.  **Initialize Accumulators:**
    Set `vert_normal_covariance_sum`, `vert_area`, and `vert_degree` to zero tensors of the specified shapes [S1, unofficial].

2.  **Iterate Over Triangle Corners:**
    For each triangle $t$ in the mesh, and for each corner index $i_C \in \{0, 1, 2\}$:
    
    a.  **Gather Vertices:**
        Let $p_{Root} = \text{tri\_pos}[t, i_C]$, $p_A = \text{tri\_pos}[t, (i_C + 1) \pmod 3]$, and $p_B = \text{tri\_pos}[t, (i_C + 2) \pmod 3]$ [S1, unofficial].
        
    b.  **Compute Edge Vectors:**
        $v_A = p_A - p_{Root}$, $v_B = p_B - p_{Root}$, and normalize them to $v_{An}$ and $v_{Bn}$ [S1, unofficial].
        
    c.  **Compute Face Geometry:**
        $area\_normal = v_A \times v_B$, $face\_area = 0.5 \times \|area\_normal\|$, and $normal = \frac{area\_normal}{\|area\_normal\| + \text{POS\_EPS}}$ [S1, unofficial].
        
    d.  **Compute Corner Angle:**
        $corner\_angle = \arccos(\text{clamp}(\text{dot}(v_{An}, v_{Bn}), -1, 1))$ [S1, unofficial].
        
    e.  **Compute Normal Outer Product:**
        $outer = normal \otimes normal$ (Resulting in a $3 \times 3$ matrix), then flatten to a vector of length 9 [S1, unofficial].
        
    f.  **Scatter Updates:**
        Let $idx = \text{tris}[t, i_C]$. Accumulate $outer$ to $\text{vert\_normal\_covariance\_sum}[idx]$, add $\frac{face\_area}{3}$ to $\text{vert\_area}[idx]$, and add 1 to $\text{vert\_degree}[idx]$ [S1, unofficial].

3.  **Process Covariance Matrices:**
    Reshape `vert_normal_covariance_sum` to $(V, 3, 3)$ and add $\text{REL\_EPS} \times I$ to each matrix, where $I$ is the identity matrix [S1, unofficial].

4.  **Compute Minimum Eigenvalues:**
    Compute the eigenvalues for each vertex's covariance matrix and take the minimum absolute value: $\text{min\_eigvals}[v] = \min(|\lambda_1|, |\lambda_2|, |\lambda_3|)$ [S1, unofficial].

5.  **Mask Degree-3 Vertices:**
    Set $\text{vert\_area}[v] = 0$ where $\text{vert\_degree}[v] == 3$ [S1, unofficial].

6.  **Normalize Area:**
    Normalize $\text{vert\_area}$ by scaling it: $\text{vert\_area} \leftarrow \text{vert\_area} \times \frac{V}{\sum \text{vert\_area}}$ [S1, unofficial].

7.  **Compute Loss:**
    Return the element-wise product $\text{vert\_area} \times \text{min\_eigvals}$, summed to a scalar [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def compute_developability_regularizer(tri_pos, tris, V, rel_eps=1e-6, pos_eps=1e-6):
    """
    Compute the Developability Regularizer loss.
    
    Args:
        tri_pos: Tensor (T, 3, 3) - Vertex positions for each triangle.
        tris: Tensor (T, 3) - Vertex indices for each triangle.
        V: int - Number of vertices.
        rel_eps: float - Regularization epsilon for eigenvalues.
        pos_eps: float - Epsilon for normalization.
        
    Returns:
        loss: Tensor (1,) - Scalar loss value.
    """
    T = tri_pos.shape[0]
    
    # Initialize accumulators
    vert_normal_covariance_sum = torch.zeros((V, 9), device=tri_pos.device)
    vert_area = torch.zeros((V), device=tri_pos.device)
    vert_degree = torch.zeros((V), dtype=torch.int32, device=tri_pos.device)
    
    # Loop over triangles and corners
    for iC in range(3):
        # Gather vertices
        p_root = tri_pos[:, iC]
        p_a = tri_pos[:, (iC + 1) % 3]
        p_b = tri_pos[:, (iC + 2) % 3]
        
        # Edge vectors
        v_a = p_a - p_root
        v_b = p_b - p_root
        
        # Normalize edges
        norm_v_a = torch.norm(v_a, dim=1, keepdim=True)
        norm_v_b = torch.norm(v_b, dim=1, keepdim=True)
        v_an = v_a / (norm_v_a + pos_eps)
        v_bn = v_b / (norm_v_b + pos_eps)
        
        # Face area and normal
        area_normal = torch.cross(v_a, v_b, dim=1)
        face_area = 0.5 * torch.norm(area_normal, dim=1)
        
        norm_area_normal = torch.norm(area_normal, dim=1, keepdim=True)
        normal = area_normal / (norm_area_normal + pos_eps)
        
        # Compute corner angle (computed per source, but unused in final loss)
        corner_angle = torch.acos(torch.clamp(torch.sum(v_an * v_bn, dim=1), -1.0, 1.0))
        
        # Outer product of normal (3x3 matrix)
        outer = torch.einsum('ti,tj->tij', normal, normal)
        outer_flat = outer.reshape(T, 9)
        
        # Scatter updates
        indices = tris[:, iC]
        vert_normal_covariance_sum.scatter_add_(0, indices.unsqueeze(1).expand(-1, 9), outer_flat)
        vert_area.scatter_add_(0, indices, face_area / 3.0)
        vert_degree.scatter_add_(0, indices, torch.ones(T, dtype=torch.int32, device=tri_pos.device))

    # Reshape covariance to (V, 3, 3)
    cov_matrix = vert_normal_covariance_sum.reshape(V, 3, 3)
    
    # Add regularization
    identity = torch.eye(3, device=tri_pos.device)
    cov_matrix = cov_matrix + rel_eps * identity
    
    # Compute eigenvalues
    eigvals = torch.linalg.eigvalsh(cov_matrix)
    
    # Minimum absolute eigenvalue
    min_eigvals = torch.min(torch.abs(eigvals), dim=1).values
    
    # Mask out degree-3 vertices
    mask_degree_3 = (vert_degree == 3)
    vert_area = vert_area.clone()
    vert_area[mask_degree_3] = 0.0
    
    # Normalize area
    total_area = vert_area.sum()
    vert_area = vert_area * (V / (total_area + pos_eps))
        
    # Final loss
    loss = torch.sum(vert_area * min_eigvals)
    
    return loss
```

## Operating characteristics
Not given in the sources.

## Background
The Developability Regularizer originates from the FlexiCubes framework [S1, unofficial]. It is used as a regularization term in differentiable surface reconstruction to encourage mesh flatness [S1, unofficial].

## Sources
[S1] GitHub - nv-tlabs/FlexiCubes, https://github.com/nv-tlabs/FlexiCubes, fetched 2026-10-27 (unofficial)
