# UE4-NeRF


## Overview
UE4-NeRF is a neural rendering system designed for the real-time interactive rendering of large-scale 3D scenes by integrating Neural Radiance Fields (NeRF) with the Unreal Engine 4 (UE4) rasterization pipeline [S1]. It partitions large environments into sub-NeRFs, representing each block using textured polygonal meshes initialized from regular octahedra, and utilizes a Level of Detail (LOD) hierarchy to maintain high frame rates at varying observation heights [S1]. Unlike Block-NeRF, which is limited to grid structures, UE4-NeRF supports diverse terrains including mountainous areas, and unlike Mobile-NeRF, it handles tilted surfaces and semi-transparent objects more effectively [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** A set of posed camera images (typically drone-captured) with GPS metadata; camera intrinsics and extrinsics; a large-scale scene bounding box.
*   **Outputs:** A set of textured polygonal meshes (shading meshes) for each scene block at multiple Levels of Detail (LOD); a neural shader (HLSL fragment shader) deployed in UE4; real-time rendered RGB images.

**Data Structures**
1.  **Scene Partitioning:** The large scene is divided into overlapping blocks. Each training area is slightly larger than the corresponding scene area to ensure smooth transitions at boundaries [S1].
2.  **Mesh Representation:** Each block is represented by a textured polygonal mesh. The mesh is initialized from a grid and optimized during training. The texture atlas stores feature vectors and opacity information [S1].
3.  **Coordinate System:**
    *   X-axis: East.
    *   Y-axis: South.
    *   Z-axis: Up.
    *   Scale: 100:1 (UE4 units to real-world units) [S1].
4.  **Network Architecture:**
    *   **Encoder:** A 4-layer MLP with dimensions $32 \times 64$, $64 \times 64$, $64 \times 64$, $64 \times 8$. It takes multi-resolution hash-encoded position features and outputs an opacity scalar $\alpha$ and an 8-dimensional feature vector $\mathbf{m}$ [S1].
    *   **Decoder:** A 3-layer MLP with dimensions $17 \times 16$, $16 \times 16$, $16 \times 3$. It takes the feature vector $\mathbf{m}$ and spherical harmonics coefficients of the view direction to predict the final RGB color $\mathbf{c}$ [S1].

**Method Steps**

1.  **Preprocessing and Coordinate Transformation:**
    *   Convert GPS-captured drone imagery coordinates to the UE4 coordinate system (X=East, Y=South, Z=Up) and apply the 100:1 scale conversion [S1].

2.  **Block Partitioning:**
    *   Partition the large-scale scene into multiple blocks (sub-NeRFs).
    *   Train a coarse model for sub-region segmentation to capture overall terrain and building outlines.
    *   Filter images associated with each sub-region and apply a mask to exclude pixels outside the specific scene area [S1].

3.  **Mesh Initialization:**
    *   Initialize a $128 \times 128 \times 128$ grid for each block [S1].
    *   Select the center point of each grid cell along with its six neighboring grids (front, back, left, right, top, bottom) to form the initial polygonal mesh structure [S1].
    *   Use regular octahedra with 20 faces (8 exterior, 12 interior) instead of the Mobile-NeRF approach to address slow/unstable convergence on tilted surfaces [S1].

4.  **Training (Optimization):**
    *   For each block, optimize the Encoder-Decoder network parameters and the vertex positions of the meshes simultaneously [S1].
    *   **Encoding:** Use multi-resolution hash encoding (from Instant-NGP) for the position input $p_i$ to accelerate training [S1].
    *   **View Direction:** Represent the viewing direction $d$ using spherical harmonics $SH(d)$ [S1].
    *   **Forward Pass:**
        *   Compute opacity and feature vector:
            $$ \mathbf{m}_i, \alpha_i = E(\text{hash}(p_i); \theta_E) $$
        *   Compute color:
            $$ \mathbf{c}_i = D(\mathbf{m}_i, SH(d_i); \theta_D) $$
        *   Where $E$ is the Encoder MLP and $D$ is the Decoder MLP [S1].
    *   **Loss:** Minimize the difference between rendered colors and ground truth images using PSNR/SSIM and LPIPS metrics [S1].
    *   **Hardware:** Implement computationally intensive portions using CUDA on NVIDIA GPUs (e.g., RTX 3090) [S1].
    *   **Iterations:** Train for 80,000 epochs per block to achieve convergence [S1].

5.  **Shading Mesh Extraction:**
    *   Extract coarse polygonal meshes without textures during training.
    *   Apply vertex optimization results.
    *   Encode texture maps (feature vectors and opacity) into the network outputs [S1].

6.  **Level of Detail (LOD) Generation:**
    *   Train meshes with varying levels of detail for different observation heights.
    *   Coarser meshes are used for higher observation heights to ensure real-time performance [S1].
    *   Extract pre-rendered polygonal meshes at different LODs for the final rendering module [S1].

7.  **Rendering in UE4:**
    *   Deploy the Encoder-Decoder network as an HLSL fragment shader in UE4 [S1].
    *   The shader takes feature vectors (from the texture atlas) as input and outputs colors [S1].
    *   Combine the extracted LOD meshes with the UE4 rasterization pipeline [S1].
    *   UE4 handles occlusion automatically; if traditional 3D models (e.g., .obj, .fbx) are imported, the neural shader computation is skipped for occluded regions [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# UE4-NeRF Training and Rendering Logic
# Note: This pseudocode outlines the logical flow based on the source description.
# Actual implementation requires CUDA kernels for hash encoding and UE4 HLSL shaders.

class UE4NeRFBlock:
    def __init__(self, scene_bounds, grid_size=128):
        self.scene_bounds = scene_bounds
        self.grid_size = grid_size
        # Initialize mesh vertices from 128x128x128 grid neighbors
        self.vertices = initialize_octahedron_mesh(grid_size) 
        # 20 faces: 8 exterior, 12 interior
        self.faces = get_octahedron_faces()
        
        # Network Parameters
        # Encoder: 4-layer MLP (32->64->64->64->8)
        self.encoder_weights = init_mlp_weights([32, 64, 64, 64, 8])
        # Decoder: 3-layer MLP (17->16->16->3)
        # Input: 8 (features) + 9 (SH coefficients) = 17
        self.decoder_weights = init_mlp_weights([17, 16, 16, 3])
        
        # Multi-resolution hash grid (from Instant-NGP)
        self.hash_grid = init_hash_grid()

    def encode_position(self, position):
        # Apply multi-resolution hash encoding to position
        # position is in UE4 coordinate system (X=East, Y=South, Z=Up)
        # Scale is 100:1
        hashed_features = hash_encode(position, self.hash_grid)
        return hashed_features

    def spherical_harmonics(self, direction):
        # Compute SH coefficients for view direction
        # direction is normalized vector
        sh_coeffs = compute_sh_coefficients(direction, degree=3) # 9 coefficients
        return sh_coeffs

    def forward_encoder(self, hashed_pos):
        # E(hash(pi); theta_E) -> Mi, alpha_i
        # Input: hashed_pos (dim 32)
        # Output: feature_vector (dim 8), opacity (scalar)
        x = hashed_pos
        for layer in self.encoder_layers:
            x = relu(layer(x))
        feature_vector = x[:, :8]
        opacity = sigmoid(x[:, 8]) # Assuming last output is opacity
        return feature_vector, opacity

    def forward_decoder(self, feature_vector, sh_coeffs):
        # D(Mi, SH(di); theta_D) -> ci
        # Input: feature_vector (dim 8) + sh_coeffs (dim 9) = 17
        # Output: color (RGB, dim 3)
        input_vec = concat(feature_vector, sh_coeffs)
        x = input_vec
        for layer in self.decoder_layers:
            x = relu(layer(x))
        color = sigmoid(x) # Output RGB
        return color

    def render_ray(self, ray_origin, ray_direction):
        # Calculate intersection with polygonal mesh
        # Ray: r(t) = o + t*d
        intersection_point, normal = intersect_mesh(ray_origin, ray_direction, self.vertices, self.faces)
        
        if intersection_point is None:
            return background_color

        # Encode position at intersection
        hashed_pos = self.encode_position(intersection_point)
        
        # Get view direction SH coefficients
        sh_coeffs = self.spherical_harmonics(ray_direction)
        
        # Get features and opacity
        feature_vector, opacity = self.forward_encoder(hashed_pos)
        
        # Get color
        color = self.forward_decoder(feature_vector, sh_coeffs)
        
        return color

    def train_step(self, images, poses, optimizer):
        # Filter images for this block using mask
        block_images, block_poses = filter_images_by_mask(images, poses, self.scene_bounds)
        
        for image, pose in zip(block_images, block_poses):
            # Generate rays for each pixel
            rays = generate_rays(pose, image.shape)
            
            rendered_colors = []
            for ray in rays:
                color = self.render_ray(ray.origin, ray.direction)
                rendered_colors.append(color)
            
            # Compute Loss (PSNR/SSIM/LPIPS)
            loss = compute_loss(rendered_colors, image)
            
            # Backpropagate to update network weights and vertex positions
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            
            # Update mesh vertices based on gradient
            update_mesh_vertices(self.vertices, self.faces, gradients)

    def extract_mesh_for_lod(self, lod_level):
        # Extract coarse polygonal meshes without textures
        # Apply vertex optimization results
        # Encode texture maps (feature vectors and opacity)
        mesh = self.vertices
        texture_atlas = generate_texture_atlas(self.encoder_weights, self.decoder_weights)
        return mesh, texture_atlas

# Rendering in UE4
def ue4_render_loop(meshes_lod, texture_atlases, camera_pose):
    # Select LOD based on camera height
    lod_index = select_lod(camera_pose.height, meshes_lod)
    mesh = meshes_lod[lod_index]
    texture = texture_atlases[lod_index]
    
    # Deploy neural shader (HLSL fragment shader)
    # Shader takes feature vectors from texture atlas
    # Outputs colors
    rendered_image = rasterize_with_neural_shader(mesh, texture, camera_pose)
    
    # Handle occlusion with imported 3D models (.obj, .fbx)
    # UE4 automatically skips neural shader for occluded regions
    final_image = handle_occlusion(rendered_image, imported_models)
    
    return final_image
```

**Parameters**
*   **Grid Initialization Size:** $128 \times 128 \times 128$ [S1].
*   **Octahedron Faces:** 20 total (8 exterior, 12 interior) [S1].
*   **Encoder MLP Dimensions:** $32 \times 64$, $64 \times 64$, $64 \times 64$, $64 \times 8$ [S1].
*   **Decoder MLP Dimensions:** $17 \times 16$, $16 \times 16$, $16 \times 3$ [S1].
*   **Training Hardware:** 1 Nvidia RTX 3090 GPU [S1].
*   **Training Iterations:** 80,000 epochs per block [S1].
*   **Coordinate System:** X=East, Y=South, Z=Up [S1].
*   **World Scale:** 100:1 [S1].

**Edge Cases and Pitfalls**
*   **Tilted Surfaces:** Mobile-NeRF suffers from slow/unstable convergence on tilted surfaces; UE4-NeRF addresses this by using regular octahedra [S1].
*   **Semi-transparent Objects:** Mobile-NeRF cannot handle transparent/semi-transparent objects (water, glass, plastic) due to opacity binarization. UE4-NeRF stores opacity in texture maps to mitigate this, though semi-transparent objects (e.g., vegetation) increase computational overhead and reduce frame rates [S1].
*   **Pre-rendering Gaps:** Pre-rendering cannot capture rays from all viewing angles, potentially resulting in gaps or holes in the final rendered scene [S1].
*   **Exposure Matching:** Consistent exposure matching with original images is difficult in UE4, causing evaluation metrics (PSNR/SSIM/LPIPS) to underestimate actual reconstruction quality [S1].
*   **Memory Overhead:** Rendering large scenes (square kilometers) incurs significant memory overhead, especially when using CUDA and NVIDIA GPUs. Increasing accuracy further amplifies memory consumption [S1].

## Operating characteristics
*   **Training Speed:** 1000× faster than Mobile-NeRF [S1].
    *   Training time per block: 40 minutes on 1×RTX 3090 [S1].
    *   Training time for full scene (n blocks): 1 hour + (40 minutes × n/4) on 1×RTX 3090 [S1].
*   **Rendering Performance:**
    *   Real-time rendering at 4K resolution with up to 43 FPS [S1].
    *   50 FPS on full scene at 2K resolution [S1].
    *   ≥70 FPS on individual blocks at 2K resolution [S1].
*   **Memory:** Significant memory overhead for large scenes (square kilometers). Requires NVIDIA GPUs and CUDA [S1].
*   **Strengths:** Real-time interactive rendering of large-scale scenes; handles tilted surfaces and semi-transparent objects better than Mobile-NeRF; supports scene editing with traditional 3D models; suitable for diverse terrains (mountainous, farmland) unlike Block-NeRF [S1].
*   **Limitations:** Pre-rendering gaps/holes; exposure matching issues in UE4; high memory consumption; semi-transparent objects reduce frame rates; requires NVIDIA GPUs [S1].
*   **When to Use:** Real-time interactive rendering of large-scale outdoor scenes (e.g., digital twins, VR, games) where high fidelity and editability are required.
*   **When Not to Use:** Scenes requiring perfect ray-casting from all angles without pre-rendering artifacts; environments with extremely high densities of semi-transparent objects where frame rate is critical; hardware without NVIDIA GPUs/CUDA support.

## Background
UE4-NeRF was introduced in the paper "UE4-NeRF: Neural Radiance Field for Real-Time Rendering of Large-Scale Scene" [S1]. It was developed to address the limitations of existing NeRF methods in real-time rendering of large-scale scenes, such as high training costs, slow rendering speeds, and lack of editability. The method combines NeRF with the Unreal Engine 4 (UE4) rasterization pipeline, leveraging LOD techniques and multi-resolution hash encoding to achieve real-time performance. It is used in UE4 for real-time interactive rendering and scene editing/manipulation, particularly for drone-captured large-scale scenes like farmlands, construction sites, and industrial parks, and has been evaluated on the Mega-NeRF public dataset [S1].

## Sources
[S1] UE4-NeRF: Neural Radiance Field for Real-Time Rendering of Large-Scale Scene, https://doi.org/10.48550/arXiv.2310.13263, fetched 2026-10-29
