# SDF and Material Optimization Pipeline

Also called: SDF training loop, relaxed boundary optimization

## Overview
This method optimizes a voxelized signed distance field (SDF) and associated material volumes (albedo, roughness, metallic) by minimizing the L1 difference between rendered images and reference images [S1, unofficial]. It differs from pure geometry optimization by jointly updating material grids, and from neural SDF methods by using explicit grid-based representations with periodic upsampling [S1, unofficial]. It builds on [SDF Evaluation Implementation](1658-sdf-evaluation-implementation.md) for the forward rendering pass [S1, unofficial].

## How it works
**Inputs:**
- Scene definition containing geometry and initial material volumes [S1, unofficial].
- Reference images rendered from multiple sensor viewpoints [S1, unofficial].
- Configuration parameters: `sdf_res`, `sphere_radius`, `lr`, `batch_size`, `spp`, `spp_grad`, `laplacian_weight`, `lr_decay`, `upsample` epochs, `lr_downsample` epochs, `checkpoint` interval [S1, unofficial].

**Outputs:**
- Optimized SDF grid stored as `.npy` [S1, unofficial].
- Optimized material grids (albedo, roughness, metallic) stored as `.npy` and `.vol` [S1, unofficial].
- Loss curves and intermediate optimization videos [S1, unofficial].

**Data Structures:**
- **SDF Grid:** 3D tensor of shape `(res, res, res)` representing the signed distance field. Initialized as a sphere [S1, unofficial].
- **Material Grids:** 3D tensors for `base_color` (albedo), `roughness`, and `metallic`. Resampled to match `sdf_res` [S1, unofficial].
- **Optimizer State:** Adam optimizer state for the SDF grid and material grids [S1, unofficial].
- **Checkpoint Format:** 1D array where the first element is the resolution `res`, followed by the flattened grid values `grid.flatten()` [S1, unofficial].

**Parameters:**
- `lr`: Base learning rate for Adam [S1, unofficial].
- `lr_albedo_scale`, `lr_roughness_scale`, `lr_metallic_scale`: Multipliers for material learning rates relative to `lr` [S1, unofficial].
- `laplacian_weight`: Weight for the Laplacian regularization term [S1, unofficial].
- `lr_decay`: Factor by which `lr` is divided during upsample/lr_downsample epochs [S1, unofficial].
- `batch_size`: Number of sensor images processed per epoch [S1, unofficial].
- `sdf_res`: Target resolution for SDF and material volumes [S1, unofficial].
- `spp`, `spp_grad`: Samples per pixel for rendering and gradient estimation [S1, unofficial].

**Steps:**

1.  **Initialization:**
    - Load the scene and sensors [S1, unofficial].
    - Render reference images using a non-differentiable integrator [S1, unofficial].
    - Initialize the SDF as a `SphereSDF` with resolution `sdf_res` and radius `sphere_radius` [S1, unofficial].
    - Enable gradients on the SDF grid tensor [S1, unofficial].
    - Traverse scene parameters to identify material volumes (`base_color`, `reflectance`, `roughness`, `metallic`) [S1, unofficial].
    - If material volumes exist, upsample their resolution to `sdf_res` using `upsample_to` and update the scene parameters [S1, unofficial].
    - Configure the Adam optimizer with base learning rate `lr` [S1, unofficial].
    - Add the SDF grid to the optimizer [S1, unofficial].
    - If materials are enabled, add their grids to the optimizer and set per-parameter learning rates:
        - `lr_albedo = lr * lr_albedo_scale`
        - `lr_roughness = lr * lr_roughness_scale`
        - `lr_metallic = lr * lr_metallic_scale` [S1, unofficial].

