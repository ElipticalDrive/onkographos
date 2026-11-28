# SDF Consistency Metrics

Also called: SDF Surface Error Metric, SDF accuracy evaluation, SDF error evaluation, mesh-to-SDF consistency, surface SDF error

## Overview
SDF Consistency Metrics evaluate how closely a reconstructed mesh matches the trained Signed Distance Function (SDF) field by measuring the network's SDF output at points on the mesh surface and at its vertices [S1, unofficial]. The method assesses the internal consistency of the implicit representation by computing statistical errors of the SDF values at these specific locations [S1, unofficial].

## How it works
### Inputs and Outputs
*   **Inputs**:
    *   `mesh`: A triangle mesh object containing vertices and normals [S1, unofficial].
    *   `net`: The trained SDF network capable of evaluating $SDF(\mathbf{p})$ [S1, unofficial].
    *   `training_data`: An object providing the scaling factor $R$ used during training [S1, unofficial].
    *   `n_samples`: Number of points to sample on the surface, default 2,097,152 [S1, unofficial].
*   **Outputs**: Dictionaries containing MAE, RMS, Max Absolute Error, and Standard Deviation of the SDF values [S1, unofficial].

### Data Structures
*   **Scaled Points**: Tensor of shape $(N, 3)$, dtype float32, representing points in the normalized training space [G].
*   **SDF Values**: Tensor of shape $(N)$, dtype float32, containing the network output $\hat{y}_i$ [S1, unofficial].

### Steps
1.  **Sampling**: For surface metrics, sample $N$ points uniformly on the mesh surface; normals are sampled but only positions are used for SDF evaluation [S1, unofficial]. For vertex metrics, use the mesh vertices directly [S1, unofficial].
2.  **Scaling**: Multiply all point coordinates by the training data scaling factor $R$ to match the network's input space [S1, unofficial].
3.  **Batching**: Split the points into batches of size `batch_size` (fixed at 32,768) to manage GPU memory [S1, unofficial].
4.  **Evaluation**: For each batch, query the network: $\hat{y}_i = \text{net.sdf}(\mathbf{p}_i)$, extract the first output channel if necessary, and concatenate results into a single tensor $\mathbf{\hat{y}}$ of size $N$ [S1, unofficial].
5.  **Metric Computation**: Calculate the four statistics defined below [S1, unofficial].

### Formulas
Let $N$ be the number of samples and $\hat{y}_i$ be the SDF value at the $i$-th sample point [S1, unofficial].

*   **Mean Absolute Error (MAE)**: $\text{MAE} = \frac{1}{N} \sum_{i=1}^N | \hat{y}_i |$ [S1, unofficial].
*   **Root Mean Square (RMS)**: $\text{RMS} = \sqrt{ \frac{1}{N} \sum_{i=1}^N \hat{y}_i^2 }$ [S1, unofficial].
*   **Max Absolute Error**: $\text{MaxAbs} = \max_{i} | \hat{y}_i |$ [S1, unofficial].
*   **Standard Deviation (Std)**: $\text{Std} = \sqrt{ \frac{1}{N} \sum_{i=1}^N (\hat{y}_i - \mu)^2 }$ where $\mu = \frac{1}{N} \sum_{i=1}^N \hat{y}_i$ is the mean of the raw SDF values [S1, unofficial].

### Parameters
*   `batch_size`: 32,768 [S1, unofficial].
*   `n_samples`: 2,097,152 for surface metrics [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def compute_sdf_consistency(points, net, batch_size=32768):
    """
    Compute SDF consistency metrics for a set of points against a trained network.
    
    Args:
        points: Tensor of shape (N, 3), dtype float32, scaled by training_data.R
        net: Trained SDF network (PyTorch model)
        batch_size: Inference batch size, fixed at 32768
        
    Returns:
        dict: MAE, RMS, MaxAbs, Std of SDF values
    """
    all_sdf_values = []
    num_points = points.shape[0]
    
    with torch.no_grad():
        for i in range(0, num_points, batch_size):
            end_idx = min(i + batch_size, num_points)
            batch_points = points[i:end_idx]
            batch_sdf = net.sdf(batch_points)[:, 0]
            all_sdf_values.append(batch_sdf.cpu())
            
    all_sdf_values = torch.cat(all_sdf_values, dim=0)
    
    abs_sdf = torch.abs(all_sdf_values)
    mae = torch.mean(abs_sdf).item()
    rms = torch.sqrt(torch.mean(all_sdf_values ** 2)).item()
    max_abs = torch.max(abs_sdf).item()
    std = torch.std(all_sdf_values).item()
    
    return {
        "mae_sdf": mae,
        "rms_sdf": rms,
        "max_abs_sdf": max_abs,
        "std_sdf": std
    }
```

### Edge Cases and Pitfalls
*   **Memory Overflow**: Large meshes with high `n_samples` may exceed GPU memory, necessitating the fixed batch size [S1, unofficial].
*   **Scaling Mismatch**: If the mesh is not scaled by $R$ correctly, the SDF values will be incorrect, leading to misleading error metrics [S1, unofficial].

## Subtopics

### Surface SDF Error
Evaluates SDF at uniformly sampled points on the mesh surface [S1, unofficial].

### Vertex SDF Error
Evaluates SDF at mesh vertices directly [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the number of samples, dominated by network inference [G].
*   **Memory Complexity**: $O(N)$ to store SDF values, reduced to $O(\text{batch\_size})$ peak usage via batching [S1, unofficial].
*   **Performance**: Evaluating 2 million points with a batch size of 32,768 takes a few seconds on standard GPUs depending on network depth [G].
*   **Strengths**: Directly measures the quality of the implicit representation independent of ground truth geometry [S1, unofficial].
*   **Limitations**: Does not measure geometric accuracy against a reference mesh and is sensitive to network scaling [G].

## Background
SDF Consistency Metrics were introduced in TetraSDF for 3D shape reconstruction using tetrahedral grids [S1, unofficial]. The metric evaluates the fidelity of the extracted mesh to the learned SDF field [S1, unofficial]. It is part of an evaluation pipeline that includes angular error metrics [S1, unofficial].

## Sources
[S1] naver-ai/TetraSDF, https://github.com/naver-ai/TetraSDF, fetched 2026-10-27 (unofficial)
