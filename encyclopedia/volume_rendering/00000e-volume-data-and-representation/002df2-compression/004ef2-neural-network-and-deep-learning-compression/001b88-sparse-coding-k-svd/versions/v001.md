# Sparse Coding (K-SVD)

Also called: Compressive Volume Rendering via Dictionary Learning, Dictionary Learning for Volume Rendering, K-SVD, K-SVD Dictionary Learning

## Overview
Sparse Coding (K-SVD) is a data-driven compression technique that represents volume-rendered images or volumetric data blocks as sparse linear combinations of atoms from an overcomplete dictionary learned via the K-SVD algorithm [S2, unofficial]. It solves the problem of high storage and transmission costs for large volume datasets by achieving high compression ratios (often under 1 bit per sample) while maintaining visual fidelity [S1, unofficial]. Unlike prescribed orthogonal bases like DCT, K-SVD learns a dictionary tailored to the specific statistical properties of the volume rendering data, offering superior reconstruction quality for missing pixel recovery and compression artifacts [S2, unofficial]. It differs from Vector Quantization (VQ) by allowing linear combinations of multiple dictionary atoms rather than selecting a single nearest neighbor, providing higher fidelity at the cost of more complex encoding [S2, unofficial].

## How it works
**Inputs:**
- Training dataset: A set of volume-rendered images (greyscale or RGB) generated from the target volume data from multiple viewpoints [S2, unofficial].
- Block size $B$: Spatial dimensions of the data blocks to be encoded (e.g., $16 \times 16$ for greyscale, $16 \times 16 \times 3$ for RGB) [S2, unofficial].
- Dictionary size $D$: Number of atoms in the dictionary [S1, unofficial][S2, unofficial].
- Sparsity level $L$ (or $K$): Maximum number of non-zero coefficients allowed per block [S1, unofficial][S2, unofficial].

**Outputs:**
- Learned dictionary matrix $\mathbf{D}$ of size $M \times D$, where $M$ is the dimension of a single block (e.g., $256$ for $16 \times 16$ greyscale, $768$ for $16 \times 16 \times 3$ RGB) [S2, unofficial].
- Sparse coefficient matrix $\mathbf{X}$ for each data block, where each column $\mathbf{x}_i$ has at most $L$ non-zero entries [S2, unofficial].

**Data Structures:**
- **Dictionary $\mathbf{D}$**: A matrix where each column is an atom. For greyscale, $\mathbf{D} \in \mathbb{R}^{256 \times 1024}$ [S2, unofficial]. For RGB, $\mathbf{D} \in \mathbb{R}^{768 \times 2304}$ [S2, unofficial].
- **Training Matrix $\mathbf{Y}$**: A matrix where each column is a flattened training block vector [S2, unofficial].
- **Coefficient Matrix $\mathbf{X}$**: A matrix where each column $\mathbf{x}_i$ corresponds to the sparse coefficients for the $i$-th block in $\mathbf{Y}$ [S2, unofficial].

**Algorithm Steps:**

1. **Training Data Generation**:
   - Render the volume dataset from multiple viewpoints (e.g., 35 views for training, 1 for testing) [S2, unofficial].
   - Center the dataset tightly in the camera view to minimize background pixels [S2, unofficial].
   - For greyscale training, convert color renderings to greyscale [S2, unofficial].
   - For RGB training, keep color channels [S2, unofficial].

2. **Block Extraction**:
   - Extract training blocks by sliding a window of size $B \times B$ (greyscale) or $B \times B \times 3$ (RGB) over the training images without overlap [S2, unofficial].
   - Flatten each block into a column vector. For RGB, linearize a $16 \times 16 \times 3$ block into a $768 \times 1$ vector: first 256 entries are Red, next 256 Green, remaining 256 Blue [S2, unofficial].
   - Assemble all training vectors into the matrix $\mathbf{Y}$ [S2, unofficial].

3. **Dictionary Learning (K-SVD)**:
   The goal is to solve:
   $$ \min_{\mathbf{D}, \mathbf{X}} \{ \|\mathbf{Y} - \mathbf{D}\mathbf{X}\|_F^2 \} \quad \text{subject to} \quad \forall_i \|\mathbf{x}_i\|_0 \leq L $$
   where $\|\cdot\|_F$ is the Frobenius norm and $\|\cdot\|_0$ is the $\ell_0$ norm (number of non-zero elements) [S2, unofficial].
   
   Iterate until convergence:
   a. **Sparse Coding Step**: For each column $\mathbf{y}_i$ in $\mathbf{Y}$, find the sparse coefficient vector $\mathbf{x}_i$ using Orthogonal Matching Pursuit (OMP) with sparsity constraint $L$ [S2, unofficial].
   b. **Dictionary Update Step**: Update the dictionary atoms in $\mathbf{D}$ to minimize the reconstruction error while preserving the sparsity pattern of $\mathbf{X}$ [S2, unofficial]. This step guarantees a reduction or no change in Mean Squared Error (MSE) [S2, unofficial].

