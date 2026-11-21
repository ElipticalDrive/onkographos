# TriVol Encoding

Also called: Grouping Encoder, Initial TriVol Generation

## Overview
TriVol Encoding is a method for compressing sparse 3D point clouds into a dense, continuous feature representation suitable for neural volume rendering [S1]. It solves the memory bottleneck of standard dense voxel grids by grouping voxels along each axis into "slim" volumes, thereby amplifying the receptive field on the grouped axis while maintaining local detail on the other two [S1]. Unlike [CNN-based 2D Feature Extraction](8649-cnn-based-2d-feature-extraction.md) which operates on 2D image planes, TriVol operates directly on 3D spatial structures derived from point clouds, enabling efficient high-resolution modeling without proportional increases in computational cost [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sparse 3D point cloud $\mathbf{P}$, represented as a grid of voxels $\mathbf{V} \in \mathbb{R}^{C \times S \times S \times S}$, where $S$ is the volume resolution and $C$ is the base channel count [S1].
*   **Output:** Three "Initial TriVols" $\{\bar{\mathbf{V}}_x, \bar{\mathbf{V}}_y, \bar{\mathbf{V}}_z\}$, each with shape $\mathbb{R}^{(C \cdot N) \times G \times S \times S}$ (permuted per axis), where $G$ is the number of groups and $N = S/G$ is the group size [S1].

**Data Structures:**
The method generates three volumes corresponding to grouping along the x, y, and z axes.
*   $\bar{\mathbf{V}}_x$: Shape $\mathbb{R}^{(C \cdot N) \times G \times S \times S}$. The first dimension is the concatenated feature channels. The second dimension $G$ represents the number of groups along the x-axis. The last two dimensions $S \times S$ correspond to the y and z axes.
*   $\bar{\mathbf{V}}_y$: Shape $\mathbb{R}^{(C \cdot N) \times S \times G \times S}$. Groups are along the y-axis (second dimension).
*   $\bar{\mathbf{V}}_z$: Shape $\mathbb{R}^{(C \cdot N) \times S \times S \times G}$. Groups are along the z-axis (third dimension).

**Parameters:**
*   $S$: Volume resolution along the non-grouping axes. Typical value: 256 [S1].
*   $G$: Number of groups along the grouping axis. Typical value: 16 [S1].
*   $N$: Group size, calculated as $N = S/G$ [S1].
*   $C$: Base channel count of the input voxel grid [S1].

**Algorithm:**
1.  **X-Axis Grouping:** Divide the input voxel grid $\mathbf{V}$ into $G \times S \times S$ groups along the x-axis. Each group consists of $N = S/G$ consecutive voxels along the x-dimension. Concatenate the feature vectors of these $N$ voxels to form a single new feature voxel. This results in $\bar{\mathbf{V}}_x \in \mathbb{R}^{(C \cdot N) \times G \times S \times S}$ [S1].
2.  **Y-Axis Grouping:** Repeat the grouping process along the y-axis. Divide $\mathbf{V}$ into $S \times G \times S$ groups. Concatenate features along the y-dimension to produce $\bar{\mathbf{V}}_y \in \mathbb{R}^{(C \cdot N) \times S \times G \times S}$ [S1].
3.  **Z-Axis Grouping:** Repeat the grouping process along the z-axis. Divide $\mathbf{V}$ into $S \times S \times G$ groups. Concatenate features along the z-dimension to produce $\bar{\mathbf{V}}_z \in \mathbb{R}^{(C \cdot N) \times S \times S \times G}$ [S1]

**Formulas:**
The encoding operation $E$ is defined as:
$$
\begin{aligned}
\bar{\mathbf{V}}_x &= E_x(\mathbf{V}) \in \mathbb{R}^{(C \cdot N) \times G \times S \times S} \\
\bar{\mathbf{V}}_y &= E_y(\mathbf{V}) \in \mathbb{R}^{(C \cdot N) \times S \times G \times S} \\
\bar{\mathbf{V}}_z &= E_z(\mathbf{V}) \in \mathbb{R}^{(C \cdot N) \times S \times S \times G}
\end{aligned}
$$
[S1]

**Pitfalls:**
*   **Computational Cost:** If $G$ is not significantly smaller than $S$ ($G \ll S$), the number of groups increases, leading to a significant increase in computing resources and memory usage [S1]. The method relies on $G \ll S$ to allow high resolution $S$ without prohibitive costs [S1].
*   **Receptive Field:** The grouping amplifies the receptive field on the grouped axis by a factor of $N$, but local features on the ungrouped axes remain standard. This asymmetry must be accounted for in downstream decoding [S1].

## Subtopics

### TriVol Decoding
TriVol Decoding transforms the Initial TriVols into Feature TriVols using 3D Convolutional Networks. It uses three separate 3D UNet modules $\{D_x, D_y, D_z\}$, one for each Initial TriVol [S1]. The decoders do not share weights [S1]. The procedure is:
$$
\mathbf{V}_x = D_x(\bar{\mathbf{V}}_x), \quad \mathbf{V}_y = D_y(\bar{\mathbf{V}}_y), \quad \mathbf{V}_z = D_z(\bar{\mathbf{V}}_z)
$$
[S1] The 3D UNets acquire non-local features on the grouped axis (due to the amplified receptive field from encoding) and local features on the ungrouped axes [S1].

### TriVol Feature Querying
This step retrieves features from the decoded Feature TriVols $\{\mathbf{V}_x, \mathbf{V}_y, \mathbf{V}_z\}$ for a specific 3D location $\mathbf{x}$ along a camera ray [S1].
1.  **Ray Generation:** Calculate a ray with camera center $\mathbf{r}_o \in \mathbb{R}^3$ and normalized direction $\mathbf{r}_d \in \mathbb{R}^3$ using camera parameters $\{R, t, K\}$ [S1]. Sample points $\mathbf{x}$ along the ray using a coarse-to-fine strategy: $\mathbf{x} = \mathbf{r}_o + z \cdot \mathbf{r}_d$, where $z \in [z_n, z_f]$ [S1].
2.  **Interpolation:** For a queried location $\mathbf{x}$, use trilinear interpolation to extract feature vectors from each Feature TriVol: $\mathbf{V}_x(\mathbf{x}) \in \mathbb{R}^F$, $\mathbf{V}_y(\mathbf{x}) \in \mathbb{R}^F$, and $\mathbf{V}_z(\mathbf{x}) \in \mathbb{R}^F$ [S1].
3.  **Concatenation:** Combine the features:
    $$
    F(\mathbf{x}) = \mathbf{V}_x(\mathbf{x}) \oplus \mathbf{V}_y(\mathbf{x}) \oplus \mathbf{V}_z(\mathbf{x})
    $$
    where $\oplus$ denotes concatenation and $F$ is the channel number of the Feature TriVol [S1].

## Operating characteristics
*   **Memory Efficiency:** By setting $G \ll S$ (e.g., $G=16, S=256$), the method reduces the spatial dimensions along the grouping axis from $S$ to $G$, significantly lowering memory requirements compared to a full dense grid $\mathbf{V}$ [S1]. This allows for high resolution $S$ to model fine point cloud details without huge computing resources [S1].
*   **Receptive Field:** The encoding step amplifies the receptive field on the grouped axis by $N$ times, allowing the network to capture non-local features along that axis, while preserving local features on the other two axes [S1].
*   **Performance:** The method enables realistic image rendering with rich details on datasets like ScanNet (512x640 resolution) using standard hardware (e.g., RTX 3090 GPUs) [S1].
*   **Limitations:** The method requires three separate 3D UNet decoders, which increases model complexity compared to single-volume approaches, although the small $G$ mitigates the memory impact [S1].

## Background
TriVol Encoding was introduced in the paper "TriVol: Point Cloud Rendering via Triple Volumes" [S1]. It was developed to address the limitations of existing point cloud rendering methods, such as NPBG++, Dense 3D ConvNets, and Sparse 3D ConvNets, by providing a dense and continuous feature representation that is both memory-efficient and capable of capturing high-frequency details [S1]. The method builds on the NeRF rendering pipeline but replaces the implicit MLP input with features derived from these triple volumes [S1].

## Sources
[S1] TriVol: Point Cloud Rendering via Triple Volumes, https://doi.org/10.1109/cvpr52729.2023.01986, fetched 2026-10-26
