# NeRF-based Mesh Extraction with Color Fusion

Also called: NeRF mesh extraction, color average method, vertex normal color method

## Overview
This method extracts a textured 3D mesh from a trained Neural Radiance Field (NeRF) by converting the volume density into a scalar field, extracting the isosurface via Marching Cubes, and coloring vertices by fusing colors from multiple training views weighted by occlusion and depth [S1, unofficial]. It distinguishes itself from simple vertex-normal querying by aggregating color information from all visible training cameras, reducing view-dependent artifacts and providing more robust colorization for complex geometries [S1, unofficial].

## How it works
**Inputs:**
- A trained NeRF model (`nerf_fine`) containing density ($\sigma$) and color prediction capabilities [S1, unofficial].
- Training dataset images, camera poses ($P_{c2w}$), and intrinsics ($K$) [S1, unofficial].
- Parameters: $N_{grid}$ (grid resolution), $\sigma_{threshold}$ (density threshold), $N_{samples}$ (coarse samples), $N_{importance}$ (fine samples), $chunk$ (batch size for memory management), $occ_{threshold}$ (occlusion threshold), $x_{range}, y_{range}, z_{range}$ (bounding box) [S1, unofficial].

**Outputs:**
- A PLY mesh file containing vertex positions (float32) and face indices (int32), with vertex colors [S1, unofficial].

**Data Structures:**
- `xyz_`: Tensor of shape $(N^3, 3)$ containing all 3D grid points [S1, unofficial].
- `dir_`: Tensor of shape $(N^3, 3)$, initialized to zeros (since $\sigma$ is direction-independent in standard NeRF queries) [S1, unofficial].
- `sigma`: 3D array of shape $(N, N, N)$ containing density values [S1, unofficial].
- `vertices`, `triangles`: Arrays from Marching Cubes [S1, unofficial].
- `v_color_sum`: Array of shape $(N_{vertices}, 3)$, accumulating weighted colors [S1, unofficial].
- `non_occluded_sum`: Array of shape $(N_{vertices}, 1)$, accumulating non-occluded weights [S1, unofficial].

**Steps:**

1.  **Initialize Model and Grid:**
    Load the pretrained `nerf_fine` checkpoint [S1, unofficial]. Initialize embeddings: `embedding_xyz` (3, 10) and `embedding_dir` (3, 4) [S1, unofficial]. Define the 3D grid using `np.linspace` for $x, y, z$ within the specified ranges with $N_{grid}$ points each [S1, unofficial]. Create `xyz_` tensor of shape $(N^3, 3)$ and `dir_` tensor of zeros [S1, unofficial].

2.  **Query Density ($\sigma$):**
    Batch query the NeRF model for occupancy [S1, unofficial]. For each chunk of points:
    - Embed coordinates: $e_{xyz} = \text{embedding\_xyz}(xyz_{chunk})$ [S1, unofficial].
    - Embed directions: $e_{dir} = \text{embedding\_dir}(dir_{chunk})$ [S1, unofficial].
    - Concatenate: $e_{combined} = \text{concat}(e_{xyz}, e_{dir})$ [S1, unofficial].
    - Pass through `nerf_fine`: $output = \text{nerf\_fine}(e_{combined})$ [S1, unofficial].
    - Extract $\sigma$ from the last channel of $output$, clamp to non-negative, and reshape to $(N, N, N)$ [S1, unofficial].

3.  **Extract Mesh Geometry:**
    Run the Marching Cubes algorithm on the $\sigma$ volume using $\sigma_{threshold}$ to retrieve vertices and triangles [S1, unofficial]. Transform vertices from grid coordinates to world coordinates: invert $x$ and $y$ (to correct for Marching Cubes convention), then scale and shift using the defined ranges [S1, unofficial]. Remove noise by keeping only the largest connected cluster of triangles [S1, unofficial].