4. **Encoding (Compression)**:
   - For a new data block $\mathbf{y}$, compute its sparse representation $\mathbf{x}$ using OMP with the learned dictionary $\mathbf{D}$ and sparsity $L$ [S2, unofficial].
   - Store the indices of the non-zero coefficients and their values [S2, unofficial].

5. **Decoding (Reconstruction)**:
   - Reconstruct the block as $\hat{\mathbf{y}} = \mathbf{D}\mathbf{x}$ [S2, unofficial].
   - For missing pixel recovery, if a signal $\mathbf{y}$ has missing pixels, form a subsampled signal $\tilde{\mathbf{y}}$ using a subsampling matrix $S$. Compute the decimated dictionary $\tilde{\mathbf{D}} = S\mathbf{D}$ and solve:
     $$ \min_{\mathbf{x}} \|\mathbf{x}\|_0 \text{ s.t. } \tilde{\mathbf{y}} = \tilde{\mathbf{D}}\mathbf{x} $$
     via OMP to recover $\mathbf{x}$, then approximate $\mathbf{y}$ as $\mathbf{D}\mathbf{x}$ [S2, unofficial].

**Parameters:**
- **Block Size**: $16 \times 16$ pixels for greyscale; $16 \times 16 \times 3$ for RGB [S2, unofficial].
- **Dictionary Size ($D$)**: $1024$ atoms for greyscale [S2, unofficial]; $2304$ atoms for RGB [S2, unofficial]; $1024$ for Chameleon/Supernova datasets in [S1, unofficial].
- **Sparsity Level ($L$ or $K$)**: $10$ for greyscale [S2, unofficial]; $6$ for Chameleon/Supernova; $4$ for Turbulence [S1, unofficial].
- **Initialization**: Dictionary can be initialized randomly or using a prescribed dictionary (e.g., DCT) [S2, unofficial].

**Pitfalls:**
- The algorithm converges under the condition that the sparse coding step provides the best approximation to the signal containing no more than $L$ nonzero entries [S2, unofficial].
- Training time is significant; e.g., 7.1 minutes for greyscale and 47.3 minutes for RGB on a single-threaded CPU [S2, unofficial].
- Using a prescribed dictionary (like DCT) may yield lower PSNR compared to a trained dictionary, especially with high missing pixel rates [S2, unofficial].

## Subtopics

### [CP Decomposition](2255-cp-decomposition.md)
CP Decomposition is a tensor factorization technique that represents a tensor as a sum of rank-one components, where each component is the outer product of vectors corresponding to each mode of the tensor. In the context of volume rendering, it is used to factorize the 3D (or 4D) radiance field tensor into compact vectors, significantly reducing the memory footprint compared to dense voxel grids. Unlike Tucker Decomposition (HOSVD), which uses a core tensor multiplied by factor matrices, CP uses only vectors, offering higher compactness but requiring more components to model complex scenes, leading to higher computational costs during reconstruction.

### [Tucker Decomposition (HOSVD)](5352-tucker-decomposition-hosvd.md)
Tucker Decomposition (HOSVD) is a global basis generation method that decomposes an $N$-dimensional input array into $N$ basis matrices and an $N$-dimensional core tensor. It solves the problem of storing and transmitting large volume datasets by reducing the reconstruction rank, thereby achieving data reduction. Unlike its sibling CP Decomposition, which sums rank-one tensors, Tucker Decomposition uses a multilinear product structure, allowing for a more compact representation of correlated data. It is distinguished from its parent Sparse Coding (K-SVD) by its specific application to global multiresolution basis generation for volume rendering, often integrated with octree hierarchies.

### [Karhunen-Loeve Transform (KLT)](5365-karhunen-loeve-transform-klt.md)
The Karhunen-Loeve Transform (KLT) is a linear decorrelation technique used in volume data compression to remove redundancy by estimating the covariance matrix and computing its eigenvectors. It is closely related to Principal Component Analysis (PCA) and provides superior decorrelation quality compared to the Discrete Cosine Transform (DCT). Unlike orthogonal basis methods like DCT, KLT bases are data-dependent and not mutually independent, which prevents the use of fast forward and inverse transforms. To mitigate the computational cost of these transforms, KLT is often combined with Vector Quantization (VQ) and block-wise decomposition, shifting the heavy reconstruction work to an offline preprocessing stage.

