# PointCloud Core

NumPy primitives for validating point clouds, aggregating points into voxels,
and mapping voxel results back to the original points. No GPU, training
framework, command-line interface, or dataset format is required.

Version `0.1.0` is prepared for the first release. The source is public; the
package has not yet been published to PyPI. Install from a local checkout
while the publisher is being configured. See the
[release notes](https://github.com/ajavanma/pointcloud-core/blob/main/docs/releases/0.1.0.md)
and [API compatibility policy](https://github.com/ajavanma/pointcloud-core/blob/main/docs/api.md#compatibility-policy).

```sh
python -m venv .venv
source .venv/bin/activate
python -m pip install .
```

Python 3.11 or later and NumPy 1.26.4 or later (below 3) are required. CI covers
Python 3.11 and 3.12 with NumPy 1.26.4 and the latest NumPy 2.x.

## Example

```python
import numpy as np
from pointcloud_core import LabelMap, PointCloud, lift_voxel_values, voxelize

classes = LabelMap((7, 42), ignore_label=255)
cloud = PointCloud(
    coords=[[0.1, 0, 0], [0.2, 0, 0], [1.1, 0, 0], [np.nan, 0, 0]],
    features=[[2], [4], [8], [10]],
    labels=classes.encode([7, 7, 42, 255]),
    point_indices=[100, 105, 900, 901],
)
voxels = voxelize(cloud, voxel_size=1.0)
assert voxels.voxel_coords.tolist() == [[0, 0, 0], [1, 0, 0]]
assert voxels.features.tolist() == [[3.0], [8.0]]
assert voxels.point_to_voxel.tolist() == [0, 0, 1, -1]

# A model or another algorithm can supply one value per voxel.
encoded_predictions = np.array([0, 1], dtype=np.int64)
point_predictions = classes.decode(
    lift_voxel_values(encoded_predictions, voxels.point_to_voxel)
)
assert point_predictions.tolist() == [7, 7, 42, 255]
assert voxels.point_indices.tolist() == [100, 105, 900, 901]
```

Every output row maps back to an input row. Original point IDs remain separate
from row positions. Invalid coordinate/feature rows receive mapping `-1`, so
they cannot accidentally index the final voxel during reconstruction.

## Numerical contract

- Coordinates use floor division by a scalar voxel size in the caller's units.
  Negative coordinates are handled by floor; no origin offset or boundary
  epsilon is applied.
- Voxels are sorted lexicographically. Features are arithmetic means. Labels
  use the most frequent nonignored value, with the smallest numeric label on
  ties. Use `LabelMap` first when declared class order should break ties.
- Coordinates/features normalize to float64; labels, coordinates of voxels,
  mappings, and point IDs use int64. Inputs are not mutated.
- Coordinates and features share one finite mask. At least one finite row is
  required. Input labels are validated even on excluded rows.
- Operations hold arrays in memory. This is not a streaming or out-of-core
  implementation, and floating reductions are not promised to be bit-identical
  across arbitrary input permutations or NumPy versions.

See the [API reference](https://github.com/ajavanma/pointcloud-core/blob/main/docs/api.md),
[contribution guide](https://github.com/ajavanma/pointcloud-core/blob/main/CONTRIBUTING.md), and
[release plan](https://github.com/ajavanma/pointcloud-core/blob/main/docs/releasing.md).

## Relationship to PointCloud Workflows

The numerical methods were extracted and generalized from
[PointCloud Workflows](https://github.com/ajavanma/PointCloud-Workflows).
PointCloud Core owns only in-memory array operations. PCW owns dataset
manifests, file readers, model execution, checkpoints, and run artifacts.

PCW uses Core through a compatibility adapter, pinned to the prepared `0.1.0`
source revision `5c28a700c30aee6186cfe30f6cd74708289f0743` with an archive hash
while the first PyPI release is pending. The adapter preserves PCW's validation
errors, artifact schemas, and historical divide-before-sum feature means. Direct
Core callers use the scaled-mean reduction documented in the API reference.
See the [integration guide](https://github.com/ajavanma/PointCloud-Workflows/blob/main/docs/core-integration.md)
and [standalone Core example](https://github.com/ajavanma/PointCloud-Workflows/blob/main/examples/core_roundtrip.py).
The example needs only NumPy and Core and demonstrates preserved point IDs,
ignored labels, invalid rows, and prediction lifting.

The original MIT attribution is retained in
[LICENSE](https://github.com/ajavanma/pointcloud-core/blob/main/LICENSE).