4.  **Color Fusion (Color Average Method):**
    Initialize `v_color_sum` and `non_occluded_sum` to zero for all vertices [S1, unofficial]. Define camera intrinsics $K$ as a $3 \times 3$ matrix using focal length and image dimensions $W, H$ [S1, unofficial]. Transform vertices to homogeneous coordinates $(N_{vertices}, 4)$ by appending 1s [S1, unofficial].
    
    For each training image index:
    - Load image and resize to $img_{wh}$. Get camera-to-world pose $P_{c2w}$, invert to get world-to-camera $P_{w2c}$ ($3 \times 4$) [S1, unofficial].
    - Project vertices to camera coordinates: $v_{cam} = P_{w2c} @ v_{homo}^T$. Flip $y$ and $z$ axes ($v_{cam}[1:] *= -1$) to match 'right down forward' convention [S1, unofficial].
    - Project to pixel coordinates: $v_{image} = (K @ v_{cam})^T$. Divide by depth ($z + \epsilon$). Clip $x$ and $y$ to $[0, W-1]$ and $[0, H-1]$ [S1, unofficial].
    - Sample colors from the image at projected pixel coordinates using bilinear interpolation (e.g., `cv2.remap`). Split into chunks of 30,000 pixels to avoid OpenCV size limits [S1, unofficial].
    - Predict occlusion: Construct rays from camera origin to vertices. Set near bound to dataset bounds minimum and far bound to vertex depth [S1, unofficial]. Query NeRF fine model for accumulated opacity using $N_{samples}$ and $N_{importance}$ [S1, unofficial].
    - Accumulate weighted colors: $v_{color\_sum} += colors \times non\_occluded$. Accumulate weights: $non\_occluded\_sum += non\_occluded$ [S1, unofficial].
    
    Final vertex color: $C_v = v_{color\_sum} / (non\_occluded\_sum + \epsilon)$ [S1, unofficial].