### [Sparse Representation (Dictionary Based)](5366-sparse-representation-dictionary-based.md)
Sparse Representation (Dictionary Based) encodes volume data blocks by finding a sparse linear combination of atoms from a pre-computed dictionary, solving an optimization problem to minimize reconstruction error under a sparsity constraint. It differs from Vector Quantization (VQ) by allowing each input block to be represented by a linear combination of multiple dictionary elements rather than a single nearest-neighbor prototype, offering higher fidelity at the cost of significantly more complex encoding.

### Orthogonal Matching Pursuit (OMP)
OMP is a greedy algorithm for sparse coding that selects dictionary atoms sequentially to find an approximate sparse solution $\mathbf{x}$ for the problem $\min \|\mathbf{x}\|_0$ subject to $\|\mathbf{y} - \mathbf{D}\mathbf{x}\|_2 \leq \epsilon$ [S2, unofficial]. It computes inner products between the signal $\mathbf{y}$ and dictionary columns $\mathbf{D}$, then deploys least squares solvers to update the solution coefficients [S2, unofficial]. The target sparsity $L$ represents the maximum number of nonzero entries allowed in $\mathbf{x}_i$ [S2, unofficial]. OMP is preferred over Matching Pursuit (MP) because it offers improvements on orthogonality and faster convergence [S2, unofficial].

### Compressive Volume Rendering with Dictionary Learning
This technique treats rendered images as collections of square pixel blocks to apply sparse coding and dictionary learning for recovery or compression [S2, unofficial]. It involves training a dictionary for greyscale images first, then extending to color images based on experimental insights [S2, unofficial]. The block size is typically $16 \times 16$ pixels for both greyscale and color images [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- **Training**: The K-SVD algorithm involves iterative sparse coding and dictionary updates. Training time is significant; e.g., 7.1 minutes for greyscale and 47.3 minutes for RGB on a single-threaded CPU [S2, unofficial].
- **Encoding**: Requires running OMP for each block, which is computationally intensive but faster than training [S2, unofficial].
- **Memory**: The dictionary size determines memory usage. For greyscale, a $256 \times 1024$ dictionary is used [S2, unofficial]. For RGB, a $768 \times 2304$ dictionary is used [S2, unofficial].

**Measured Performance:**
- **Compression Ratio**: Achieves bits-per-sample (bps) of 0.25 for Chameleon ($1024^2$), 0.07 for Supernova (1080p), and 0.20 for Turbulence ($512^4$) [S1, unofficial].
- **Quality**: Deblocked SSIM scores are 0.72 (Chameleon), 0.76 (Supernova), and 0.84 (Turbulence) [S1, unofficial].
- **Missing Pixel Recovery**: With 70% missing pixels, the trained dictionary achieves a PSNR of 31.2 dB, compared to 22.97 dB for a DCT dictionary [S2, unofficial].

**Strengths and Limitations:**
- **Strengths**: High compression ratios, superior reconstruction quality compared to prescribed dictionaries (like DCT), ability to handle missing data effectively [S2, unofficial].
- **Limitations**: High computational cost for training and encoding, sensitivity to sparsity level and dictionary size, requires a representative training dataset [S2, unofficial].

**When to Use:**
- Use when high compression ratios are needed and computational resources allow for offline training [S1, unofficial].
- Use when dealing with missing or incomplete data, as the method excels at recovery [S2, unofficial].

**When Not to Use:**
- Avoid when real-time encoding is required without pre-computed dictionaries [S2, unofficial].
- Avoid when the dataset is not representative of the training data, as the dictionary may not generalize well [S2, unofficial].

## Background
Sparse Coding with K-SVD was introduced in the context of volume rendering by researchers including those cited in [S1, unofficial] and [S2, unofficial]. The K-SVD algorithm itself was originally developed for general signal processing and image processing applications [MAB06, cited in S2, unofficial]. In volume rendering, it was adapted to treat rendered images as collections of pixel blocks, leveraging the redundancy and structure in volume-rendered data to achieve efficient compression and artifact reduction [S2, unofficial]. The method has been applied to datasets such as Chameleon, Supernova, and Turbulence, demonstrating its effectiveness in real-time deblocked GPU rendering scenarios [S1, unofficial].

## Sources
[S1] Real-time Deblocked GPU Rendering of Compressed Volumes, https://doi.org/10.2312/vmv.20141290, fetched 2026-10-26 (unofficial)
[S2] Compressive Volume Rendering, https://doi.org/10.11575/prism/25391, fetched 2026-10-27 (unofficial)
