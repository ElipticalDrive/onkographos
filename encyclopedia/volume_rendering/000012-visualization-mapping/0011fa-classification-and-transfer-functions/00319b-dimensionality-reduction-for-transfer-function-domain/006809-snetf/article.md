# SNETF

Also called: SNE-Based 2D Transfer Function Building, Stochastic Neighbor Embedding-Based 2D Transfer Function Building

## Overview
SNETF is a volume rendering method that constructs transfer functions by mapping high-dimensional voxel feature vectors into a 2D latent space using Stochastic Neighbor Embedding (SNE) [S1]. It solves the problem that high-dimensional feature spaces are unusable for direct visual exploration, enabling users to sketch 2D regions that correspond to specific volumetric structures [S1]. Unlike PCA or SOM-based methods which often merge objects or cause boundary intersections, SNETF preserves neighborhood identities to separate objects and their boundaries without overlap [S1].

## How it works
**Inputs:**
*   Input volume $V$ of dimensions $W \times H \times D$.
*   Number of training voxels $N$ (e.g., 10,000 for real data, 2,500 for synthetic) [S1].
*   SNE perplexity-like parameter $\kappa$ (e.g., 120) [S1].
*   Interpolation neighbor count $\vartheta$ (e.g., 25) [S1].

**Outputs:**
*   2D latent space matrix $Y \in \mathbb{R}^{N \times 2}$ for training voxels [S1].
*   2D latent space coordinates for all non-background voxels (via interpolation) [S1].
*   Rendered volume $\hat{V}$ based on user-defined regions in the 2D space [S1].

**Data Structures:**
*   $X \in \mathbb{R}^{N \times P}$: High-dimensional feature matrix for training voxels. $P=7$ in the standard implementation [S1].
*   $Y \in \mathbb{R}^{N \times 2}$: Low-dimensional embedding matrix [S1].
*   $D \in \mathbb{R}^{N \times N}$: Squared Euclidean distance matrix in HD space [S1].
*   $B \in \mathbb{R}^{N \times N}$: Squared Euclidean distance matrix in LD space [S1].
*   $\gamma^{(X)} \in \mathbb{R}^{N \times N}$: Conditional probabilities in HD space [S1].
*   $\gamma^{(Y)} \in \mathbb{R}^{N \times N}$: Conditional probabilities in LD space [S1].

**Steps:**

1.  **Feature Extraction:** Compute 7 features for each voxel in the volume. The features are grouped into three mappings [S1]:
    *   $Q_1=3$: Intensity, gradient magnitude, and second derivative [S1].
    *   $Q_2=2$: Mean and standard deviation of voxel intensities within a spherical neighborhood [S1].
    *   $Q_3=2$: Low value and high value from a histogram-based mapping [S1].
    *   The resulting feature vector for voxel $r$ is $s_r \in \mathbb{R}^P$ where $P=7$ [S1].

2.  **Sampling and Normalization:**
    *   Remove background voxels using a thresholding operation [S1].
    *   Randomly select $N$ voxels from the remaining set [S1].
    *   Normalize each column of the feature matrix $X$ to the range $[0, 1]$ [S1].

3.  **SNE Embedding (Training):**
    *   Compute the squared Euclidean distance matrix $D$ where $d_{nn'} = \|x_n - x_{n'}\|^2$ [S1].
    *   Compute the squared Euclidean distance matrix $B$ for the initial random positions in $\mathbb{R}^2$ where $b_{nn'} = \|y_n - y_{n'}\|^2$ [S1].
    *   Determine Gaussian kernel widths $\sigma_n^X$ and $\sigma_n^Y$ such that the entropy of the probability distribution over $\kappa$ neighbors equals $\log(\kappa)$ [S1].
    *   Compute conditional probabilities in HD space:
        $$ \gamma_{nn'}^{(X)} = \frac{\exp(-d_{nn'}^2 / \sigma_n^X)}{\sum_{q \neq n} \exp(-d_{nq}^2 / \sigma_n^X)} $$
        [S1]
    *   Compute conditional probabilities in LD space:
        $$ \gamma_{nn'}^{(Y)} = \frac{\exp(-b_{nn'}^2 / \sigma_n^Y)}{\sum_{q \neq n} \exp(-b_{nq}^2 / \sigma_n^Y)} $$
        [S1]
    *   Optimize the positions $Y$ to minimize the Kullback-Leibler divergence between the distributions:
        $$ Y^* = \arg \min_{y_n} \sum_{n=1}^N \sum_{n'=1}^N \gamma_{nn'}^{(X)} \log \left( \frac{\gamma_{nn'}^{(X)}}{\gamma_{nn'}^{(Y)}} \right) $$
        [S1]
    *   This optimization is performed using a gradient descent-based algorithm [S1].