2.  **Optimization Loop:**
    - Calculate steps per epoch: `steps = ceil(num_sensor / batch_size)` [S1, unofficial].
    - For each epoch:
        - Calculate number of indices: `num_indices = min(batch_size, num_sensor)` [S1, unofficial].
        - Generate indices: `indices = [(j * steps + epoch % steps) % num_sensor for j in range(num_indices)]` [S1, unofficial].
        - Shuffle `indices` [S1, unofficial].
        - Initialize `loss_sum = 0` [S1, unofficial].
        - For each index `i` in `indices`:
            - Render image using the SDF integrator with `spp`, `spp_grad`, and seed `epoch * num_sensor + i` [S1, unofficial].
            - Compute L1 loss: `loss = l1_loss(image, ref_images[i]) / batch_size` [S1, unofficial].
            - Accumulate loss: `loss_sum += loss` [S1, unofficial].
            - Backpropagate: `dr.backward(loss)` [S1, unofficial].
        - Compute regularization loss: `reg_loss = laplacian_weight * laplacian_loss(sdf.grid.tensor())` [S1, unofficial].
        - Backpropagate regularization: `dr.backward(reg_loss)` [S1, unofficial].
        - Update parameters: `optimizer.step()` [S1, unofficial].
        - Redistance the SDF grid: `grid = redistance(optimizer['grid'], method='fastsweep')` [S1, unofficial].
        - Convert grid to 4D tensor: `grid = atleast_4d(TensorXf(grid))` [S1, unofficial].
        - Update optimizer grid: `optimizer['grid'] = grid` [S1, unofficial].
        - Update scene parameters with optimizer state [S1, unofficial].
        - If `epoch` in `upsample` list:
            - Decay learning rate: `lr /= lr_decay` [S1, unofficial].
            - Update optimizer learning rates for grid and materials [S1, unofficial].
            - Upsample SDF: `upsample_sdf` [S1, unofficial].
            - Upsample materials: `upsample_grid` (only if `resolution != 256`) [S1, unofficial].
        - If `epoch` in `lr_downsample` list:
            - Decay learning rate: `lr /= lr_decay` [S1, unofficial].
            - Update optimizer learning rates for grid and materials [S1, unofficial].
        - If `epoch % checkpoint == 0`:
            - Save checkpoint: flatten grid, prepend resolution, save as `[resolution] + grid.tolist()` to `.npy` [S1, unofficial].
            - Repeat for materials if enabled [S1, unofficial].

3.  **Finalization:**
    - Save final SDF and material grids in `[resolution] + grid.tolist()` format to `.npy` [S1, unofficial].
    - Write material grids to `.vol` files via `mi.VolumeGrid` [S1, unofficial].
    - Generate optimization video (24 FPS) from `opt_dir` [S1, unofficial].
    - Plot L1 loss over epochs and save as `loss.pdf`, `loss.png`, `loss.npy` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import os
import numpy as np
import drjit as dr

# Self-contained helper definitions
def get_h2_sensors(num_theta, num_phi, resx, resy, radius, height):
    return [type('Sensor', (), {'id': i})() for i in range(num_theta * num_phi)]

class SphereSDF:
    def __init__(self, res, radius):
        self.grid = dr.zeros((res, res, res), dtype=dr.float32)
        coords = np.mgrid[-1:1:res*1j, -1:1:res*1j, -1:1:res*1j]
        dist = np.sqrt(np.sum(coords**2, axis=0))
        self.grid[:] = (dist - radius) * 2.0
    def tensor(self):
        return self.grid

class Adam:
    def __init__(self, lr):
        self.lr = lr
        self.state = {}
    def __setitem__(self, key, val):
        self.state[key] = val
    def set_learning_rate(self, mapping):
        for k, v in mapping.items():
            self.state[k] = v
    def step(self):
        for k, v in self.state.items():
            v[:] -= self.lr * dr.grad(v)

def upsample_to(tensor, target_res):
    return dr.interpolate(tensor, target_res)

def upsample_sdf(sdf, grid):
    sdf.grid[:] = upsample_to(grid, sdf.grid.shape[0])

def upsample_grid(tensor):
    return upsample_to(tensor, tensor.shape[0] * 2)

def redistance(grid, method):
    return dr.abs(grid) * dr.sign(grid)

def atleast_4d(tensor):
    return dr.reshape(tensor, (1, *tensor.shape))

def TensorXf(tensor):
    return tensor

def laplacian_loss(grid):
    kernel = np.zeros((3,3,3))
    kernel[1,1,1] = -6
    kernel[1,1,0] = kernel[1,1,2] = kernel[1,0,1] = kernel[1,2,1] = kernel[0,1,1] = kernel[2,1,1] = 1
    return dr.sum(dr.convolve(grid, kernel)**2)

def l1_loss(a, b):
    return dr.sum(dr.abs(a - b))

def save_checkpoint(exp_dir, epoch, sdf, scene_params, opt_albedo, opt_roughness, opt_metallic, albedo_name, roughness_name, metallic_name):
    sdf_dir = os.path.join(exp_dir, "params")
    os.makedirs(sdf_dir, exist_ok=True)
    grid = sdf.grid.tensor()
    res = grid.shape[0]
    data = [res] + grid.flatten().tolist()
    np.save(os.path.join(sdf_dir, f"sdf-{epoch}.npy"), data)
    if opt_albedo:
        data = [res] + scene_params[albedo_name].flatten().tolist()
        np.save(os.path.join(sdf_dir, f"albedo-{epoch}.npy"), data)
    if opt_roughness:
        data = [res] + scene_params[roughness_name].flatten().tolist()
        np.save(os.path.join(sdf_dir, f"roughness-{epoch}.npy"), data)
    if opt_metallic:
        data = [res] + scene_params[metallic_name].flatten().tolist()
        np.save(os.path.join(sdf_dir, f"metallic-{epoch}.npy"), data)

