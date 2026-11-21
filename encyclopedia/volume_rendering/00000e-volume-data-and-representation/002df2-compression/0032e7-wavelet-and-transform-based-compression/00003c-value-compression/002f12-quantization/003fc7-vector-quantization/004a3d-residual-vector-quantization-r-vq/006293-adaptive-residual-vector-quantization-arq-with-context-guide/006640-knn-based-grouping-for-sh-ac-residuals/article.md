# KNN-based Grouping for SH-AC Residuals

Also called: SH-AC Residual Grouping

## Overview
This method clusters the 45 dimensions of Spherical Harmonic (SH) Anchor Coefficient (SH-AC) residuals into non-overlapping groups based on the similarity of their residual patterns. It operates on the residuals produced by the first-stage Vector Quantization (VQ) rather than the original coefficients, preparing the data for the second-stage VQ where separate codebooks are trained for each group [S1]. This grouping enables efficient entropy coding of indices by reducing the dimensionality of the residual sub-vectors [S1].

## How it works
**Inputs:**
- $N$: The number of 3D Gaussians in the model.
- $D=45$: The total number of SH-AC dimensions.
- $G$: The target number of groups (must be a divisor of 45). Valid configurations are $G \in \{1, 3, 5, 9\}$, resulting in sub-vector dimensions of $45, 15, 9, 5$ respectively [S1].

**Outputs:**
- A partition of the set of dimension indices $\{1, \dots, 45\}$ into $G$ disjoint sets $S_1, \dots, S_G$, where $|S_g| = 45/G$.

**Data Structures:**
- `residuals`: A matrix of shape $(N, 45)$, where `residuals[i, d]` is the residual of the $d$-th dimension for the $i$-th Gaussian after first-stage VQ.
- `distances`: A $45 \times 45$ symmetric matrix storing pairwise Euclidean distances between normalized dimension vectors.
- `groups`: A list of $G$ lists, each containing dimension indices.
- `assigned`: A boolean array of size 45, initialized to `False`.

**Steps:**

1. **Compute Residual Vectors:**
   For each dimension $d \in \{1, \dots, 45\}$, extract the vector of residuals across all Gaussians:
   $$ r_d = [r_{1,d}, r_{2,d}, \dots, r_{N,d}]^T $$
   where $r_{i,d}$ is the residual value for Gaussian $i$ and dimension $d$ [S1].

2. **Normalize Residual Vectors:**
   For each dimension $d$, compute the minimum and maximum residual values across all Gaussians:
   $$ r_{d,\min} = \min_{i} r_{i,d} $$
   $$ r_{d,\max} = \max_{i} r_{i,d} $$
   Apply min-max normalization to each element $r_{i,d}$ to obtain the normalized residual $\bar{r}_{i,d}$:
   $$ \bar{r}_{i,d} = \frac{r_{i,d} - r_{d,\min}}{r_{d,\max} - r_{d,\min} + \epsilon} $$
   where $\epsilon$ is a small constant to prevent division by zero [S1].
   Construct the normalized residual-dimension vector:
   $$ \bar{r}_d = [\bar{r}_{1,d}, \bar{r}_{2,d}, \dots, \bar{r}_{N,d}]^T $$

3. **Calculate Pairwise Distances:**
   Compute the Euclidean distance between every pair of normalized dimension vectors $d$ and $j$:
   $$ \delta(d,j) = \|\bar{r}_d - \bar{r}_j\|_2 $$
   Store these values in the `distances` matrix [S1].

