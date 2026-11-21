# Hierarchical CT Reconstruction via Domain Transform

Also called: Hierarchical DL framework, Hierarchical DL reconstruction, Hierarchical Deep Learning Tomographic Reconstruction, Hierarchical Domain Transform Decomposition, Hierarchical decomposition of domain transforms, Partial line integral domain transform

## Overview
Hierarchical CT Reconstruction via Domain Transform is a deep learning framework that solves large-scale inverse problems, such as tomographic reconstruction, by decomposing the non-local, space-variant integral transform into a sequence of simpler, local hierarchical steps [S1]. Unlike brute-force fully connected networks that require intractable parameter counts for full-scale data, this method avoids dense connectivity by introducing virtual intermediate data representations (partial line integrals) [S1]. It sets itself apart from siblings like [Deep Image Prior Reconstruction](1221-deep-image-prior-reconstruction.md), [R2-Gaussian](6090-r2-gaussian.md), [End-to-End Reconstruction Model Architecture](7504-end-to-end-reconstruction-model-architecture.md), and [Transformer-based Architectures](13148-transformer-based-architectures.md) by explicitly constructing a homotopy-like continuum of data domains (from line integrals to partial line integrals to voxels) rather than relying on implicit neural priors or direct end-to-end mapping of the full sinogram to the image [S1].

## How it works
### Inputs and Outputs
*   **Input:** A sinogram $p(r, \theta)$ representing line integrals, with dimensions $N_r$ radial bins and $N_\theta$ projection angles. For the reference implementation, $N_r = 512$ and $N_\theta = 512$ [S1].
*   **Output:** A reconstructed image $f(x, y)$ (or $f(r, t)$ at $\theta=0$) with dimensions $N_x \times N_y$. For the reference implementation, $512 \times 512$ [S1].

### Data Structures
The method operates on a hierarchy of stages $L=0$ to $L=N$. Each stage $L$ holds data in the **partial line integral domain**, defined by three dimensions:
1.  **Depth bins ($N_t$):** Represents depth resolution along the projection ray [S1].
2.  **Projection angles ($N_\theta$):** The number of angular views [S1].
3.  **Radial bins ($N_r$):** The number of detector channels [S1].

The data tensor at stage $L$ has shape $(N_t^{(L)}, N_\theta^{(L)}, N_r^{(L)})$. The total data size is kept approximately constant across stages: $N_t^{(L)} \times N_\theta^{(L)} \times N_r^{(L)} \approx \text{Constant}$ [S1].

**Reference Dimension Progression (2D, $512^2$ output):**
*   $L=0$ (Input Sinogram): $(1, 512, 512)$
*   $L=1$: $(2, 256, 512)$
*   $L=2$: $(8, 64, 512)$
*   $L=3$: $(32, 16, 512)$
*   $L=4$: $(128, 4, 512)$
*   $L=5$ (Output Image): $(512, 1, 512)$ [S1]

The ratio of depth bins to projection angles between stages does not need to be a power of two or an integer, as the network is trained to perform resampling and interpolation of the data between hierarchical stages [S1].

### Formulas
The forward model is the Fredholm integral equation of the first kind [S1]:
$$
p(v) = \int K(u,v)f(u)du
$$
where $K(u,v) = \delta(u_x \cos v_\theta + u_y \sin v_\theta - v_r)$ is the Dirac delta kernel for parallel-beam geometry, $u=(u_x, u_y)$ are image coordinates, and $v=(v_r, v_\theta)$ are projection coordinates [S1].

The method introduces a virtual intermediate representation $q_\alpha(v, t)$ using a window function $\Pi(\cdot)$ [S1]:
$$
q_\alpha(v,t) = \int K(u,v)f(u)\Pi\left(\frac{t-u}{\alpha}\right)du
$$
where:
*   $\alpha$ is a scale factor controlling the window size [S1].
*   $t$ is a parameter specifying the location of the window (depth) [S1].
*   $\Pi(x) = 1$ for $x < 0.5$ and $0$ elsewhere [S1].

In the limit $\alpha \to \infty$, $q_\alpha(v,t) = p(v)$ (line integral domain). In the limit $\alpha \to 0$, the window becomes a Dirac delta impulse, yielding the voxel domain $f(u)$ [S1].

### Parameters
*   **$\alpha$ (Scale Factor):** Controls the width of the integration window. Decreases from $\infty$ (input) to $0$ (output) across stages [S1].
*   **$\Pi(x)$:** Rectangular window function [S1].
*   **Neighborhood Size:** For sparse connection layers, each input neuron connects to a small neighborhood of output neurons. Empirical size: $3 \text{ depths} \times 5 \text{ angles} \times 3 \text{ radial bins}$ [S1].
*   **Number of Stages ($L$):** On the order of $O(\log n)$, where $n$ is the data dimension ($512 \times 512$) [S1].