4.  **Interpolation (Projection):**
    *   For every non-training voxel $v$ in the volume, compute its 7-dimensional feature vector $f_v$ [S1].
    *   Normalize $f_v$ using the same scaling factors derived from $X$ in Step 2 [S1].
    *   Find the $\vartheta$ nearest neighbors of $f_v$ in the training feature matrix $X$ [S1].
    *   Map $v$ to the 2D latent space $y_v$ using k-nearest neighbor interpolation based on the trained positions $Y$ [S1].

5.  **Transfer Function Building and Rendering:**
    *   The user sketches a region $\Lambda$ in the 2D latent space $Y$ [S1].
    *   The rendered volume $\hat{V}$ consists of voxels $v_r$ whose mapped 2D position $y_n$ falls within $\Lambda$ [S1].
    *   The index $n$ for a voxel at coordinates $(i, j, k)$ is calculated as $n = i + j \cdot w + k \cdot w \cdot h$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_features(volume):
    """
    Computes the 7-dimensional feature vector for each voxel.
    Returns a 3D array of shape (H, W, D, 7).
    """
    # Q1: Intensity, Gradient Magnitude, Second Derivative
    intensity = volume
    gradient_mag = compute_gradient_magnitude(volume)
    second_deriv = compute_second_derivative(volume)
    
    # Q2: Mean and Standard Deviation in spherical neighborhood
    # Note: Neighborhood size grows to maintain normal distribution similarity
    mean_val = compute_local_mean(volume)
    std_val = compute_local_std(volume)
    
    # Q3: Low and High values from histogram mapping
    low_val = compute_histogram_low(volume)
    high_val = compute_histogram_high(volume)
    
    features = np.stack([intensity, gradient_mag, second_deriv, 
                         mean_val, std_val, low_val, high_val], axis=-1)
    return features

def normalize_columns(X):
    """
    Normalizes each column of matrix X to [0, 1].
    Returns normalized X and the min/max values used for scaling.
    """
    mins = X.min(axis=0)
    maxs = X.max(axis=0)
    ranges = maxs - mins
    # Avoid division by zero
    ranges[ranges == 0] = 1.0
    X_norm = (X - mins) / ranges
    return X_norm, mins, ranges

def compute_gaussian_widths(distances, kappa):
    """
    Finds sigma such that entropy over kappa neighbors equals log(kappa).
    distances: 1D array of distances from a point to all others.
    """
    # Sort distances to find the kappa-th neighbor
    sorted_dists = np.sort(distances)
    # Initial guess for sigma based on kappa-th neighbor distance
    sigma = sorted_dists[kappa] 
    # In practice, this requires an iterative solver to match entropy
    # Here we assume a helper function exists for this specific root-finding
    return sigma

