# Neural Volume Rendering Training (MSE)


## Overview
Neural Volume Rendering Training (MSE) is the foundational optimization procedure for training Implicit Neural Representations (INRs) of volumetric data. It solves the problem of learning a continuous function that approximates a discrete volume by minimizing the Mean Squared Error (MSE) between the network's predictions and ground-truth samples. It sets itself apart from sibling methods like [NeRF Training Loop](970-nerf-training-loop.md) by operating directly on volumetric scalar fields (e.g., density, attenuation) rather than radiance fields, and from [Photometric Optimization](5790-photometric-optimization.md) by using pixel-space supervision rather than volume-space supervision.

## How it works
**Inputs and Outputs:**
*   **Input:** A discrete volume $V$ of size $N^3$, a neural network $f_\theta$ with parameters $\theta$, an encoding function $\gamma$, a learning rate $\eta$, and a batch size $B$.
*   **Output:** A trained network $f_\theta$ representing $V$ as a continuous function.

**Data Structures:**
*   **Volume $V$:** A 3D grid of scalar values.
*   **Network $f_\theta$:** A multilayer perceptron (MLP) or similar architecture taking encoded coordinates as input and outputting a scalar value.
*   **Encoding $\gamma$:** A function mapping 3D coordinates to a higher-dimensional feature vector (e.g., positional encoding or hash encoding).

**Numbered Steps:**
1.  For each training iteration:
2.  Sample $B$ coordinates $\{(x_j, y_j, z_j)\}_{j=1}^B$ uniformly from the volume domain.
3.  For each coordinate $(x_j, y_j, z_j)$:
    a. Compute the encoded feature vector: $e_j \leftarrow \gamma(x_j, y_j, z_j)$.
    b. Compute the network prediction: $\hat{v}_j \leftarrow f_\theta(e_j)$.
    c. Retrieve the ground-truth value: $v_j \leftarrow \text{trilinear}(V, x_j, y_j, z_j)$.
4.  Compute the MSE loss: $L \leftarrow \frac{1}{B} \sum_{j=1}^B (\hat{v}_j - v_j)^2$.
5.  Update the network parameters: $\theta \leftarrow \theta - \eta \nabla_\theta L$.

**Formulas:**
The loss function is defined as:
$$
L = \frac{1}{B} \sum_{j=1}^B (\hat{v}_j - v_j)^2
$$
where $\hat{v}_j$ is the predicted value and $v_j$ is the ground-truth value obtained via trilinear interpolation from $V$.

**Parameters:**
*   **Batch Size ($B$):** Not given in the sources.
*   **Learning Rate ($\eta$):** Not given in the sources.
*   **Encoding ($\gamma$):** Positional or hash encoding [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Interpolation:** Ground truth values are retrieved using trilinear interpolation, which assumes the volume is defined on a regular grid [S2, unofficial].
*   **Sampling:** Uniform sampling from the volume domain is used, which may not capture rare features if the distribution is non-uniform [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_inr(volume, network, encoding, learning_rate, batch_size, num_iterations):
    """
    Trains an Implicit Neural Representation (INR) for volumetric data.
    
    Args:
        volume (ndarray): Discrete volume of size N^3.
        network (nn.Module): Neural network f_theta.
        encoding (callable): Encoding function gamma.
        learning_rate (float): Step size for optimization.
        batch_size (int): Number of samples per iteration.
        num_iterations (int): Total number of training iterations.
        
    Returns:
        nn.Module: Trained network f_theta.
    """
    optimizer = torch.optim.Adam(network.parameters(), lr=learning_rate)
    
    for _ in range(num_iterations):
        # Sample B coordinates uniformly from the volume domain
        # Assuming volume is defined in [0, N-1]^3 or normalized [0,1]^3
        # Adjust sampling range based on volume definition
        coords = torch.rand(batch_size, 3) * volume.shape[0]
        
        # Encode coordinates
        encoded_coords = encoding(coords)
        
        # Network prediction
        predictions = network(encoded_coords)
        
        # Ground truth via trilinear interpolation
        # Note: Trilinear interpolation requires coordinates to be within valid bounds
        # and typically assumes a grid structure.
        ground_truth = trilinear_interpolate(volume, coords)
        
        # Compute MSE loss
        loss = torch.mean((predictions - ground_truth) ** 2)
        
        # Gradient update
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
    return network

def trilinear_interpolate(volume, coords):
    """
    Performs trilinear interpolation on a 3D volume.
    
    Args:
        volume (ndarray): 3D array of shape (N, N, N).
        coords (Tensor): Tensor of shape (B, 3) with coordinates.
        
    Returns:
        Tensor: Interpolated values of shape (B,).
    """
    # Implementation details for trilinear interpolation
    # This is a standard operation and assumed to be available
    # or implemented as a helper function.
    # For the purpose of this pseudocode, we assume a standard
    # trilinear interpolation function is available.
    return torch.nn.functional.grid_sample(
        volume.unsqueeze(0).unsqueeze(0), # Add batch and channel dims
        coords.unsqueeze(0).unsqueeze(0), # Add batch and channel dims
        align_corners=True
    ).squeeze()
```

## Subtopics

### Inverse Rendering for Texture/Attribute Refinement
This variant uses inverse rendering to optimize texture attributes (TexF) by propagating image-space supervision through a differentiable rendering process. It minimizes a photometric loss between UV reference images and TexF renderings to refine voxel attributes, reducing transfer and discretization errors [S1, unofficial].

### INR Training Loop
This is the standard loop for training INRs for volumetric data. It samples coordinates uniformly, encodes them, predicts values via the network, retrieves ground truth via trilinear interpolation, and updates parameters using MSE loss [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Simple to implement; directly minimizes the error in the volume domain.
*   **Limitations:** Requires uniform sampling; may not capture rare features well; relies on trilinear interpolation for ground truth.
*   **When to use:** When the goal is to learn a continuous representation of a scalar volume field.
*   **When not to use:** When the goal is to learn a radiance field for novel view synthesis (use [NeRF Training Loop](970-nerf-training-loop.md)) or when image-space supervision is preferred (use [Photometric Optimization](5790-photometric-optimization.md)).

## Background
This method is based on the general approach of training Implicit Neural Representations (INRs) for volumetric data, as described in [S2, unofficial]. It is also used in the context of refining surface-aligned texture fields (TexF) via inverse rendering, as described in [S1, unofficial].

## Sources
[S1] Beyond UV Mapping: Mesh Texture Compression via Surface-Aligned Texture Fields, https://doi.org/10.48550/arxiv.2609.23606, fetched 2026-10-26 (unofficial)
[S2] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