### Steps
1.  **Define Hierarchy:** Determine the sequence of data dimensions $(N_t, N_\theta, N_r)$ for stages $L=0$ to $L=N$. Ensure $N_\theta$ decreases and $N_t$ increases to transition from sinogram to image [S1].
2.  **Initialize Network:** Construct $L$ sparse connection layers. Initialize sparse connection weights to all ones. Initialize convolution kernels (if used) with the Glorot uniform initializer [S1].
3.  **Pre-training (Optional):** Train each sparse connection layer individually using synthetic noise data to initialize weights to a reasonable magnitude. This reduces total training time but does not affect the final loss function [S1].
4.  **End-to-End Training (Noise-Based):**
    *   Generate training pairs: Computer-generated white Gaussian random noise patterns $f_{noise}$ and their forward projections $p_{noise}$ using distance-driven projection [S1].
    *   Compute intermediate labels $q_{\alpha}^{(L)}$ for each stage using Filtered Back Projection (FBP) on the noise patterns onto non-isotropic voxel grids corresponding to stage $L$ dimensions [S1].
    *   Train the network to minimize Mean Squared Error (MSE) between the network output and the intermediate labels [S1].
    *   **Vanishing Gradient Handling:** During end-to-end training, update only one layer per iteration while freezing others. Select the layer randomly at each iteration [S1].
    *   **Linear Constraint:** Exclude non-linear activation units during this phase to constrain the network to learning a linear inverse operator, preventing it from learning noise appearance as a prior [S1].
5.  **End-to-End Training (Clinical Refinement):**
    *   Generate training pairs from clinical CT images: Reproject clinical images to create sinograms with added random noise [S1].
    *   Include non-linear activation units (e.g., ReLU) in convolutional layers to learn non-linear priors [S1].
    *   Train end-to-end using MSE loss [S1].
