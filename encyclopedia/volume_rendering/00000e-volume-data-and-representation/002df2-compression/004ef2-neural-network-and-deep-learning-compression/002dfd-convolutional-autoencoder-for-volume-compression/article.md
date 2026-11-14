# Convolutional Autoencoder for Volume Compression

Also called: Autoencoder-based volume rendering, CNN Autoencoder, Compressed Volume Rendering using Deep Learning, Data-driven compressed volume rendering, Deep Learning Compression

## Overview
This method obtains a data-driven compact representation of multivariate time-varying volumetric data using a deep convolutional autoencoder network, enabling interactive rendering and navigation by decompressing the representation on the GPU [S1, unofficial]. It solves the problem of visualizing large scientific datasets by storing only the compressed latent codes on GPU memory and reconstructing slices on the fly, achieving significantly lower Mean Squared Error (MSE) and higher Peak Signal-to-Noise Ratio (PSNR) than traditional wavelet compression at the same storage ratio [S1, unofficial]. It differs from its sibling, Sparse Coding (K-SVD), by using non-linear convolutional layers to capture hierarchical features rather than linear combinations of dictionary atoms [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Multivariate time-varying volume data represented as a multidimensional array of size $X \times Y \times Z \times K \times T$, where $X, Y, Z$ are spatial dimensions, $K$ is the number of multivariate fields (channels), and $T$ is the number of time steps [S1, unofficial].
*   **Output:** A compressed latent representation (latent code) for each time step, and a reconstructed volume slice during rendering [S1, unofficial].

**Data Structures:**
The network architecture is a stacked convolutional autoencoder. The specific layer configuration is defined in Table 1 of [S1, unofficial].
*   **Encoder Path:**
    1.  Convolutional: Input Channels 1, Output Channels 64, Stride 1, Filter Size 3x3.
    2.  Convolutional: Input Channels 64, Output Channels 64, Stride 1, Filter Size 3x3.
    3.  Max Pooling: Input Channels 64, Output Channels 64, Stride 2, Filter Size 2x2.
    4.  Convolutional: Input Channels 64, Output Channels 64, Stride 1, Filter Size 3x3.
    5.  Convolutional: Input Channels 64, Output Channels 32, Stride 1, Filter Size 3x3.
    6.  Max Pooling: Input Channels 32, Output Channels 32, Stride 2, Filter Size 2x2.
    7.  Convolutional: Input Channels 32, Output Channels 16, Stride 1, Filter Size 3x3.
    8.  Convolutional: Input Channels 16, Output Channels 8, Stride 1, Filter Size 3x3.
    9.  Max Pooling: Input Channels 8, Output Channels 8, Stride 2, Filter Size 2x2.
    10. ConvolutionalE (Encoder Output): Input Channels 8, Output Channels 4, Stride 1, Filter Size 3x3.
*   **Decoder Path:**
    1.  Convolutional: Input Channels 4, Output Channels 8, Stride 1, Filter Size 3x3.
    2.  Convolutional: Input Channels 8, Output Channels 16, Stride 1, Filter Size 3x3.
    3.  Convolutional: Input Channels 16, Output Channels 32, Stride 1, Filter Size 3x3.
    4.  Up Sampling: Input Channels 32, Output Channels 32, Stride 2, Filter Size 2x2.
    5.  Convolutional: Input Channels 32, Output Channels 64, Stride 1, Filter Size 3x3.
    6.  Convolutional: Input Channels 64, Output Channels 64, Stride 1, Filter Size 3x3.
    7.  Up Sampling: Input Channels 64, Output Channels 64, Stride 2, Filter Size 2x2.
    8.  Convolutional: Input Channels 64, Output Channels 64, Stride 1, Filter Size 3x3.
    9.  Convolutional: Input Channels 64, Output Channels 64, Stride 1, Filter Size 3x3.
    10. Up Sampling: Input Channels 64, Output Channels 64, Stride 2, Filter Size 2x2.
    11. ConvolutionalD (Decoder Output): Input Channels 64, Output Channels 1, Stride 1, Filter Size 3x3.

**Parameters:**
*   **Filter Size:** 3x3 for all convolutional layers [S1, unofficial].
*   **Optimizer:** ADADELTA solver with momentum of 0.95 [S1, unofficial].
*   **Loss Function:** Cross-entropy between the input slice and the reconstructed slice. Training stops when the Mean Squared Error (MSE) between the input volume and the reconstructed volume stops decreasing [S1, unofficial].

**Steps:**

1.  **Preprocessing:**
    *   Normalize the input volume data between 0 and 1 for each multivariate dimension across all time steps [S1, unofficial].
    *   Arrange the data as multivariate slices along the z-axis. The multivariate field acts as a channel to the network. Specifically, for a given time step, the data is reshaped such that the input to the network is a 2D slice with $K$ channels (one per variable) [S1, unofficial].

2.  **Training:**
    *   Randomly sample multivariate slices from all time steps [S1, unofficial].
    *   Feed slices through the autoencoder.
    *   Update weights using backpropagation to minimize the cross-entropy loss [S1, unofficial].
    *   The output activation function for the final layer is sigmoid:
        $$f(v_i) = \frac{1}{1 + e^{-v_i}}$$
        This maps the output of the final convolutional layer to the range [0, 1] [S1, unofficial].
    *   Continue until the MSE between input and reconstructed volume converges [S1, unofficial].

3.  **Decompression (Rendering):**
    *   Split the trained autoencoder into separate encoding and decoding networks [S1, unofficial].
    *   For a required time step, decompress subsampled slices along the z-axis using the decoder network to create a coarser volume [S1, unofficial].
    *   Render this coarser volume using ray casting [S1, unofficial].
    *   Continuously refine by decompressing remaining slices while rendering. Switch to finer volume rendering once decompression is complete [S1, unofficial].
    *   Apply a user-defined transfer function to map voxel intensity (0-255) to color and opacity. Opacity and color are linearly interpolated between adjacent control points [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def preprocess_volume(volume_data, num_time_steps, num_channels):
    """
    volume_data: np.array of shape (X, Y, Z, K, T)
    Returns: list of np.arrays, each shape (Z, K) representing multivariate slices
    """
    # Normalize data between 0 and 1 for each multivariate dimension across all time steps
    # Assuming volume_data is float32/64
    min_vals = np.min(volume_data, axis=(0, 1, 2, 3), keepdims=True)
    max_vals = np.max(volume_data, axis=(0, 1, 2, 3), keepdims=True)
    
    # Avoid division by zero
    range_vals = max_vals - min_vals
    range_vals[range_vals == 0] = 1
    
    normalized_data = (volume_data - min_vals) / range_vals
    
    # Arrange data as multivariate slices along the z-axis
    # Reshape to (T, Z, K) effectively, but stored as list of slices
    slices = []
    for t in range(num_time_steps):
        # Extract slice at each z-index for all channels
        # The source implies arranging as slices along z-axis, so we iterate Z
        # However, for training, we sample slices. Let's assume we extract all Z slices for all T
        for z in range(volume_data.shape[2]):
            slice_data = normalized_data[:, :, z, :, t] # Shape (X, Y, K) ? 
            # Note: Source says "arrange data as multivariate slices along the z-axis".
            # Standard interpretation for CNN input (X,Y,K) where K is channels.
            # The source table shows Input Channels 1 for the first layer, 
            # but preprocessing says "multivariate field acts as a channel".
            # There is a contradiction in the source table (Input 1) vs text (K channels).
            # We follow the text description of preprocessing: K channels.
            # However, Table 1 explicitly says Input Channels 1.
            # [G] Assuming the source table is specific to a single-channel demo or K=1,
            # but the text describes K channels. We will implement for K channels as per text.
            slices.append(slice_data)
    return slices

def train_autoencoder(slices, encoder_weights, decoder_weights, learning_rate=0.95):
    """
    Jointly train encoder and decoder using ADADELTA.
    """
    # Initialize optimizer states for ADADELTA
    # ... (Standard ADADELTA implementation details omitted for brevity, assumed available)
    
    for epoch in range(max_epochs):
        total_loss = 0
        for slice_data in slices:
            # Forward pass Encoder
            latent_code = encoder_forward(slice_data, encoder_weights)
            
            # Forward pass Decoder
            reconstructed_slice = decoder_forward(latent_code, decoder_weights)
            
            # Apply Sigmoid to output
            reconstructed_slice = 1.0 / (1.0 + np.exp(-reconstructed_slice))
            
            # Calculate Cross Entropy Loss
            # Note: Inputs are normalized to [0,1]. 
            # Cross entropy is typically used for binary classification, 
            # but source specifies it here.
            loss = -np.mean(slice_data * np.log(reconstructed_slice + 1e-8) + 
                            (1 - slice_data) * np.log(1 - reconstructed_slice + 1e-8))
            
            # Calculate MSE for stopping criterion
            mse = np.mean((slice_data - reconstructed_slice) ** 2)
            
            # Backpropagation to update weights
            encoder_weights, decoder_weights = update_weights_ada_delta(
                encoder_weights, decoder_weights, loss, learning_rate
            )
            
            total_loss += loss
            
        if mse_converged(mse):
            break
            
    return encoder_weights, decoder_weights

def render_volume(encoder_weights, decoder_weights, compressed_representations, transfer_function):
    """
    compressed_representations: list of latent codes for each time step
    """
    for t in range(len(compressed_representations)):
        # Decompress subsampled slices along z-axis
        # Create coarser volume
        coarser_volume = []
        for z in range(0, Z, step): # Step depends on subsampling
            latent = compressed_representations[t][z]
            # Decoder forward pass
            recon_slice = decoder_forward(latent, decoder_weights)
            recon_slice = 1.0 / (1.0 + np.exp(-recon_slice))
            coarser_volume.append(recon_slice)
            
        # Render coarser volume using ray casting
        render_ray_cast(coarser_volume, transfer_function)
        
        # Refine by decompressing remaining slices
        for z in range(0, Z, step):
            if z not in coarser_volume_indices:
                latent = compressed_representations[t][z]
                recon_slice = decoder_forward(latent, decoder_weights)
                recon_slice = 1.0 / (1.0 + np.exp(-recon_slice))
                update_volume(coarser_volume, z, recon_slice)
                render_ray_cast(coarser_volume, transfer_function)
```

## Subtopics

### [3D Shape Autoencoder](6689-3d-shape-autoencoder.md)
The 3D Shape Autoencoder is a convolutional autoencoder architecture designed to learn a compact prior distribution of 3D shapes. It solves the problem of reconstructing complex 3D geometry from limited input data by encoding spatial structures into a low-dimensional latent vector and decoding them back into volumetric representations. It differs from its parent, the Convolutional Autoencoder for Volume Compression, by focusing specifically on learning a generalizable shape prior for geometric reconstruction tasks (such as novel-view synthesis and relighting) rather than compressing time-varying multivariate scientific data. It is distinct from Deep Implicit Volume Compression, which uses implicit neural representations (INRs) rather than explicit voxel-based convolutional encoding/decoding [6807-deep-implicit-volume-compression.md].

### [Deep Implicit Volume Compression](6807-deep-implicit-volume-compression.md)
Deep Implicit Volume Compression is a block-based neural network architecture trained end-to-end to compress Truncated Signed Distance Fields (TSDF) stored in 3D voxel grids, achieving state-of-the-art rate-distortion trade-offs. It solves the problem of topological errors in reconstructed surfaces caused by quantization noise in magnitude values by losslessly compressing the sign configuration separately from the geometric magnitude. Unlike standard convolutional autoencoders that minimize only reconstruction error, this method employs dual-output heads and conditional entropy coding to minimize bit rate while guaranteeing that the reconstructed surface is homeomorphic to the ground truth.

### Conditional Diffusion Model for Volume Compression
A conditional diffusion model that compresses tensor blocks, with post-processing that corrects blocks exceeding the user-specified error bound [S2, unofficial]. On E3SM climate data, it achieved at least 2× larger compression ratios than SZ3 above NRMSE $10^{-4}$ [S2, unofficial]. However, the iterative denoising process imposes a severe decoding bottleneck: decoding time is 978 seconds, compared to 12 seconds for SZ3 and 2 seconds for a standard autoencoder, making data-space diffusion impractical for interactive use [S2, unofficial].

### FlowNet
The first deep learning framework for clustering and selecting representative streamlines/stream surfaces, which voxelizes flow structures into binary volumes and trains a 3D CNN autoencoder to produce feature descriptors [S2, unofficial]. Steps: 1. Voxelize each flow structure into a binary volume. 2. Train a 3D CNN autoencoder to produce a 1024-dimensional feature descriptor per structure. 3. Perform dimensionality reduction via t-SNE to project descriptors into 2D space. 4. Use DBSCAN clustering to identify groups for filtering and selection [S2, unofficial]. Across eleven flow field datasets, it achieved training F1 scores of 0.72–0.96 and test F1 scores of 0.57–0.86 [S2, unofficial].

### Temporal Selection via Autoencoder Descriptors
An extension of the autoencoder paradigm for selecting representative timesteps from long time-varying simulations by producing multivariate temporal descriptors [S2, unofficial]. Steps: 1. Train an encoder-decoder network accepting multichannel volumes (one channel per scalar variable). 2. Produce a 1024-dimensional descriptor per timestep. 3. Project descriptors via t-SNE. 4. Select representatives by arc-length or angular criteria [S2, unofficial].

### Large-Scale Mapping with Moving Active Volume
Extends SDF Tracker with a moving active volume centered around the camera translation vector [S3]. For every 16-voxel increment, it encodes voxel blocks exiting the active TSDF on the lagging end (if not empty) and decodes voxel blocks entering the active TSDF on the front end (if they contain a previously stored descriptor) [S3]. This reduces voxel blocks from 4096 voxels to descriptor size or a token if the block is empty [S3].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** Stores only the compressed representation for all time steps on GPU memory. Decompression requires additional memory for only one reconstructed slice at full resolution [S1, unofficial].
*   **Compression Ratio:** Achieved a compression ratio of 14.28:1 (compressed representation takes 7% of total memory) [S1, unofficial].

**Measured Performance:**
*   **Dataset 1:** Size 52x52x37, 88 timesteps, 2.56GB. CNN Autoencoder MSE=0.000061, PSNR=42.125. Best Wavelet (Discrete Mayer) MSE=0.0015, PSNR=28.239 [S1, unofficial].
*   **Dataset 2:** Size 100x100x100, 20 timesteps, 5.81GB. CNN Autoencoder MSE=0.0049, PSNR=23.098. Best Wavelet (Biorthogonal 9/3) MSE=0.0219, PSNR=16.588 [S1, unofficial].
*   **Comparison:** Compared to Wavelet compression (Daubechies, Mayer, Biorthogonal), the CNN Autoencoder achieves significantly lower MSE and higher PSNR at the same 7% storage usage (14.28:1 compression ratio). Wavelets result in coarse reconstruction with artifacts; Autoencoder provides close approximation with no visible artifacts [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Enables interactive rendering and navigation of large time-varying datasets by fitting the compressed representation on GPU memory. Learns hierarchical features that capture high-level structures better than linear methods like wavelets [S1, unofficial].
*   **Limitations:** Requires offline training. The decoder network must be simple enough to run on the GPU for real-time decompression [S1, unofficial].

## Background
This method was proposed to address the challenge of visualizing large amounts of multivariate time-varying volumetric data generated by scientific simulations [S1, unofficial]. It builds on the principle that deep neural networks correspond to hierarchical Tucker decomposition, offering richer representational capacity than Tensor Approximation [S1, unofficial]. The approach was demonstrated on datasets related to portland cement hydration simulations, including X-ray microtomography scans of Ca3SiO5 particles [S1, unofficial].

## Sources
[S1] Compressed volume rendering using deep learning, https://www.cs.umd.edu/sites/default/files/scholarly_papers/Jain.pdf, fetched 2026-10-26 (unofficial)
[S2] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
[S3] Compressed Voxel-Based Mapping Using Unsupervised Learning, https://doi.org/10.3390/robotics6030015, fetched 2026-10-26