def sne_step(X_train, kappa):
    """
    Performs SNE dimensionality reduction on training data.
    Returns Y (N x 2) embedding.
    """
    N, P = X_train.shape
    U = 2
    
    # Initialize Y randomly
    Y = np.random.randn(N, U)
    
    # Compute squared distances in HD
    # ||x_n - x_n'||^2
    diff = X_train[:, np.newaxis, :] - X_train[np.newaxis, :, :]
    D = np.sum(diff**2, axis=2)
    
    # Compute sigma_n for each point
    sigmas = np.zeros(N)
    for n in range(N):
        # Exclude self-distance (0)
        dists_n = D[n, :]
        sigmas[n] = compute_gaussian_widths(dists_n, kappa)
        
    # Compute gamma^(X)
    gamma_X = np.zeros((N, N))
    for n in range(N):
        exp_vals = np.exp(-D[n, :] / (2 * sigmas[n]**2))
        exp_vals[n] = 0 # Exclude self
        gamma_X[n, :] = exp_vals / np.sum(exp_vals)
        
    # Gradient Descent to minimize KL divergence
    # Loss = sum_{n,n'} gamma_X[n,n'] * log(gamma_X[n,n'] / gamma_Y[n,n'])
    # gamma_Y depends on Y
    learning_rate = 0.5
    for iteration in range(1000): # Fixed iterations for implementation
        # Compute squared distances in LD
        diff_y = Y[:, np.newaxis, :] - Y[np.newaxis, :, :]
        B = np.sum(diff_y**2, axis=2)
        
        # Compute sigma_n for LD space (similar to HD)
        sigmas_y = np.zeros(N)
        for n in range(N):
            dists_n = B[n, :]
            sigmas_y[n] = compute_gaussian_widths(dists_n, kappa)
            
        # Compute gamma^(Y)
        gamma_Y = np.zeros((N, N))
        for n in range(N):
            exp_vals = np.exp(-B[n, :] / (2 * sigmas_y[n]**2))
            exp_vals[n] = 0
            gamma_Y[n, :] = exp_vals / np.sum(exp_vals)
            
        # Compute gradients
        # dL/dY_n = 4 * sum_{n'} (gamma_X[n,n'] - gamma_Y[n,n']) * (Y_n - Y_n')
        # This is a simplified vectorized form
        diff = Y[:, np.newaxis, :] - Y[np.newaxis, :, :] # N x N x U
        joint = gamma_X - gamma_Y # N x N
        # Gradient for each point n
        grad = np.zeros((N, U))
        for n in range(N):
            # Sum over n'
            # (gamma_X[n,n'] - gamma_Y[n,n']) * (Y[n] - Y[n'])
            term = joint[n, :, np.newaxis] * diff[n, :, :] # N x U
            grad[n] = 4 * np.sum(term, axis=0)
            
        Y -= learning_rate * grad
        
    return Y

def interpolate_voxels(volume_features, X_train_norm, Y_train, theta):
    """
    Projects all volume voxels to 2D space using k-NN interpolation.
    volume_features: (H, W, D, 7) normalized features for all voxels
    X_train_norm: (N, 7) normalized training features
    Y_train: (N, 2) trained 2D positions
    theta: number of neighbors for interpolation
    """
    H, W, D, _ = volume_features.shape
    total_voxels = H * W * D
    
    # Flatten volume features
    all_features = volume_features.reshape(-1, 7)
    
    # Compute distances from all voxels to training voxels
    # This can be memory intensive. For implementation, use KDTree or batch processing.
    # Here we assume a function that returns indices of theta nearest neighbors
    # and their distances or weights.
    
    # Using a simple brute-force for clarity (inefficient for large volumes)
    # In practice, use scipy.spatial.KDTree
    from scipy.spatial import cKDTree
    tree = cKDTree(X_train_norm)
    
    # Query theta nearest neighbors
    distances, indices = tree.query(all_features, k=theta)
    
    # Interpolate Y coordinates
    # Weighted average based on inverse distance or similar
    Y_all = np.zeros((total_voxels, 2))
    
    for i in range(total_voxels):
        neighbors_idx = indices[i]
        neighbor_dists = distances[i]
        
        # Avoid division by zero if a voxel is in training set (dist=0)
        # If dist is 0, take the exact Y_train position
        if np.any(neighbor_dists == 0):
            exact_idx = neighbor_dists[neighbor_dists == 0][0]
            Y_all[i] = Y_train[neighbors_idx[neighbor_dists == 0][0]]
        else:
            # Inverse distance weighting
            weights = 1.0 / neighbor_dists
            weights = weights / np.sum(weights)
            Y_all[i] = np.sum(weights[:, np.newaxis] * Y_train[neighbors_idx], axis=0)
            
    # Reshape back to volume dimensions
    Y_volume = Y_all.reshape(H, W, D, 2)
    return Y_volume

