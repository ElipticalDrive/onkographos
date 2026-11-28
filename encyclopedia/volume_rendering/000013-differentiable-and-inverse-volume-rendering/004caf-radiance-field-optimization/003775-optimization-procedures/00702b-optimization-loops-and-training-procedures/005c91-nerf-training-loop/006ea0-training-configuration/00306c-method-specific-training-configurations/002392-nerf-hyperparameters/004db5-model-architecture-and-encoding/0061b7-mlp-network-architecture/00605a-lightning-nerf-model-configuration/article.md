# Lightning-NeRF Model Configuration

Also called: LightningNeRFModelConfig

## Overview
Lightning-NeRF is a configuration for efficient radiance field optimization that utilizes a hybrid representation combining a hash-encoded density grid with a lightweight MLP for color prediction [S1, unofficial]. It distinguishes itself from standard NeRF implementations by initializing the density field from point clouds and using a sparse hash grid for density, allowing for faster training and rendering on large-scale real-world scenes [S1, unofficial].

## How it works
The method defines the hyperparameters for the neural network architecture, rendering bounds, and optimization schedules.

**Inputs and Outputs**
The configuration class accepts parameters for network depth, hidden sizes, ray sampling bounds, and optimizer settings. It outputs the instantiated model and training pipeline configuration.

**Data Structures**
The model consists of:
1.  **Density Grid:** A hash-encoded grid initialized from a point cloud.
2.  **View-Independent MLP (VI-MLP):** Processes position features to predict density.
3.  **View-Dependent MLP (VD-MLP):** Processes direction features and VI-MLP outputs to predict RGB color.
4.  **Appearance Embedding:** Optional latent vector for appearance variations.

**Parameters**
*   `near_plane`: Start distance for ray sampling. Default: 0.01 [S1, unofficial].
*   `far_plane`: End distance for ray sampling. Default: 6.0 [S1, unofficial].
*   `vi_mlp_num_layers`: Layers in VI-MLP. Default: 3 [S1, unofficial].
*   `vi_mlp_hidden_size`: Hidden size of VI-MLP. Default: 64 [S1, unofficial].
*   `vd_mlp_num_layers`: Layers in VD-MLP. Default: 2 [S1, unofficial].
*   `vd_mlp_hidden_size`: Hidden size of VD-MLP. Default: 32 [S1, unofficial].
*   `appearance_embedding_dim`: Dimension of appearance embedding. 0 to disable. Default: 0 [S1, unofficial].
*   `use_average_appearance_embedding`: Use average embedding for inference. Default: True [S1, unofficial].
*   `background_color`: 'random', 'black', or 'white'. Default: 'random' [S1, unofficial].
*   `alpha_thre`: Opacity skip threshold. Default: 0.01 [S1, unofficial].
*   `cone_angle`: Ray marching cone angle. 0.0 for Blender, 1./256 for real scenes. Default: 0.004 [S1, unofficial].
*   `point_cloud_path`: Path to PCD file for initialization [S1, unofficial].
*   `frontal_axis`: 'x' or 'y' for background augmentation. Default: 'x' [S1, unofficial].
*   `init_density_value`: Initial density for occupied regions. Default: 10.0 [S1, unofficial].
*   `density_grid_base_res`: Base resolution for density grid. Default: 256 [S1, unofficial].
*   `density_log2_hashmap_size`: Log2 size of density hash map. Default: 24 [S1, unofficial].
*   `bg_density_grid_res`: Resolution for background density grid. Default: 32 [S1, unofficial].
*   `bg_density_log2_hashmap_size`: Log2 size of background hash map. Default: 18 [S1, unofficial].

