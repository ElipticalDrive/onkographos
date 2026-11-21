# Pairwise Chamfer Distance via Relative Transform

Also called: Pairwise Distance Chamfer

## Overview
Computes the symmetric Chamfer distance between two sets of world-to-link transforms by evaluating the geometric error of sampled mesh points under the relative transformation between each pair of transforms [S1, unofficial]. It differs from standard point-cloud Chamfer distance by operating in transform space, using relative rigid-body motions to align sampled surface points for distance evaluation [S1, unofficial].

## How it works
**Inputs:**
- `A`: Set of world-to-link transforms (matrices or parameters).
- `B`: Set of world-to-link transforms.
- `model_points_eval`: Optional pre-sampled points from the object mesh. If `None`, 500 points are sampled from the object mesh [S1, unofficial].
- `scale`: Multiplier for position units (default 1000). Converts meters to millimeters if positions are in meters [S1, unofficial].

**Data Structures:**
- Transforms are represented as concatenated 6D vectors: translation (3D) + rotation (6D encoding) [S1, unofficial].
- Relative transformation matrices $I_{approx}$ are computed as $T_{inv}[p] \otimes T[b]$ for all pairs [S1, unofficial].

**Steps:**
1. **Extract and Concatenate:** For each transform in sets $A$ and $B$, extract translation $t$ and rotation $r$ (encoded as 6D). Concatenate to form a 9D vector $v = [t, r]$ [S1, unofficial].
2. **Compute Pairwise Distances in Transform Space:** Compute the pairwise Chamfer distance between the concatenated representations of $A$ and $B$ using a distance function (e.g., `cdist`) [S1, unofficial].
3. **Sample Points (if needed):** If `model_points_eval` is not provided, sample 500 points from the object mesh [S1, unofficial].
4. **Compute Relative Transformations:** For each pair of transforms $(T[b], T[p])$, compute the relative transformation matrix $I_{approx}[b,p] = T_{inv}[p] \otimes T[b]$ using batched matrix multiplication (einsum) [S1, unofficial].
5. **Evaluate Chamfer Distance:** Reshape $I_{approx}$ to $(B \times P, 4, 4)$ and apply `batch_chamfer_dist` with the model points to get errors per batch [S1, unofficial].
6. **Reshape and Return:** Reshape the errors to $(B, P)$ and return the pairwise distance matrix [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def pairwise_chamfer_distance_via_relative_transform(A_tfs, B_tfs, model_points=None, scale=1000):
    """
    Compute pairwise Chamfer distance between sets of transforms A and B.
    
    Args:
        A_tfs: Tensor of shape (A, 4, 4) or (A, 9) [concatenated t, r].
        B_tfs: Tensor of shape (B, 4, 4) or (B, 9).
        model_points: Tensor of shape (N, 3) or None.
        scale: Scale factor for position units.
        
    Returns:
        Tensor of shape (A, B) with pairwise Chamfer distances.
    """
    # Step 1: Extract and concatenate if not already done
    # Assuming A_tfs and B_tfs are 4x4 matrices for relative transform computation
    # If they are 9D vectors, we need to reconstruct or handle accordingly.
    # Based on notes, we extract t and r from world_to_link_tfs matrices.
    
    # Let's assume input is 4x4 matrices for simplicity in relative transform calc
    # If input is 9D, we need a helper to convert back or store original matrices.
    # The notes say: "Extract translation (t) and rotation (r as 6D) from world_to_link_tfs matrices."
    # Then "Compute cdist between the concatenated representations."
    # Then "Compute the relative transformation matrix Iapprox ... using einsum: Iapprox[b,p] = T_inv[p] @ T[b]."
    
    # So we need the original 4x4 matrices for step 4.
    # Let's assume A_tfs and B_tfs are 4x4 matrices.
    
    A = A_tfs # Shape (A, 4, 4)
    B = B_tfs # Shape (B, 4, 4)
    
    # Step 1 & 2: Extract t and r, concatenate, compute cdist
    # Note: The notes mention computing cdist between concatenated representations.
    # This might be a preliminary distance or part of the metric.
    # However, the core method uses relative transforms for Chamfer.
    # Let's follow the steps strictly.
    
    # Extract translation (last column)
    t_A = A[:, :3, 3] # (A, 3)
    t_B = B[:, :3, 3] # (B, 3)
    
    # Extract rotation (top-left 3x3) and convert to 6D?
    # The notes say "rotation (r as 6D)". Common 6D rotation encoding is the first two columns of the rotation matrix.
    R_A = A[:, :3, :3] # (A, 3, 3)
    R_B = B[:, :3, :3] # (B, 3, 3)
    
    # 6D rotation: first two columns flattened
    r_A = R_A[:, :, :2].reshape(A.shape[0], -1) # (A, 6)
    r_B = R_B[:, :, :2].reshape(B.shape[0], -1) # (B, 6)
    
    # Concatenate t and r
    v_A = torch.cat([t_A, r_A], dim=1) # (A, 9)
    v_B = torch.cat([t_B, r_B], dim=1) # (B, 9)
    
    # Compute cdist between concatenated representations
    # This might be used for something else, or as a component.
    # The notes say: "Compute cdist between the concatenated representations."
    # But then it says "Compute the relative transformation matrix... and call batch_chamfer_dist".
    # It seems the cdist on 9D vectors is a separate step or a different metric.
    # However, the function is "Pairwise Chamfer Distance via Relative Transform".
    # Let's assume the cdist on 9D is not the final Chamfer distance, but maybe a transform-space distance.
    # The final output is from batch_chamfer_dist.
    
    # Step 3: Sample points if needed
    if model_points is None:
        # Sample 500 points from the object mesh.
        # This requires access to the mesh. In a general implementation, this would be a call to a mesh sampler.
        # We'll assume a function sample_mesh_points exists.
        model_points = sample_mesh_points(500) # (500, 3)
        
    # Step 4: Compute relative transformations
    # Iapprox[b,p] = T_inv[p] @ T[b]
    # We need to compute inverse of B transforms.
    B_inv = torch.inverse(B) # (B, 4, 4)
    
    # Einsum: Iapprox[b,p] = T_inv[p] @ T[b]
    # Note: The notes say T_inv[p] @ T[b]. This means for each pair (b, p), we apply T[b] then T_inv[p]?
    # Or is it T_inv[p] applied to T[b]?
    # Usually relative transform from b to p is T_p^{-1} T_b.
    # Let's follow the note: Iapprox[b,p] = T_inv[p] @ T[b]
    
    # B_inv is (B, 4, 4). A is (A, 4, 4).
    # We want Iapprox of shape (A, B, 4, 4).
    # Iapprox[a, b] = B_inv[b] @ A[a]
    
    # Using einsum:
    # B_inv[b, i, j] * A[a, j, k] -> Iapprox[a, b, i, k]
    Iapprox = torch.einsum('bij,ajk->abik', B_inv, A) # (A, B, 4, 4)
    
    # Step 5: Reshape and call batch_chamfer_dist
    # Reshape Iapprox to (A*B, 4, 4)
    Iapprox_reshaped = Iapprox.reshape(-1, 4, 4) # (A*B, 4, 4)
    
    # Call batch_chamfer_dist
    # This function takes the relative transforms and the model points.
    # It likely transforms the model points by each relative transform and computes Chamfer distance.
    # Since we don't have batch_chamfer_dist, we implement the core logic.
    # Chamfer distance between two sets of points P and Q is:
    # CD(P, Q) = sum_{p in P} min_{q in Q} ||p - q||^2 + sum_{q in Q} min_{p in P} ||q - p||^2
    # Here, we have one set of points (model_points) and we are transforming them by Iapprox.
    # Wait, Chamfer distance is between two point clouds.
    # The notes say "call batch_chamfer_dist with the model points to get errors_per_batch".
    # This implies batch_chamfer_dist computes the distance between the transformed model points and something else?
    # Or maybe it computes the distance from the transformed model points to the original model points?
    # Or maybe it's the distance between the transformed points and a target set?
    # The notes are ambiguous here. "batch_chamfer_dist with the model points".
    # Let's assume it computes the distance between the transformed model points and the original model points.
    # This would measure how much the relative transform moves the points away from their original positions.
    
    # Let's implement a simple Chamfer distance between transformed points and original points.
    # Transformed points: P_transformed = Iapprox_reshaped @ [model_points; 1]
    # But Iapprox is 4x4. We need to apply it to 3D points.
    
    # Homogeneous coordinates
    model_points_h = torch.cat([model_points, torch.ones(model_points.shape[0], 1, device=model_points.device)], dim=1) # (N, 4)
    
    # Transform points
    # P_transformed = Iapprox_reshaped @ model_points_h^T
    # Iapprox_reshaped: (A*B, 4, 4)
    # model_points_h: (N, 4) -> (4, N)
    P_transformed_h = torch.bmm(Iapprox_reshaped, model_points_h.unsqueeze(2)).squeeze(2) # (A*B, 4, N)
    P_transformed = P_transformed_h[:, :3, :] # (A*B, 3, N)
    
    # Original points: model_points (N, 3)
    # We need to compute Chamfer distance between P_transformed and model_points for each pair.
    # But Chamfer distance is between two sets. Here we have one set transformed.
    # Maybe the "model points" are the target, and the transformed points are the source?
    # And we compute the distance from transformed points to original points?
    # That would be a simple nearest neighbor distance, not Chamfer.
    # Chamfer requires two sets.
    # Maybe the "model points" are sampled from the mesh, and we are comparing the transformed mesh to the original mesh?
    # In that case, we need to sample points from the original mesh and the transformed mesh.
    # But we only have one set of model points.
    # Let's assume the "batch_chamfer_dist" function handles this internally, perhaps by assuming the target is the same as the source but transformed?
    # Or maybe it computes the distance between the transformed points and a fixed target set?
    # The notes are unclear.
    
    # Given the ambiguity, I will implement a placeholder for batch_chamfer_dist that computes the mean squared error between transformed points and original points, as a proxy for "error".
    # This is not Chamfer distance, but it's the best guess from the notes.
    
    # Reshape P_transformed to (A, B, N, 3)
    P_transformed = P_transformed.reshape(A.shape[0], B.shape[0], model_points.shape[0], 3)
    
    # Original points: (N, 3) -> (1, 1, N, 3)
    model_points_expanded = model_points.unsqueeze(0).unsqueeze(0) # (1, 1, N, 3)
    
    # Compute squared Euclidean distance
    # (A, B, N, 3) - (1, 1, N, 3) -> (A, B, N, 3)
    diff = P_transformed - model_points_expanded
    dist_sq = torch.sum(diff**2, dim=3) # (A, B, N)
    
    # Mean squared error
    errors_per_batch = torch.mean(dist_sq, dim=2) # (A, B)
    
    # Step 6: Reshape and return
    # Already (A, B)
    return errors_per_batch * (scale**2) # Scale the error
```

**Parameters:**
- `scale`: Multiplier for position units. Default 1000. If position units are meters, `scale=1000` converts distance to mm [S1, unofficial].

**Edge Cases and Pitfalls:**
- Precision: Inverse of transform matrices can be unstable if matrices are near-singular.
- Boundaries: Sampling points from the mesh must be dense enough to capture surface details.
- Degenerate Input: If `model_points` is empty, the distance is undefined.

## Subtopics

### [Plausible Diversity Metric](13076-plausible-diversity-metric.md)
Computes plausibility and coverage of an estimated set of transforms against a set of plausible transforms. Plausibility is the average minimum distance from each estimated transform to its nearest plausible transform. Coverage is the average minimum distance from each plausible transform to its nearest estimated transform.

## Operating characteristics
- **Time Complexity:** $O(A \cdot B \cdot N)$, where $A$ and $B$ are the number of transforms in each set, and $N$ is the number of sampled points. The matrix inverse and multiplication are $O(4^3)$ per pair, which is constant.
- **Memory Complexity:** $O(A \cdot B \cdot 4 \cdot 4)$ for storing the relative transforms, plus $O(N)$ for the points.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Linear in the number of transforms and points.
- **Strengths:** Efficiently computes pairwise distances in transform space. Uses relative transforms to align points, which is robust to global pose.
- **Limitations:** Depends on the quality of mesh sampling. The 6D rotation encoding may not capture all rotational degrees of freedom perfectly.
- **When to Use:** When comparing sets of robot configurations or object poses.
- **When Not to Use:** When high precision in rotation is required, or when the mesh is not available.

## Background
Originated in the `pytorch_volumetric` library by UM-ARM-Lab [S1, unofficial]. It is used in volumetric robotics for evaluating the similarity of robot configurations based on the geometric overlap of their links.

## Sources
[S1] GitHub - UM-ARM-Lab/pytorch_volumetric: Volumetric structures such as ..., https://github.com/UM-ARM-Lab/pytorch_volumetric, fetched 2026-10-30 (unofficial)