def render(volume, Y_volume, region_lambda):
    """
    Renders the volume based on the 2D region lambda.
    region_lambda: A boolean mask or geometric definition in 2D space
    """
    # Create a mask for voxels whose 2D position is in lambda
    # Assuming region_lambda is a function or mask that evaluates to True for points in Y
    mask = np.zeros((volume.shape[0], volume.shape[1], volume.shape[2]), dtype=bool)
    
    # Iterate or vectorize check against region_lambda
    # Example: if lambda is a polygon, use point-in-polygon test
    for i in range(volume.shape[0]):
        for j in range(volume.shape[1]):
            for k in range(volume.shape[2]):
                y_pos = Y_volume[i, j, k]
                if is_in_region(y_pos, region_lambda):
                    mask[i, j, k] = True
                    
    # Apply transfer function (e.g., opacity) based on mask
    # This is a simplified representation
    rendered_volume = volume * mask.astype(float)
    return rendered_volume
```

**Parameters:**
*   $N$: Number of training voxels. Typical values: 2,500 for synthetic data, 10,000 for real data (e.g., tooth tomography) [S1].
*   $\kappa$ (kappa): Parameter for SNE, related to the number of neighbors for entropy calculation. Typical value: 120 (real data), 90 (synthetic data) [S1].
*   $\vartheta$ (theta): Number of neighbors for k-NN interpolation. Typical value: 25 (real data), 15 (synthetic data) [S1].
*   $P$: Dimensionality of HD feature space. Fixed at 7 in the cited implementation [S1].
*   $U$: Dimensionality of LD space. Fixed at 2 [S1].

**Edge Cases and Pitfalls:**
*   **High-Dimensional Usability:** High-dimensional feature spaces cannot be used for visual exploration; DR is mandatory [S1].
*   **Background Removal:** Background voxels must be removed via thresholding before sampling to increase the probability of selecting samples from small structures [S1].
*   **Object Separation:** Unlike PCA, which may merge object boundaries, SNETF separates objects and boundaries without intersections [S1].
*   **SOM Overlap:** Unlike SOM, which causes voxel overlap making TF building difficult, SNETF provides better separation [S1].

## Operating characteristics
*   **Time Complexity:** The SNE optimization step is $O(N^2)$ per iteration due to the distance matrix computation and gradient calculation over all pairs of training voxels [S1]. Interpolation is $O(M \log N)$ or $O(M \cdot N)$ depending on the nearest neighbor search implementation, where $M$ is the total number of voxels [S1].
*   **Memory Complexity:** Requires storing the $N \times N$ distance and probability matrices for SNE, and the full volume feature map and 2D embedding for rendering [S1].
*   **Performance:** Tested on a synthetic volume and a tooth tomography volume ($256 \times 256 \times 161$) [S1].
*   **Strengths:** High separability among objects in the 2D space; preserves boundaries as isolated transitions without intersections; better segmentation than PCA and SOM [S1].
*   **Limitations:** Manual TF building is still required in the 2D space; computational cost of SNE optimization for large $N$ [S1].
*   **When to use:** When high-dimensional feature spaces are needed for separability but visual TF design is required, and when object boundaries must be preserved without overlap [S1].
*   **When not to use:** When fully automated segmentation is required without user interaction [S1].

## Background
SNETF was introduced by Walter Serna-Serna, Andres M. Álvarez-Meza, and Álvaro-Ángel Orozco-Gutierrez from the Automatics Research Group at Universidad Tecnológica de Pereira, Colombia [S1]. The method is based on Stochastic Neighbor Embedding (SNE), a dimensionality reduction technique that preserves neighborhood identities by minimizing the Kullback-Leibler divergence between probability distributions in high and low dimensions [S1]. It was developed to address the limitations of previous DR methods like PCA and SOM in the context of multidimensional transfer functions, where PCA failed to segment boundaries individually and SOM caused voxel overlap [S1].

## Sources
[S1] Volume Rendering by Stochastic Neighbor Embedding-Based 2D Transfer Function Building, https://link.springer.com/chapter/10.1007/978-3-319-75193-1_74, fetched 2026-10-26