4. **Greedy Grouping:**
   Initialize `groups` as an empty list and `assigned` as all `False`.
   While there are unassigned dimensions:
   a. Select an unassigned dimension $d_{ref}$ as the reference.
   b. Identify all unassigned dimensions $j$ (excluding $d_{ref}$).
   c. Sort these unassigned dimensions by their distance $\delta(d_{ref}, j)$ in ascending order.
   d. Select the nearest $k = (45/G) - 1$ unassigned dimensions to form a group with $d_{ref}$.
   e. Add $\{d_{ref}\} \cup \{ \text{selected dimensions} \}$ to `groups`.
   f. Mark all dimensions in this group as `assigned`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def knn_group_sh_ac_residuals(residuals, num_groups):
    """
    residuals: numpy array of shape (N, 45)
    num_groups: int, one of [1, 3, 5, 9]
    Returns: list of lists, each inner list contains dimension indices
    """
    N, D = residuals.shape
    assert D == 45
    assert 45 % num_groups == 0
    group_size = 45 // num_groups
    
    # Step 1 & 2: Normalize residual vectors
    # r_d is shape (N,) for each d
    r_min = np.min(residuals, axis=0)
    r_max = np.max(residuals, axis=0)
    epsilon = 1e-8
    
    # Normalize: (r - min) / (max - min + eps)
    # Handle case where max == min to avoid division by zero
    denom = r_max - r_min + epsilon
    normalized_residuals = (residuals - r_min) / denom
    
    # Step 3: Calculate pairwise distances
    # dist_matrix[d, j] = Euclidean distance between normalized_residuals[:, d] and normalized_residuals[:, j]
    dist_matrix = np.zeros((D, D))
    for d in range(D):
        for j in range(D):
            if d != j:
                dist_matrix[d, j] = np.linalg.norm(normalized_residuals[:, d] - normalized_residuals[:, j])
    
    # Step 4: Greedy Grouping
    assigned = [False] * D
    groups = []
    
    for _ in range(num_groups):
        # Find first unassigned dimension as reference
        ref_dim = -1
        for d in range(D):
            if not assigned[d]:
                ref_dim = d
                break
        
        if ref_dim == -1:
            break
            
        assigned[ref_dim] = True
        current_group = [ref_dim]
        
        # Find nearest unassigned dimensions
        candidates = []
        for j in range(D):
            if not assigned[j]:
                candidates.append((dist_matrix[ref_dim, j], j))
        
        # Sort by distance
        candidates.sort(key=lambda x: x[0])
        
        # Select remaining group_size - 1 dimensions
        for k in range(group_size - 1):
            if k < len(candidates):
                _, selected_dim = candidates[k]
                current_group.append(selected_dim)
                assigned[selected_dim] = True
        
        groups.append(current_group)
        
    return groups
```

**Parameters:**
- `num_groups`: The number of groups $G$. Typical values are 1, 3, 5, or 9 [S1].
- `codebook_size`: 1024 for both stages. This results in a 10-bit coarse index and a 10-bit residual index per sub-vector group [S1].

**Edge Cases and Pitfalls:**
- If $r_{d,\max} = r_{d,\min}$ for any dimension, the denominator becomes $\epsilon$. The normalized values will be 0, which is acceptable as it indicates no variation in residuals for that dimension [S1].
- The grouping is deterministic given the same residuals and tie-breaking rules (here, index order for equal distances).
- The method assumes residuals are available from a prior first-stage VQ step [S1].

## Subtopics

### [Two-Stage Vector Quantization (VQ) for SH-AC](19632-two-stage-vector-quantization-vq-for-sh-ac.md)
Applies VQ to the grouped residual sub-vectors using separate codebooks, enabling efficient entropy coding.

## Operating characteristics
- **Time Complexity:** The distance calculation is $O(D^2 \cdot N)$ where $D=45$ and $N$ is the number of Gaussians. The greedy grouping is $O(D^2 \log D)$ or $O(D^2)$ depending on implementation. Since $D$ is fixed and small, this is negligible compared to the VQ steps.
- **Memory Complexity:** Requires storing the $45 \times 45$ distance matrix and the normalized residuals. This is minimal.
- **Bitrate Control:** The SH-AC bitrate is controlled by varying the number of residual groups $G$. More groups mean smaller sub-vectors, which may allow for better quantization accuracy but increase the number of indices to encode [S1].
- **Strengths:** Adapts to the specific residual patterns of the trained model, potentially improving compression efficiency by grouping similar dimensions [S1].
- **Limitations:** Requires a two-stage process (first VQ, then grouping, then second VQ). The grouping is specific to the model and must be regenerated for each compressed 3DGS model [S1].

## Background
This method is part of a hybrid compression approach for trained 3D Gaussian Splatting models, combining Vector Quantization (VQ) and HEVC coding [S1]. It was introduced to improve the coding efficiency of SH-AC residuals by leveraging the similarity between different SH dimensions [S1].

## Sources
[S1] Hybrid Compression Method for Trained 3D Gaussian Splatting Models Based on VQ and HEVC., https://doi.org/10.3390/s26134125, fetched 2026-10-27
