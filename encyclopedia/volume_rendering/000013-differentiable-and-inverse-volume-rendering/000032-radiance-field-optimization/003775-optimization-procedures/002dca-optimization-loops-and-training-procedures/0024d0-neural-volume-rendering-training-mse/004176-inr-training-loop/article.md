# INR Training Loop
## Overview
The Implicit Neural Representation (INR) training loop optimizes a neural network to approximate a discrete volumetric dataset by minimizing the mean squared error between network predictions and ground-truth values sampled from the volume. It differs from explicit volume rendering methods by learning a continuous function $f_\theta$ rather than storing discrete voxels, enabling resolution-independent querying [S1, unofficial].
## How it works
**Inputs and Outputs:**
*   **Input:** A discrete 3D volume $V$ of size $N \times N \times N$; a neural network $f_\theta$ with parameters $\theta$; an encoding function $\gamma$ (positional or hash); learning rate $\eta$; batch size $B$.
*   **Output:** Trained network parameters $\theta$ representing $V$ as a continuous function.

**Data Structures:**
*   **Volume $V$:** Stored as a 3D array of size $N^3$.
*   **Network $f_\theta$:** A multi-layer perceptron (MLP) or similar architecture.
*   **Encoding $\gamma$:** Maps 3D coordinates to a higher-dimensional latent space.

**Steps:**
1.  **Sampling:** For each training iteration, sample $B$ coordinates $\{(x_j, y_j, z_j)\}_{j=1}^B$ uniformly from the volume domain.
2.  **Encoding:** For each coordinate $(x_j, y_j, z_j)$, compute the encoded representation $e_j = \gamma(x_j, y_j, z_j)$.
3.  **Prediction:** Pass the encoded coordinate through the network to obtain the predicted value $\hat{v}_j = f_\theta(e_j)$.
4.  **Ground Truth Retrieval:** Retrieve the ground truth value $v_j$ from the discrete volume $V$ at $(x_j, y_j, z_j)$ using trilinear interpolation: $v_j = \text{trilinear}(V, x_j, y_j, z_j)$.
5.  **Loss Calculation:** Compute the Mean Squared Error (MSE) loss over the batch:
    $$ L = \frac{1}{B} \sum_{j=1}^{B} (\hat{v}_j - v_j)^2 $$
6.  **Update:** Perform a gradient descent step to update the network parameters:
    $$ \theta \leftarrow \theta - \eta \nabla_\theta L $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_inr(volume_V, network_f_theta, encoding_gamma, learning_rate_eta, batch_size_B, num_iterations):
    """
    Trains an Implicit Neural Representation to approximate a discrete volume.
    
    Args:
        volume_V: 3D numpy array of shape (N, N, N) containing ground truth values.
        network_f_theta: Neural network function taking encoded coordinates and returning scalar predictions.
        encoding_gamma: Function mapping 3D coordinates to encoded vectors.
        learning_rate_eta: Scalar learning rate for gradient descent.
        batch_size_B: Number of coordinates to sample per iteration.
        num_iterations: Total number of training steps.
    """
    # Assume network_f_theta has a method to compute gradients and update weights
    # or we use a standard optimizer loop. Here we assume a generic update step.
    
    for iteration in range(num_iterations):
        # 1. Sample B coordinates uniformly from the volume domain [0, N-1] or [0, 1]
        # Assuming coordinates are in continuous space [0, 1] for trilinear interpolation
        coords = np.random.uniform(0, 1, size=(batch_size_B, 3))
        
        losses = []
        gradients = []
        
        for j in range(batch_size_B):
            x, y, z = coords[j]
            
            # 2. Encode coordinates
            e_j = encoding_gamma(x, y, z)
            
            # 3. Network prediction
            v_hat_j = network_f_theta(e_j)
            
            # 4. Ground truth via trilinear interpolation
            # Note: Trilinear interpolation requires the volume to be defined on a grid.
            # If volume_V is size N x N x N, coordinates should be scaled to [0, N-1]
            # or the interpolation function handles [0,1] mapping.
            v_j = trilinear_interpolate(volume_V, x, y, z)
            
            # 5. Compute loss for this sample (for gradient accumulation or batch mean)
            loss_j = (v_hat_j - v_j) ** 2
            losses.append(loss_j)
            
            # Compute gradients for this sample (conceptually, usually done via backprop on the batch)
            # In practice, we compute the batch loss and backprop once.
        
        # Compute batch loss
        batch_loss = np.mean(losses)
        
        # 6. Gradient update
        # Compute gradients of loss with respect to theta
        grad_theta = compute_gradients(network_f_theta, batch_loss)
        
        # Update parameters
        network_f_theta.update_parameters(grad_theta, learning_rate_eta)

def trilinear_interpolate(volume, x, y, z):
    """
    Performs trilinear interpolation on a 3D volume.
    Assumes volume is a numpy array of shape (N, N, N).
    Coordinates x, y, z are expected in the range [0, N-1] or [0, 1] depending on implementation.
    This is a placeholder for the actual interpolation logic.
    """
    # Implementation details depend on the specific library or custom code.
    # Standard trilinear interpolation involves finding the 8 surrounding voxels
    # and weighting them based on the fractional part of the coordinates.
    # Not given in the sources. [G]
    pass
```

**Parameters:**
*   **Batch Size ($B$):** Not given in the sources.
*   **Learning Rate ($\eta$):** Not given in the sources.
*   **Encoding ($\gamma$):** Positional encoding or hash encoding. Hash encodings replace position with multi-resolution hash grids [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Trilinear interpolation is used to retrieve ground truth from discrete volumes, which may introduce smoothing artifacts compared to nearest-neighbor, but is standard for continuous approximation [S1, unofficial].
*   **Boundaries:** Sampling is uniform from the volume domain. Boundary handling depends on the interpolation function.
*   **Degenerate Input:** None specified.

## Subtopics

### Meta-INR
A method reducing training cost for time-varying INRs by applying Model-Agnostic Meta-Learning (MAML) to pretrain a shared parameter set on a spatiotemporally subsampled subset, followed by per-volume finetuning [S1, unofficial].
*   **Definition:** Pretrain a shared parameter set on a spatiotemporally subsampled subset (<1% of original data) using MAML, then perform per-volume finetuning from this initialization [S1, unofficial].
*   **Performance:** Converged 5.87× faster on average than training a SIREN from scratch, with higher reconstruction quality (55.92 dB vs. 48.89 dB on the half-cylinder dataset) [S1, unofficial].
*   **Pitfall:** Validated only with MLP-based SIREN backbones; transfer to hash-grid or feature-grid encoders is untested [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Training a fresh INR for each timestep scales linearly with volume count [S1, unofficial]. Meta-INR reduces this cost significantly.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Enables resolution-independent querying; Meta-INR offers faster convergence for time-varying data [S1, unofficial].
*   **Limitations:** Meta-INR is limited to SIREN backbones in current validation; hash-grid transfer is untested [S1, unofficial].
*   **When to use:** When continuous representation is needed or for time-varying data where Meta-INR can be applied.
*   **When not:** When hash-grid/feature-grid encoders are required for Meta-INR (currently untested) [S1, unofficial].

## Background
The INR training loop is a fundamental procedure for optimizing implicit neural representations of volumetric data. It builds on the concept of representing continuous functions with neural networks. Meta-INR extends this to time-varying data using MAML [S1, unofficial]. HyperINR is another variant that predicts INR weights from scene parameters via a hypernetwork, avoiding per-volume optimization entirely [S1, unofficial].

## Sources
[S1] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