5.  **Save Mesh:**
    Save the mesh to a PLY file with vertex positions and face indices [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np
import cv2
import mcubes
from collections import defaultdict

def query_density(nerf_fine, embedding_xyz, embedding_dir, xyz_chunk):
    """Query NeRF for density values."""
    xyz_embedded = embedding_xyz(xyz_chunk)
    dir_embedded = embedding_dir(torch.zeros_like(xyz_chunk)) # Direction independent for sigma
    xyzdir_embedded = torch.cat([xyz_embedded, dir_embedded], 1)
    out = nerf_fine(xyzdir_embedded)
    return out[:, -1] # Return sigma

def render_rays(nerf_fine, embeddings, rays, N_samples, use_disp, near, far, N_importance, chunk, white_back, test_time):
    """Standard batched NeRF ray marching to compute accumulated opacity."""
    # rays: (B, 6) -> [origin, direction]
    # Returns dict with 'acc' (accumulated opacity)
    # Implementation follows standard NeRF volumetric rendering loop
    # ... (standard ray marching logic) ...
    pass

def query_opacity(rays_o, rays_d, nerf_fine, embedding_xyz, embedding_dir, N_samples, N_importance, chunk):
    """Query accumulated opacity along rays using batched inference."""
    B = rays_o.shape[0]
    results = defaultdict(list)
    for i in range(0, B, chunk):
        chunk_rays = torch.cat([rays_o[i:i+chunk], rays_d[i:i+chunk]], dim=1)
        rendered = render_rays(nerf_fine, [embedding_xyz, embedding_dir], chunk_rays, N_samples, False, 0, 0, N_importance, chunk, False, test_time=True)
        results['acc'].append(rendered['acc'])
    return torch.cat(results['acc'], 0)

def extract_mesh(nerf_fine, embedding_xyz, embedding_dir, args, dataset):
    """Extract mesh geometry from NeRF density."""
    N = args.N_grid
    xmin, xmax = args.x_range
    ymin, ymax = args.y_range
    zmin, zmax = args.z_range
    
    x = np.linspace(xmin, xmax, N)
    y = np.linspace(ymin, ymax, N)
    z = np.linspace(zmin, zmax, N)
    
    xyz_ = torch.FloatTensor(np.stack(np.meshgrid(x, y, z), -1).reshape(-1, 3)).cuda()
    dir_ = torch.zeros_like(xyz_).cuda()
    
    sigma_chunks = []
    with torch.no_grad():
        for i in range(0, xyz_.shape[0], args.chunk):
            chunk = xyz_[i:i+args.chunk]
            sigma_chunks.append(query_density(nerf_fine, embedding_xyz, embedding_dir, chunk))
    
    sigma = torch.cat(sigma_chunks, 0).cpu().numpy()
    sigma = np.maximum(sigma, 0).reshape(N, N, N)
    
    vertices, triangles = mcubes.marching_cubes(sigma, args.sigma_threshold)
    
    # Transform to world coordinates
    vertices[:, 0] = -vertices[:, 0] # Invert x
    vertices[:, 1] = -vertices[:, 1] # Invert y
    # Scale and shift based on ranges
    vertices[:, 0] = (vertices[:, 0] / (N - 1)) * (xmax - xmin) + xmin
    vertices[:, 1] = (vertices[:, 1] / (N - 1)) * (ymax - ymin) + ymin
    vertices[:, 2] = (vertices[:, 2] / (N - 1)) * (zmax - zmin) + zmin
    
    # Remove noise (largest connected component)
    # Implementation depends on library (e.g., open3d)
    # mesh = o3d.geometry.TriangleMesh()
    # mesh.vertices = o3d.utility.Vector3dVector(vertices)
    # mesh.triangles = o3d.utility.Vector3iVector(triangles)
    # ... (largest connected component logic) ...
    
    return vertices, triangles

def fuse_colors(vertices, triangles, dataset, nerf_fine, embedding_xyz, embedding_dir, args):
    """Color vertices by fusing colors from training views."""
    W, H = args.img_wh
    K = np.array([[dataset.focal, 0, W/2],
                  [0, dataset.focal, H/2],
                  [0, 0, 1]]).astype(np.float32)
    
    N_vertices = len(vertices)
    vertices_homo = np.concatenate([vertices, np.ones((N_vertices, 1))], 1)
    
    v_color_sum = np.zeros((N_vertices, 3))
    non_occluded_sum = np.zeros((N_vertices, 1))
    
    for i in range(len(dataset)):
        sample = dataset[i]
        img = sample['rgb'] # Load and resize to img_wh
        P_c2w = sample['pose'] # Camera to world
        P_w2c = np.linalg.inv(P_c2w)[:3, :] # World to camera (3x4)
        camera_center = P_w2c[:, 3]
        
        # Project vertices
        vertices_cam = P_w2c @ vertices_homo.T
        vertices_cam[1:] *= -1 # Flip y, z
        
        vertices_image = (K @ vertices_cam).T
        depth = vertices_image[:, 2]
        u = vertices_image[:, 0] / (depth + 1e-6)
        v = vertices_image[:, 1] / (depth + 1e-6)
        
        # Clip to image bounds
        valid = (u >= 0) & (u < W-1) & (v >= 0) & (v < H-1)
        u = u[valid].astype(np.float32)
        v = v[valid].astype(np.float32)
        vertex_indices = np.where(valid)[0]
        
        if len(u) == 0:
            continue
            
        # Sample colors
        colors = cv2.remap(img, u.reshape(1,-1), v.reshape(1,-1), cv2.INTER_LINEAR)
        colors = colors.squeeze() # Shape (N_valid, 3)
        
        # Predict occlusion
        rays_o = camera_center[valid].unsqueeze(0).expand(len(u), -1, -1).reshape(-1, 3)
        rays_d = (vertices_cam[:, :3, valid].T - camera_center[valid].unsqueeze(0))
        rays_d = rays_d / rays_d.norm(dim=1, keepdim=True)
        
        # Query accumulated opacity
        accumulated_opacity = query_opacity(rays_o, rays_d, nerf_fine, embedding_xyz, embedding_dir, 
                                            args.N_samples, args.N_importance, args.chunk)
        non_occluded = 1.0 - accumulated_opacity.cpu().numpy()
        
        # Accumulate
        v_color_sum[vertex_indices] += colors * non_occluded
        non_occluded_sum[vertex_indices] += non_occluded
        
    # Normalize
    non_occluded_sum = np.maximum(non_occluded_sum, 1e-6)
    vertex_colors = v_color_sum / non_occluded_sum
    
    return vertex_colors
```

## Subtopics

### TSDF Estimation via NeRF Occupancy
A method to estimate the Truncated Signed Distance Field (TSDF) by exploiting NeRF occupancy aggregated along rays to determine points inside and outside the surface [S3]. It computes the median depth $\hat{z}_{i,p,50}$ as the first index where accumulated transmittance $T_k > 50/100$. It estimates points outside the surface by rendering depth at the 16th percentile $\hat{z}_{i,p,16}$ and inside at the 84th percentile $\hat{z}_{i,p,84}$ [S3]. These 3D coordinates are fed into an SSAN $\phi$ to estimate SDFs, normals, and features: $[\hat{t}_{50}, \hat{n}_{50}, \hat{f}_{50}] = \phi(r_{i,p}(\hat{z}_{i,p,50}))$, $[\hat{t}_{16}, \hat{n}_{16}, \hat{f}_{16}] = \phi(r_{i,p}(\hat{z}_{i,p,16}))$, $[\hat{t}_{84}, \hat{n}_{84}, \hat{f}_{84}] = \phi(r_{i,p}(\hat{z}_{i,p,84}))$ [S3].

### Nerfacto Point Cloud Extraction (Depth-based)
A method to extract 3D geometry from a NeRF by rendering depth maps and back-projecting them, used for real-world unbounded scenes [S4]. It renders depth maps for each input image using the expected ray termination in the density field and back-projects points to 3D coordinates using estimated camera poses [S4]. It uses a small neural network with hash encoding for computational efficiency [S4].

### Textured Mesh Extraction
Extracts a traditional textured mesh from the refined SuGaR model as a tool for visualization, composition and animation in Blender using a dedicated add-on [S5, unofficial]. It follows a pipeline of optimizing Gaussians alignment with the surface, extracting a mesh, refining Gaussians and mesh together, and finally exporting the textured mesh [S5, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** $N_{grid}=256$ provides a balance between resolution and computation time [S1, unofficial]. $\sigma_{threshold}=20.0$ is a typical value for occupancy [S1, unofficial].
- **Strengths:** Produces watertight meshes with consistent colors by leveraging multiple views [S1, unofficial].
- **Limitations:** Computationally expensive due to volumetric sampling and multiple NeRF queries [S1, unofficial]. Sensitive to the choice of $\sigma_{threshold}$ and $occ_{threshold}$ [S1, unofficial].

## Background
This method originates from the `nerf_pl` repository, which implements NeRF in PyTorch Lightning [S1, unofficial]. It builds on standard NeRF implementations [S1, unofficial] by adding mesh extraction and color fusion capabilities. The color fusion technique addresses the view-dependent nature of NeRF colors by aggregating information from all training views [S1, unofficial].

## Sources
[S1] kwea123/nerf_pl, https://github.com/kwea123/nerf_pl, fetched 2026-10-26 (unofficial)
[S2] Points2NeRF: Generating Neural Radiance Fields from 3D point cloud, https://doi.org/10.1016/j.patrec.2024.07.002, fetched 2026-10-29
[S3] NeRFMeshing: Distilling Neural Radiance Fields into Geometrically-Accurate 3D Meshes, https://doi.org/10.1109/3dv62453.2024.00093, fetched 2026-10-29
[S4] Vision through Obstacles—3D Geometric Reconstruction and Evaluation of Neural Radiance Fields (NeRFs), https://doi.org/10.3390/rs16071188, fetched 2026-10-30
[S5] Anttwo/SuGaR, https://github.com/Anttwo/SuGaR, fetched 2026-10-26 (unofficial)