6.  **Reconstruction:** For a new sinogram, pass it through the trained network. The final stage $L=N$ outputs the reconstructed image $f(x,y)$ where $\theta=0$ and $\alpha$ corresponds to the voxel size [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

class SparseConnectionLayer:
    """
    Implements a sparse fully-connected layer where each input neuron 
    connects only to a small neighborhood of output neurons.
    Forward operator: product of input data and sparse matrix.
    Backpropagation operator: product of transpose of sparse matrix and gradient.
    """
    def __init__(self, input_shape, output_shape, neighborhood):
        self.input_shape = input_shape
        self.output_shape = output_shape
        self.neighborhood = neighborhood
        self.weights = None
        self._build_sparsity()

    def _build_sparsity(self):
        # Generate sparse indices based on neighborhood size
        # Each input neuron connects to a local neighborhood in the output grid
        rows, cols = [], []
        for i in range(np.prod(self.input_shape)):
            in_coords = np.unravel_index(i, self.input_shape)
            for dt in range(-self.neighborhood[0]//2, self.neighborhood[0]//2 + 1):
                for dtheta in range(-self.neighborhood[1]//2, self.neighborhood[1]//2 + 1):
                    for dr in range(-self.neighborhood[2]//2, self.neighborhood[2]//2 + 1):
                        out_coords = (in_coords[0] + dt, in_coords[1] + dtheta, in_coords[2] + dr)
                        if 0 <= out_coords[0] < self.output_shape[0] and \
                           0 <= out_coords[1] < self.output_shape[1] and \
                           0 <= out_coords[2] < self.output_shape[2]:
                            rows.append(np.ravel_multi_index(out_coords, self.output_shape))
                            cols.append(i)
        self.rows = np.array(rows)
        self.cols = np.array(cols)

    def forward(self, x):
        # x shape: (batch, Nt, Ntheta, Nr)
        batch_size = x.shape[0]
        flat_x = x.reshape(batch_size, -1)
        flat_out = np.zeros((batch_size, np.prod(self.output_shape)))
        # Apply sparse weights: out[row] += weights[col] * in[col]
        for b in range(batch_size):
            for idx in range(len(self.rows)):
                flat_out[b, self.rows[idx]] += self.weights[idx] * flat_x[b, self.cols[idx]]
        return flat_out.reshape(batch_size, *self.output_shape)

    def backward(self, grad_out):
        # grad_out shape: (batch, Nt_out, Ntheta_out, Nr_out)
        batch_size = grad_out.shape[0]
        flat_grad_out = grad_out.reshape(batch_size, -1)
        flat_grad_in = np.zeros((batch_size, np.prod(self.input_shape)))
        # Backprop: grad_in[col] += weights[col] * grad_out[row]
        for b in range(batch_size):
            for idx in range(len(self.rows)):
                flat_grad_in[b, self.cols[idx]] += self.weights[idx] * flat_grad_out[b, self.rows[idx]]
        return flat_grad_in.reshape(batch_size, *self.input_shape)

def create_hierarchical_network(stage_dims, neighborhood=(3, 5, 3)):
    """
    Creates the hierarchical DL network.
    stage_dims: List of tuples (Nt, Ntheta, Nr) for each stage L=0 to L=N.
    """
    layers = []
    for i in range(1, len(stage_dims)):
        input_dims = stage_dims[i-1]
        output_dims = stage_dims[i]
        layer = SparseConnectionLayer(input_dims, output_dims, neighborhood)
        layers.append(layer)
        
        # Optional: Add convolutional layers between stages
        # L1 is sinogram domain convolutional layer with filter kernel size 512
        # L2-L5 are image domain convolutional layers with 3x3 kernels and ReLU activation
        if i == 1:
            conv = np.zeros((1, 512, 1)) # Placeholder for sinogram conv kernel
        elif i > 1:
            conv = np.zeros((1, 3, 3))   # Placeholder for image conv kernel
        layers.append(conv)
    return layers

def train_hierarchical_network(layers, sinograms, noise_images, num_iterations=1000, batch_size=20):
    """
    Trains the network end-to-end with noise-based training.
    """
    for iteration in range(num_iterations):
        # Randomly select one layer to update
        layer_idx = np.random.randint(0, len(layers))
        
        # Freeze all other layers (conceptual: skip gradient accumulation for them)
        # Generate batch of noise
        noise_batch = noise_images[np.random.choice(len(noise_images), batch_size)]
        
        # Forward projection to get sinograms (distance-driven)
        sinogram_batch = forward_projection(noise_batch)
        
        # Forward pass through selected layer
        predictions = layers[layer_idx].forward(sinogram_batch)
        
        # Compute loss (MSE)
        loss = np.mean((predictions - noise_batch) ** 2)
        
        # Backward pass through selected layer
        grad_out = 2 * (predictions - noise_batch) / (predictions.size)
        grad_in = layers[layer_idx].backward(grad_out)
        
        # Update weights (conceptual SGD step)
        # layers[layer_idx].weights -= learning_rate * gradient_of_loss_wrt_weights
```

### Edge Cases and Pitfalls
*   **Vanishing Gradients:** During end-to-end training, gradients may vanish across the deep hierarchy. This is mitigated by updating only one layer per iteration while freezing others [S1].
*   **Parameter Explosion:** Implementing the transforms as fully connected layers would require $512^4 \approx 69$ billion parameters per stage, which is intractable. The sparse connection layer reduces the parameter count to $O(n \log n)$ [S1].
*   **Noise Learning:** If non-linear activations are used during noise-based training, the network may learn to reproduce noise artifacts as prior information. Using linear activations constrains the network to learn the linear inverse operator [S1].
*   **Dimension Ratios:** The ratio of depth bins to projection angles between stages does not need to be an integer, but the network must be trained to handle the resampling [S1].

## Operating characteristics
*   **Time Complexity:** The computational complexity scales linearly with data size, even for extension to three dimensions, due to sparse network connections between hierarchical levels [S1].
*   **Memory Complexity:** The number of parameters scales as $O(n \log n)$ for 2D and $O(n^2 \log n)$ for 3D, where $n$ is the data dimension. For a $512 \times 512$ 2D image, the network uses ~42 million parameters, which is 0.06% of the 69 billion parameters required by a fully connected layer [S1].
*   **Measured Performance:** In the reference implementation, training took ~4 hours for pre-training individual layers and ~1 day for end-to-end refinement on an NVIDIA Tesla V100 GPU [S1].
*   **Strengths:** Scalable to full-size problems (e.g., $512^4$ system matrix); avoids brute-force dense networks; does not rely on direct or iterative inversion techniques [S1].
*   **Limitations:** Requires synthetic noise data for initial training; clinical refinement is needed for optimal image quality; currently demonstrated primarily for 2D parallel-beam geometry [S1].
*   **When to Use:** When reconstructing large-scale tomographic data where traditional iterative methods are too slow and fully connected deep learning models are intractable.
*   **When Not to Use:** For small-scale problems where analytical methods (FBP) are sufficient and fast, or when hardware constraints prevent the training of deep hierarchical networks.

## Background
The hierarchical approach was conceived independently of the AUTOMAP approach [S1]. Prior attempts using supervised learning relied on brute-force fully connected networks limited to $128^4$ system matrix sizes, which could not practically scale to realistic data sizes like $512^4$ or $512^6$ [S1]. This method was introduced to solve such large-scale inverse problems by decomposing the domain transform into hierarchical stages [S1]. It is applicable to other imaging problems such as emission and magnetic resonance reconstruction [S1].

## Sources
[S1] Deep learning tomographic reconstruction through hierarchical decomposition of domain transforms., https://doi.org/10.1186/s42492-022-00127-y, fetched 2026-10-26