**Optimization Configuration**
The training loop uses specific optimizers and schedulers for different components [S1, unofficial]:
*   **Density Encoder (`den_encoder`):** RAdam optimizer, lr=1.0. Exponential decay scheduler, warmup=10 steps, linear ramp, final lr=0.01, max steps=10,000.
*   **Color Encoder (`col_encoder`):** RAdam optimizer, lr=1.0. Exponential decay scheduler, warmup=10 steps, linear ramp, final lr=0.01, max steps=10,000.
*   **Network (`network`):** Adam optimizer, lr=1e-2, eps=1e-15. Exponential decay scheduler, final lr=0.0001, max steps=30,000.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def configure_lightning_nerf():
    # Define model hyperparameters
    model_config = {
        "near_plane": 0.01,
        "far_plane": 6.0,
        "vi_mlp_num_layers": 3,
        "vi_mlp_hidden_size": 64,
        "vd_mlp_num_layers": 2,
        "vd_mlp_hidden_size": 32,
        "appearance_embedding_dim": 0,
        "use_average_appearance_embedding": True,
        "background_color": "random",
        "alpha_thre": 0.01,
        "cone_angle": 0.004,
        "point_cloud_path": None, # Optional
        "frontal_axis": "x",
        "init_density_value": 10.0,
        "density_grid_base_res": 256,
        "density_log2_hashmap_size": 24,
        "bg_density_grid_res": 32,
        "bg_density_log2_hashmap_size": 18,
        "eval_num_rays_per_chunk": 1 << 17
    }

    # Define optimizer configurations
    optimizers = {
        "den_encoder": {
            "optimizer": {
                "type": "RAdam",
                "lr": 1.0
            },
            "scheduler": {
                "type": "ExponentialDecay",
                "warmup_steps": 10,
                "ramp": "linear",
                "lr_final": 0.01,
                "max_steps": 10000
            }
        },
        "col_encoder": {
            "optimizer": {
                "type": "RAdam",
                "lr": 1.0
            },
            "scheduler": {
                "type": "ExponentialDecay",
                "warmup_steps": 10,
                "ramp": "linear",
                "lr_final": 0.01,
                "max_steps": 10000
            }
        },
        "network": {
            "optimizer": {
                "type": "Adam",
                "lr": 1e-2,
                "eps": 1e-15
            },
            "scheduler": {
                "type": "ExponentialDecay",
                "lr_final": 0.0001,
                "max_steps": 30000
            }
        }
    }

    # Define training loop parameters
    training_config = {
        "max_num_iterations": 30001,
        "steps_per_eval_batch": 500,
        "steps_per_save": 1000,
        "steps_per_eval_image": 30000,
        "steps_per_eval_all_images": 30000,
        "mixed_precision": True,
        "datamanager": {
            "train_num_rays_per_batch": 65536,
            "eval_num_rays_per_batch": 2048
        },
        "viewer": {
            "num_rays_per_chunk": 1 << 15
        }
    }

    return model_config, optimizers, training_config
```

**Edge Cases and Pitfalls**
*   **Scene Scale:** The `cone_angle` parameter must be adjusted based on the scene type. It should be 0.0 for synthetic Blender scenes but set to 1./256 for real-world scenes to handle scale correctly [S1, unofficial].
*   **Background Clipping:** When calculating the foreground region (AABB) from camera information, the height (z-axis) of view frustums should be clipped to a minimum value of -5m in world coordinates to avoid wasting space on underground areas [S1, unofficial].
*   **Initialization:** If `point_cloud_path` is provided, the density grid is initialized using this data. If not, the behavior is not explicitly defined in the sources [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** The configuration is designed for efficient training and rendering, particularly for autonomous driving scenes like KITTI-360 and Argoverse2 [S1, unofficial].
*   **Strengths:** Fast training due to hash encoding and lightweight MLP.
*   **Limitations:** Not given in the sources.
*   **When to Use:** Large-scale real-world scenes where efficiency is critical.
*   **When Not to Use:** Synthetic scenes with simple geometry where standard NeRF may suffice.

## Background
Lightning-NeRF was developed by VISION-SJTU. The configuration class is part of the Nerfstudio plugin ecosystem, allowing for easy integration into the training pipeline [S1, unofficial]. It is specifically tailored for autonomous driving datasets.

## Sources
[S1] Lightning-NeRF, https://github.com/VISION-SJTU/Lightning-NeRF, fetched 2026-10-29 (unofficial)
