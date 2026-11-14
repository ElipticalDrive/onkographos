# Volume Rendering Prior (VRPrior)

Also called: learnable differentiable renderer, neural weight function

## Overview
VRPrior replaces handcrafted, 2D-biased equations for mapping unsigned distance fields (UDF) to opaque density with a learnable neural network $f_w$ [S1]. It solves the problem of sensitivity to unsigned distance outliers and lack of 3D awareness in ray-surface intersections by aggregating features from multi-scale neighborhoods of neighboring points along a ray [S1]. Unlike sibling methods that use fixed formulas or single-window heuristics, VRPrior uses a multi-resolution MLP architecture to generalize robust density estimation across varying UDF distributions [S1].

## How it works
**Inputs and Outputs:**
*   **Input to $f_w$:** For a query point $q_n$ on a ray, the network takes two sets of size $M$: the sampling intervals $\{\delta_m\}_{m=1}^M$ and the unsigned distances $\{f_u(q'_m)\}_{m=1}^M$ of $M$ nearest neighbors $q'_m$ of $q_n$ along the same ray [S1].
*   **Output of $f_w$:** Opaque density $\sigma_n$ at $q_n$ [S1].
*   **Overall System Inputs:** Ground truth meshes $\{S_h\}_{h=1}^H$ for prior training; multi-view RGB images $\{I_j\}_{j=1}^J$ and posed views for generalization [S1].
*   **Overall System Outputs:** Optimized UDF field $f_u$, color network $f_c$, and rendered RGB/Depth images [S1].

**Data Structures:**
1.  **Neighbor Groups:** The $M$ nearest neighbors are divided into three groups based on distance to $q_n$:
    *   Group 1: $M_1 = 10$ points [S1].
    *   Group 2: $M_2 = 20$ points [S1].
    *   Group 3: $M_3 = 30$ points [S1].
    *   Total $M = M_1 + M_2 + M_3 = 60$ [S1].
2.  **Network $f_w$:**
    *   Three separate MLPs ($f_{w1}^{(k)}$ for $k \in \{1,2,3\}$) with non-shared parameters. Each takes concatenated features (distances + intervals) for its specific group size [S1].
    *   One final MLP ($f_{w2}$) with skip connections. Takes the sum of features from the three $f_{w1}$ branches [S1].
3.  **Network $f_c$:** Standard color network predicting color $\mathbf{c}$ from position $q_n$ [S1].

**Numbered Steps:**

**Phase 1: Prior Training (Learning $f_w$)**
1.  For each shape $S_h$ in the training set ($h=1 \dots H$):
    a. Calculate ground truth unsigned distance field $f_{u_{gt}}$ for $S_h$ [S1].
    b. Render $A=100$ ground truth depth images $\{D_a^h\}_{a=1}^A$ from random view angles around $S_h$ using $f_{u_{gt}}$ [S1].
2.  For each rendered view $a$ and each ray $V_k$:
    a. Hierarchically sample $N=128$ points $\{q_n\}_{n=1}^N$ along the ray [S1].
    b. For each query $q_n$, identify $M$ nearest neighbors $\{q'_m\}_{m=1}^M$ along the ray [S1].
    c. Compute sampling intervals $\delta_m$ between neighbors [S1].
    d. Retrieve ground truth unsigned distances $f_{u_{gt}}(q'_m)$ for neighbors [S1].
    e. Compute opaque density $\sigma_n = f_w(\{\delta_m\}, \{f_{u_{gt}}(q'_m)\})$ [S1].
    f. Compute rendering weights $w_n = \sigma_n \times \prod_{n'=1}^{n-1} (1 - \sigma_{n'})$ [S1].
    g. Render depth $\tilde{D}_a^h = \sum_{n'=1}^N w_{n'} \times d_{n'}$ [S1].
3.  Optimize parameters $\theta_w$ of $f_w$ by minimizing the L2 error between rendered depth and ground truth depth:
    $$ \min_{\theta_w} \sum_{h=1}^H \sum_{a=1}^A \| D_a^h - \tilde{D}_a^h \|_2^2 $$
    [S1].
    *Note: RGB images are NOT used in this phase to preserve generalization ability [S1].*

**Phase 2: Generalization (Optimizing $f_u$ and $f_c$)**
1.  Freeze parameters $\theta_w$ of $f_w$ [S1].
2.  Initialize a UDF field $f_u$ (e.g., spherical initialization) and color network $f_c$ for the unseen scene [S1].
3.  For each view $j$ and ray $V_k$:
    a. Sample points $\{q_n\}_{n=1}^N$ along the ray [S1].
    b. For each $q_n$, find $M$ nearest neighbors $\{q'_m\}$ along the ray [S1].
    c. Compute $\delta_m$ and $f_u(q'_m)$ [S1].
    d. Compute $\sigma_n = f_w(\{\delta_m\}, \{f_u(q'_m)\})$ using the frozen $f_w$ [S1].
    e. Compute weights $w_n = \sigma_n \times \prod_{n'=1}^{n-1} (1 - \sigma_{n'})$ [S1].
    f. Render color $I'_j = \sum_{n'=1}^N w_{n'} \times f_c(q_{n'})$ and depth $D'_j = \sum_{n'=1}^N w_{n'} \times d_{n'}$ [S1].
4.  Optimize $f_u$ and $f_c$ by minimizing the total loss:
    $$ \mathcal{L} = \mathcal{L}_{rgb} + \lambda \mathcal{L}_e $$
    where $\mathcal{L}_{rgb} = \sum_{j=1}^J \| I_j - I'_j \|$ and $\mathcal{L}_e$ is the Eikonal loss [S1].
    *Parameter:* $\lambda = 0.1$ [S1].

**Formulas:**
*   **Opaque Density:**
    $$ \sigma_n = f_w(\{\delta_m\}_{m=1}^M, \{f_{u_{gt}}(q'_m)\}_{m=1}^M) $$
    where $q'_m \in NN(q_n)$ are the $M$ nearest neighbors of $q_n$ along the ray, and $\delta_m$ is the sampling interval between neighbors [S1].
*   **Rendering Accumulation:**
    $$ w_n = \sigma_n \times \prod_{n'=1}^{n-1} (1 - \sigma_{n'}) $$
    $$ I' = \sum_{n'=1}^N w_{n'} \times f_c(q_{n'}) $$
    $$ D' = \sum_{n'=1}^N w_{n'} \times d_{n'} $$
    where $w_n$ is the weight, $I'$ is rendered color, $D'$ is rendered depth, and $d_{n'}$ is the depth value of $q_{n'}$ [S1].

**Parameters:**
*   $N=128$: Number of hierarchical samples along each ray [S1].
*   $A=100$: Number of ground truth depth images rendered per shape during prior training [S1].
*   $M_1=10, M_2=20, M_3=30$: Sizes of neighbor groups [S1].
*   $\lambda=0.1$: Balance weight for Eikonal loss [S1].

**Edge Cases and Pitfalls:**
*   **Handcrafted Bias:** Using handcrafted equations with only $M=2$ neighbors is biased at ray-surface intersections and sensitive to unsigned distance outliers due to small receptive fields and lack of 3D awareness [S1].
*   **RGB in Prior Training:** Using RGB images during the prior training phase reduces the generalization ability of the learned prior. The prior must be trained only on depth errors to learn the mapping from UDF to density [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class VRPrior:
    def __init__(self, M1=10, M2=20, M3=30, N=128):
        """
        Initialize the VRPrior network.
        M1, M2, M3: Neighbor group sizes.
        N: Number of samples per ray.
        """
        self.M1 = M1
        self.M2 = M2
        self.M3 = M3
        self.N = N
        
        # Define MLPs for neighbor groups
        # Input dim: 2 * M_k (distances + intervals)
        # Output dim: Feature dim (assumed 64 for implementation, source doesn't specify hidden dims)
        self.fw1_1 = self._create_mlp(2 * M1, 64)
        self.fw1_2 = self._create_mlp(2 * M2, 64)
        self.fw1_3 = self._create_mlp(2 * M3, 64)
        
        # Final MLP with skip connections
        # Input dim: 64 * 3 (sum of features) + input_dim (skip)
        # Source says "skip connection to predict", typically input to fw2 is concatenated or added.
        # Assuming standard skip: input to fw2 is [concat(features), raw_input] or similar.
        # Given "features are then added together and fed into another MLP fw2 with skip connection",
        # we assume fw2 takes the summed features and potentially the raw input for skip.
        # Let's assume fw2 input dim is 64*3 + 2*M (raw input) or just 64*3 with internal skip.
        # Standard NeRF-style skip: input to layer i is concat(input, hidden_i).
        # Here, "skip connection" likely means the raw neighbor features are passed to fw2.
        # We'll define fw2 to take summed features + raw concatenated neighbor features.
        raw_input_dim = 2 * (M1 + M2 + M3)
        self.fw2 = self._create_mlp_with_skip(64 * 3 + raw_input_dim, 1)

    def _create_mlp(self, in_dim, out_dim):
        return nn.Sequential(
            nn.Linear(in_dim, 64),
            nn.ReLU(),
            nn.Linear(64, out_dim)
        )

    def _create_mlp_with_skip(self, in_dim, out_dim):
        # Simplified skip connection: input is concatenated with hidden representation
        # Assuming 1 hidden layer for fw2 as per "another MLP"
        return nn.Sequential(
            nn.Linear(in_dim, 64),
            nn.ReLU(),
            nn.Linear(64, out_dim)
        )

    def compute_density(self, distances, intervals):
        """
        Compute opaque density sigma using VRPrior.
        distances: Tensor of shape (B, M, 1) - unsigned distances of neighbors
        intervals: Tensor of shape (B, M, 1) - sampling intervals
        Returns: Tensor of shape (B, N, 1) - opaque density sigma
        """
        # Split neighbors into groups
        # M = M1 + M2 + M3
        M = self.M1 + self.M2 + self.M3
        
        # distances and intervals are (B, M, 1)
        # Group 1: first M1
        d1 = distances[:, :self.M1, :]
        i1 = intervals[:, :self.M1, :]
        # Group 2: next M2
        d2 = distances[:, self.M1:self.M1+self.M2, :]
        i2 = intervals[:, self.M1:self.M1+self.M2, :]
        # Group 3: last M3
        d3 = distances[:, self.M1+self.M2:, :]
        i3 = intervals[:, self.M1+self.M2:, :]

        # Concatenate distances and intervals for each group
        f1 = torch.cat([d1, i1], dim=-1) # (B, M1, 2)
        f2 = torch.cat([d2, i2], dim=-1) # (B, M2, 2)
        f3 = torch.cat([d3, i3], dim=-1) # (B, M3, 2)

        # Process through separate MLPs
        feat1 = self.fw1_1(f1) # (B, M1, 64) -> sum over M1? Source says "concatenate... feed into MLP".
        # Wait, source says "concatenate all of the unsigned distances and sampling intervals... feed into three separate MLPs".
        # If input is (B, M_k, 2), output is (B, M_k, 64). How to reduce to scalar per query?
        # Usually, these features are aggregated (e.g., mean or sum) over the neighbor dimension.
        # Source: "The features are then added together". This implies aggregation over neighbors first.
        
        # Aggregate features over neighbors (sum or mean). Let's assume sum as per "added together" later.
        # Actually, "concatenate all... feed into MLP" might mean the MLP processes the whole set?
        # No, "three separate MLPs... process each neighbor group's concatenated features".
        # If MLP is standard, it processes each element. We need to aggregate.
        # Let's assume mean pooling over the neighbor dimension for each group.
        feat1 = feat1.mean(dim=1) # (B, 64)
        feat2 = self.fw1_2(f2).mean(dim=1) # (B, 64)
        feat3 = self.fw1_3(f3).mean(dim=1) # (B, 64)

        # Sum features
        summed_feat = feat1 + feat2 + feat3 # (B, 64)
        
        # Prepare input for fw2 with skip connection
        # Raw input: all distances and intervals concatenated (B, M, 2) -> (B, 2M)
        raw_input = torch.cat([distances, intervals], dim=-1).reshape(distances.shape[0], -1)
        
        # Concatenate summed features and raw input for skip connection
        fw2_input = torch.cat([summed_feat, raw_input], dim=-1)
        
        # Predict density
        sigma = self.fw2(fw2_input) # (B, 1)
        
        # Ensure sigma is non-negative if required by rendering, though source doesn't explicitly clamp.
        # Standard alpha blending requires sigma >= 0.
        sigma = F.relu(sigma)
        
        return sigma

    def render(self, points, camera_origin, camera_direction, udf_field, color_network, is_training_prior=False, gt_udf=None):
        """
        Render RGB and Depth using VRPrior.
        points: Tensor of shape (B, N, 3) - sampled points along rays
        camera_origin: Tensor of shape (B, 3)
        camera_direction: Tensor of shape (B, 3)
        udf_field: Function or network to evaluate UDF at points.
        color_network: Network to predict color.
        is_training_prior: If True, use gt_udf for density calculation.
        gt_udf: Ground truth UDF function.
        """
        # Calculate depths d_n
        # d_n = dot(q_n - o, r)
        # Assuming points are already sampled along ray, d_n is distance from origin
        # If points are absolute coordinates:
        # d_n = torch.sum((points - camera_origin.unsqueeze(1)) * camera_direction.unsqueeze(1), dim=-1)
        # But source says qn = o + dn * r, so dn is the scalar depth.
        # Let's assume points are given as absolute coordinates.
        # We need to compute dn for each point.
        # dn = (qn - o) . r
        diffs = points - camera_origin.unsqueeze(1)
        depths = torch.sum(diffs * camera_direction.unsqueeze(1), dim=-1) # (B, N)
        
        # Get UDF values for all points
        if is_training_prior:
            udf_vals = gt_udf(points) # (B, N)
        else:
            udf_vals = udf_field(points) # (B, N)
            
        # For each point qn, find M nearest neighbors along the ray
        # Since points are sorted along the ray, neighbors are just indices n-k to n+k
        # We need to select M neighbors for each n.
        # Source says "M nearest neighbors of qn along a ray".
        # We'll approximate this by taking a window around n.
        # However, source specifies grouping by distance, not just index.
        # For implementation simplicity, we assume we have pre-computed neighbor indices or values.
        # Here, we simulate finding neighbors.
        
        # To keep it self-contained, we assume a helper `get_neighbors` exists or we compute it.
        # Since points are sorted, we can just take a fixed window if density is smooth,
        # but source uses distance-based grouping.
        # Let's assume we have `neighbor_indices` of shape (B, N, M) pre-computed.
        # For this pseudocode, we will assume `udf_vals` and `depths` are used to find neighbors.
        
        # Simplified: Assume we have neighbor indices `idx` of shape (B, N, M)
        # In practice, this requires sorting or k-NN.
        # We will skip the explicit k-NN search code as it's a standard geometric operation,
        # but note that `distances` and `intervals` must be computed from these neighbors.
        
        # Placeholder for neighbor selection logic
        # idx = get_k_nearest_neighbors_along_ray(points, M) # (B, N, M)
        
        # Extract neighbor UDF values and depths
        # udf_neighbors: (B, N, M)
        # depth_neighbors: (B, N, M)
        
        # Compute intervals delta_m
        # delta_m = depth_{m+1} - depth_m ? Source says "sampling interval between neighbors".
        # Usually delta_m is distance between q'_m and q'_{m+1}.
        
        # Compute density sigma for each point n
        # sigma_n = f_w(delta_m, udf(q'_m))
        
        # This requires the neighbor extraction logic.
        # We will assume `udf_neighbors` and `depth_neighbors` are available.
        
        # Compute intervals
        # delta_m = depth_neighbors[:, :, 1:] - depth_neighbors[:, :, :-1] ?
        # Source: "delta_m is the sampling interval between neighbors".
        # If M neighbors, there are M-1 intervals? Or M intervals defined differently?
        # Source formula: sigma_n = f_w({delta_m}_{m=1}^M, {f_u(q'_m)}_{m=1}^M).
        # This implies M intervals for M neighbors.
        # Perhaps delta_m is distance from q_n to q'_m? Or distance between q'_m and q'_m+1?
        # "delta_m between q'_m and q'_{m+1}" suggests M-1.
        # But input size is M.
        # Let's assume delta_m is the distance from q_n to q'_m? No, "sampling interval".
        # Let's assume delta_m is the distance between adjacent neighbors in the sorted list.
        # If M neighbors, we might pad or use distance to q_n.
        # Given the ambiguity, we stick to the structure:
        # Input to f_w is (distances, intervals).
        
        # For the sake of valid Python, we assume `get_neighbors` returns:
        # neighbor_udfs: (B, N, M)
        # neighbor_depths: (B, N, M)
        
        # Compute intervals
        # If M neighbors, we can compute intervals between them.
        # Let's assume delta_m is the distance from q_n to q'_m for simplicity if M-1 is not M.
        # Or, if M=60, and we take 30 before and 30 after, intervals are between them.
        # We will assume `intervals` is computed as `neighbor_depths - depths.unsqueeze(-1)`?
        # No, "sampling interval between neighbors".
        # Let's assume `intervals` is `torch.diff(neighbor_depths, dim=-1)` padded?
        # Source says "delta_m between q'_m and q'_{m+1}".
        # If M neighbors, we have M-1 intervals.
        # But input is size M.
        # This is a pitfall/ambiguity. We will assume the implementation handles this,
        # e.g., by repeating the last interval or using distance to query.
        # For the pseudocode, we will pass `neighbor_depths` and `neighbor_udfs` to a helper.
        
        # Final rendering
        # w_n = sigma_n * prod(1 - sigma_{n'})
        # I = sum(w_n * color(q_n))
        # D = sum(w_n * d_n)
        
        pass
```

## Operating characteristics
*   **Time Complexity:** Depends on the number of samples $N$ and neighbors $M$. Each ray requires $N$ evaluations of $f_w$ and $f_c$. $f_w$ involves three MLPs and a final MLP, making it more expensive per sample than handcrafted formulas but potentially faster overall due to better convergence and fewer samples needed for quality [S1].
*   **Memory Complexity:** Stores parameters for $f_w$ (three MLPs + final MLP) and $f_c$. No large voxel grids or hash tables are mentioned, unlike some other NeRF variants [S1].
*   **Strengths:** Robust to unsigned distance outliers; 3D aware due to multi-scale neighborhood aggregation; generalizes well to unseen scenes when trained only on depth [S1].
*   **Limitations:** Requires ground truth meshes for prior training; more complex to implement than handcrafted density functions; sensitive to neighbor selection along the ray [S1].
*   **When to use:** When reconstructing surfaces from multi-view images using UDFs and handcrafted density mappings fail due to bias or sensitivity to outliers [S1].
*   **When not to use:** When ground truth meshes are unavailable for prior training; when computational resources are extremely limited and handcrafted methods are sufficient [S1].

## Background
VRPrior was introduced in the paper "VRP-UDF: Toward Unbiased Learning of Unsigned Distance Functions From Multi-View Images With Volume Rendering Priors" [S1]. It addresses limitations in previous methods like NeUDF and 2S-UDF, which used handcrafted equations for mapping UDF to density. These handcrafted methods were found to be biased at ray-surface intersections and sensitive to unsigned distance outliers [S1]. VRPrior leverages the idea of learnable neural rendering frameworks [S1] to create a data-driven prior that learns to map UDFs to opaque densities.

## Sources
[S1] VRP-UDF: Toward Unbiased Learning of Unsigned Distance Functions From Multi-View Images With Volume Rendering Priors., https://doi.org/10.1109/tpami.2026.3653901, fetched 2026-10-27