def save_final(exp_dir, sdf, scene_params, opt_albedo, opt_roughness, opt_metallic, albedo_name, roughness_name, metallic_name):
    sdf_dir = os.path.join(exp_dir, "params")
    os.makedirs(sdf_dir, exist_ok=True)
    grid = sdf.grid.tensor()
    res = grid.shape[0]
    data = [res] + grid.flatten().tolist()
    np.save(os.path.join(sdf_dir, "sdf-final.npy"), data)
    if opt_albedo:
        data = [res] + scene_params[albedo_name].flatten().tolist()
        np.save(os.path.join(sdf_dir, "albedo-final.npy"), data)
    if opt_roughness:
        data = [res] + scene_params[roughness_name].flatten().tolist()
        np.save(os.path.join(sdf_dir, "roughness-final.npy"), data)
    if opt_metallic:
        data = [res] + scene_params[metallic_name].flatten().tolist()
        np.save(os.path.join(sdf_dir, "metallic-final.npy"), data)

def generate_videos(opt_dir):
    pass

def plot_loss(losses, exp_dir):
    pass

def optimize_sdf_and_materials(exp_config, render_config, diff_config, scene_path, out_dir):
    exp_dir = os.path.join(out_dir, exp_config["exp_name"])
    os.makedirs(exp_dir, exist_ok=True)

    scene = mi.load_file(scene_path, HIDE_ENV=render_config["hide_env"])
    sensors = get_h2_sensors(
        num_sensor_theta=exp_config["num_sensor_theta"],
        num_sensor_phi=exp_config["num_sensor_phi"],
        resx=render_config["resx"], resy=render_config["resy"],
        radius=1.3, height=0.4
    )
    num_sensor = len(sensors)

    ref_dir = os.path.join(exp_dir, "ref")
    os.makedirs(ref_dir, exist_ok=True)
    ref_integrator = mi.load_dict({"type": render_config["ref_integrator"]})
    ref_images = []
    with dr.suspend_grad():
        for i in range(num_sensor):
            ref_image = ref_integrator.render(scene=scene, sdf=SphereSDF(1, 1), sensor=sensors[i], spp=render_config["ref_spp"])
            ref_images.append(ref_image)

    sdf = SphereSDF(res=exp_config["sdf_res"], radius=exp_config["sphere_radius"])
    dr.enable_grad(sdf.grid.tensor())

    params = mi.traverse(scene)
    opt_albedo = 'main-bsdf.base_color.volume.data' in params or 'main-bsdf.reflectance.volume.data' in params
    opt_roughness = 'main-bsdf.roughness.volume.data' in params
    opt_metallic = 'main-bsdf.metallic.volume.data' in params

    scene_params = params
    if opt_albedo:
        albedo_name = 'main-bsdf.base_color.volume.data' if 'main-bsdf.base_color.volume.data' in params else 'main-bsdf.reflectance.volume.data'
        scene_params[albedo_name] = upsample_to(scene_params[albedo_name], exp_config["sdf_res"])
        scene_params.update()

    if opt_roughness:
        roughness_name = 'main-bsdf.roughness.volume.data'
        scene_params[roughness_name] = upsample_to(scene_params[roughness_name], exp_config["sdf_res"])
        scene_params.update()

    if opt_metallic:
        metallic_name = 'main-bsdf.metallic.volume.data'
        scene_params[metallic_name] = upsample_to(scene_params[metallic_name], exp_config["sdf_res"])
        scene_params.update()

    optimizer = Adam(lr=exp_config["lr"])
    optimizer['grid'] = sdf.grid.tensor()

    if opt_albedo:
        optimizer[albedo_name] = scene_params[albedo_name]
        optimizer.set_learning_rate({albedo_name: exp_config["lr"] * exp_config["lr_albedo_scale"]})

    if opt_roughness:
        optimizer[roughness_name] = scene_params[roughness_name]
        optimizer.set_learning_rate({roughness_name: exp_config["lr"] * exp_config["lr_roughness_scale"]})

    if opt_metallic:
        optimizer[metallic_name] = scene_params[metallic_name]
        optimizer.set_learning_rate({metallic_name: exp_config["lr"] * exp_config["lr_metallic_scale"]})

    sdf_integrator = mi.load_dict({"type": render_config["sdf_integrator"]})

    steps = int(np.ceil(num_sensor / exp_config["batch_size"]))
    losses = []
    lr = exp_config["lr"]

    for epoch in range(exp_config["num_epoch"]):
        loss_sum = dr.Float(0)
        num_indices = min(exp_config["batch_size"], num_sensor)
        indices = [(j * steps + epoch % steps) % num_sensor for j in range(num_indices)]
        np.random.shuffle(indices)

        for i in indices:
            seed = epoch * num_sensor + i
            image = sdf_integrator.render(
                scene=scene, sdf=sdf, sensor=sensors[i],
                spp=render_config["spp"], spp_grad=render_config["spp_grad"],
                seed=seed, params=scene_params
            )
            loss = l1_loss(image, ref_images[i]) / exp_config["batch_size"]
            loss_sum += loss
            dr.backward(loss)

        reg_loss = exp_config["laplacian_weight"] * laplacian_loss(sdf.grid.tensor())
        loss_sum += reg_loss
        dr.backward(reg_loss)

        optimizer.step()

        grid = redistance(optimizer['grid'], method='fastsweep')
        grid = atleast_4d(TensorXf(grid))
        optimizer['grid'] = grid

        sdf_params.update(optimizer)
        scene_params.update(optimizer)

        if epoch in exp_config["upsample"]:
            lr /= exp_config["lr_decay"]
            optimizer.set_learning_rate({'grid': lr})
            if opt_albedo:
                optimizer.set_learning_rate({albedo_name: lr * exp_config["lr_albedo_scale"]})
            if opt_roughness:
                optimizer.set_learning_rate({roughness_name: lr * exp_config["lr_roughness_scale"]})
            if opt_metallic:
                optimizer.set_learning_rate({metallic_name: lr * exp_config["lr_metallic_scale"]})

            upsample_sdf(sdf, optimizer['grid'])
            if exp_config["sdf_res"] != 256:
                if opt_albedo:
                    scene_params[albedo_name] = upsample_grid(scene_params[albedo_name])
                if opt_roughness:
                    scene_params[roughness_name] = upsample_grid(scene_params[roughness_name])
                if opt_metallic:
                    scene_params[metallic_name] = upsample_grid(scene_params[metallic_name])
                scene_params.update()

        if epoch in exp_config["lr_downsample"]:
            lr /= exp_config["lr_decay"]
            optimizer.set_learning_rate({'grid': lr})
            if opt_albedo:
                optimizer.set_learning_rate({albedo_name: lr * exp_config["lr_albedo_scale"]})
            if opt_roughness:
                optimizer.set_learning_rate({roughness_name: lr * exp_config["lr_roughness_scale"]})
            if opt_metallic:
                optimizer.set_learning_rate({metallic_name: lr * exp_config["lr_metallic_scale"]})

        if epoch % exp_config["checkpoint"] == 0:
            save_checkpoint(exp_dir, epoch, sdf, scene_params, opt_albedo, opt_roughness, opt_metallic, albedo_name, roughness_name, metallic_name)

        losses.append(loss_sum)

    save_final(exp_dir, sdf, scene_params, opt_albedo, opt_roughness, opt_metallic, albedo_name, roughness_name, metallic_name)
    generate_videos(exp_dir)
    plot_loss(losses, exp_dir)
```

**Pitfalls:**
- Upsampling materials is skipped if `resolution == 256` to prevent redundant operations or artifacts at the target resolution [S1, unofficial].
- Learning rate decay is applied in two separate phases: during explicit upsample epochs and near the end via `lr_downsample` epochs, ensuring stable convergence [S1, unofficial].
- Precision and boundary conditions are not explicitly detailed in the sources [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Dominated by rendering per epoch. Each epoch processes `batch_size` images. Total epochs is `num_epoch`. Rendering cost depends on `spp`, `spp_grad`, and scene complexity [S1, unofficial].
- **Memory Complexity:** Proportional to `sdf_res^3` for the SDF grid and material grids [S1, unofficial].
- **Strengths:** Jointly optimizes geometry and materials. Uses explicit grids for easy manipulation and upsampling [S1, unofficial].
- **Limitations:** Resolution is fixed until upsample epochs. Upsampling materials is skipped at `resolution == 256` to prevent artifacts [S1, unofficial]. Learning rate decay is critical for convergence [S1, unofficial].
- **When to use:** When reference images are available and both geometry and material properties need to be recovered [S1, unofficial].
- **When not to use:** When no reference images are available or when neural representations are preferred for higher resolution without upsampling [S1, unofficial].

## Background
This method originates from the `zichenwang01/relaxed-boundary` repository [S1, unofficial]. It was developed by Zichen Wang and integrates material optimization into the SDF training loop [S1, unofficial]. It builds on earlier work in SDF rendering and optimization [S1, unofficial].

## Sources
[S1] zichenwang01/relaxed-boundary, https://github.com/zichenwang01/relaxed-boundary, fetched 2026-10-27 (unofficial)
