# Point Sampling State Preparation


## Overview
Extracts mesh geometry, materials, and textures from a target collection to build the state required for point sampling [S1, unofficial]. It serves as the initialization phase for generating point clouds from mesh surfaces, ensuring that all necessary data (triangles, materials, textures) is loaded and cached before the actual sampling execution begins [S1, unofficial]. It distinguishes itself by managing the lifecycle of the sampling state, including caching, asynchronous worker spawning, and memory release, rather than performing the geometric sampling itself.

## How it works
**Inputs:**
- `session`: A build session object containing `point_state`, `diagnostics`, `camera_info`, `target_collection`, `frames`, `scene`, `render_state`, `temp_camera_obj`, `output_dir`, `warnings`, and `snapshot`.
- `context`: Execution context (implied).
- `collection`: The target collection containing mesh data.
- `warnings`: List to accumulate warnings.
- `warning_keys`: Set of keys for warning deduplication.
- `sample_count`: Integer specifying the number of points to sample.
- `diagnostics`: Diagnostic logger.

**Outputs:**
- Updates `session.point_state` with the prepared `PointSamplingState`.
- Returns the prepared state or triggers asynchronous/synchronous execution depending on configuration.

**Data Structures:**

`PointSamplingState` layout:
- `triangles`: `list[PlainTriangleData]`
- `cumulative_areas`: `array` (likely numpy array or list of floats)
- `total_area`: `float`
- `sample_count`: `int`
- `materials`: `list[PlainMaterialData]`
- `images`: `list[PlainImageData]`
- `random_seed`: `int`
- `texture_count`: `int`
- `texture_bytes`: `int`
- `fallback_triangle_count`: `int` (default 0)
- `fallback_material_triangle_counts`: `dict[str, int]`
- `points`: `list[PointRecord]` (default `[]`)
- `sampled_count`: `int` (default 0)
- `worker_state`: `PointSamplingWorkerState | None` (default `None`)

`PointSamplingWorkerState` layout:
- `process`: `object | None` (subprocess handle)
- `temp_dir`: `Path`
- `task_path`: `Path`
- `progress_path`: `Path`
- `result_path`: `Path`
- `error_path`: `Path`
- `stderr_path`: `Path`

**Steps:**

1.  **Cache Check:** Check if `session.point_state` is not `None`. If it exists, return it immediately (cache hit) [S1, unofficial].
2.  **Diagnostics:** Log diagnostics info about preparing point sampling state [S1, unofficial].
3.  **Preparation:** Call `_prepare_point_sampling_state` with `context`, `collection`, `warnings`, `warning_keys`, `sample_count`, and `diagnostics` [S1, unofficial].
4.  **Assignment:** Assign the result of the preparation step to `session.point_state` [S1, unofficial].
5.  **Execution Decision:** The method may subsequently trigger either asynchronous or synchronous execution based on the presence of `worker_state` or configuration flags (see Subtopics).

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def prepare_point_sampling_state(session, context, collection, warnings, warning_keys, sample_count, diagnostics):
    # Step 1: Cache Check
    if session.point_state is not None:
        return session.point_state

    # Step 2: Diagnostics
    diagnostics.info("Preparing point sampling state.")

    # Step 3: Core Preparation
    # This function extracts mesh geometry, materials, and textures.
    # It populates PointSamplingState with triangles, cumulative_areas, total_area, etc.
    point_state = _prepare_point_sampling_state(
        context=context,
        collection=collection,
        warnings=warnings,
        warning_keys=warning_keys,
        sample_count=sample_count,
        diagnostics=diagnostics
    )

    # Step 4: Assignment
    session.point_state = point_state

    return point_state
```

**Edge Cases and Pitfalls:**
- **Cache Hit:** If `point_state` is already initialized, the method returns immediately, avoiding redundant computation [S1, unofficial].
- **Missing State:** If execution is attempted before preparation, `session.point_state` may be `None`, leading to errors in downstream steps [S1, unofficial].
- **Memory Management:** In asynchronous mode, inputs are released after worker spawn to free memory [S1, unofficial].

## Subtopics

### Asynchronous Point Sampling Worker
Spawns a subprocess to perform point sampling asynchronously, passing task data via pickle and tracking progress/error via files [S1, unofficial].

**Steps:**
1.  Check if `session.point_state` is `None`; if so, raise `DatasetBuildError` [S1, unofficial].
2.  Check if `point_state.worker_state` is not `None`; if so, return immediately (worker already started) [S1, unofficial].
3.  Create a temporary directory with prefix `"three_dgs_point_sampling_"` [S1, unofficial].
4.  Define paths: `task_path`, `progress_path`, `result_path`, `error_path`, `stderr_path` inside `temp_dir` [S1, unofficial].
5.  Serialize task data to `task_path` using `_write_pickle` [S1, unofficial].
6.  Resolve python executable and worker script path (`core/point_sampling_worker.py`) [S1, unofficial].
7.  Configure `worker_env` with `PYTHONPATH` including `addon_parent` [S1, unofficial].
8.  Open `stderr_path` for writing and start subprocess with `Popen`, passing executable, worker script, and file paths as arguments [S1, unofficial].
9.  On success, close `stderr_handle`, create `PointSamplingWorkerState` with process and paths, assign to `point_state.worker_state` [S1, unofficial].
10. Call `_release_point_sampling_inputs(point_state)` to free memory [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def start_async_worker(session, point_state):
    if session.point_state is None:
        raise DatasetBuildError("Point state not prepared.")
    
    if point_state.worker_state is not None:
        return point_state.worker_state

    temp_dir = tempfile.mkdtemp(prefix="three_dgs_point_sampling_")
    task_path = temp_dir / "task.pkl"
    progress_path = temp_dir / "progress.pkl"
    result_path = temp_dir / "result.pkl"
    error_path = temp_dir / "error.pkl"
    stderr_path = temp_dir / "stderr.txt"

    _write_pickle(task_path, _build_point_sampling_task_data(point_state))

    python_executable = _resolve_python_executable()
    worker_script = Path(__file__).resolve().parent / "core" / "point_sampling_worker.py"
    
    worker_env = os.environ.copy()
    addon_parent = str(Path(__file__).resolve().parent.parent)
    existing_pythonpath = worker_env.get("PYTHONPATH", "")
    worker_env["PYTHONPATH"] = (
        addon_parent if not existing_pythonpath else os.pathsep.join((addon_parent, existing_pythonpath))
    )

    stderr_handle = stderr_path.open("w", encoding="utf-8")
    session.diagnostics.info(
        f"Starting point sampling worker (task_dir={temp_dir}, samples={point_state.sample_count}, peak_rss={format_peak_rss_mb()})."
    )
    
    process = subprocess.Popen(
        [
            python_executable,
            str(worker_script),
            str(task_path),
            str(progress_path),
            str(result_path),
            str(error_path),
        ],
        stdout=subprocess.DEVNULL,
        stderr=stderr_handle,
        env=worker_env
    )

    stderr_handle.close()
    point_state.worker_state = PointSamplingWorkerState(
        process=process,
        temp_dir=temp_dir,
        task_path=task_path,
        progress_path=progress_path,
        result_path=result_path,
        error_path=error_path,
        stderr_path=stderr_path
    )
    
    _release_point_sampling_inputs(point_state)
    return point_state.worker_state
```

### Synchronous Point Sampling Execution
Executes point sampling synchronously using the prepared state, returning the total number of sampled points [S1, unofficial].

**Steps:**
1.  Check if `session.point_state` is `None`; if so, raise `DatasetBuildError` [S1, unofficial].
2.  Assign `session.point_state` to local `point_state` variable [S1, unofficial].
3.  Check if `point_state.points` is non-empty; if so, return `len(point_state.points)` immediately [S1, unofficial].
4.  Log diagnostics info about starting synchronous point sampling [S1, unofficial].
5.  Call `sample_points` with `_build_point_sampling_task_data(point_state)` and assign result to `point_state.points` [S1, unofficial].
6.  Update `point_state.sampled_count` to `len(point_state.points)` [S1, unofficial].
7.  Log diagnostics info about completed sampling and return `point_state.sampled_count` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def execute_synchronous_sampling(session, point_state):
    if session.point_state is None:
        raise DatasetBuildError("Point state not prepared.")
    
    point_state = session.point_state
    
    if len(point_state.points) > 0:
        return len(point_state.points)

    session.diagnostics.info("Starting synchronous point sampling.")
    
    task_data = _build_point_sampling_task_data(point_state)
    point_state.points = sample_points(task_data)
    
    point_state.sampled_count = len(point_state.points)
    
    session.diagnostics.info(
        f"Completed sampling. Sampled count: {point_state.sampled_count}."
    )
    
    return point_state.sampled_count
```

## Operating characteristics
- **Time Complexity:** Preparation is $O(N)$ where $N$ is the number of triangles in the mesh, as it iterates to extract geometry and compute cumulative areas. Asynchronous spawning is $O(1)$ relative to sampling work, delegating the heavy lifting to the subprocess. Synchronous execution is $O(S)$ where $S$ is the sample count, involving random sampling over the triangle distribution.
- **Memory Complexity:** $O(N)$ for storing triangle data, cumulative areas, and materials. Asynchronous mode reduces peak memory in the main process by releasing inputs after spawning the worker [S1, unofficial].
- **Performance:** Depends heavily on mesh size and sample count. Asynchronous mode allows the main process to continue or manage other tasks while sampling occurs in the background.
- **Strengths:** Caching prevents redundant preparation. Asynchronous mode enables memory efficiency and non-blocking execution.
- **Limitations:** Synchronous mode blocks the main thread. Asynchronous mode requires inter-process communication overhead and file I/O.
- **When to Use:** Use asynchronous mode for large datasets or when integrating into a larger pipeline that needs to remain responsive. Use synchronous mode for small datasets or debugging.

## Background
Originated in the `3dgs-dataset-builder` project by yumehiko [S1, unofficial]. It is part of the pipeline for generating 3D Gaussian Splatting datasets from mesh collections, specifically handling the transition from mesh geometry to point cloud representation.

## Sources
[S1] yumehiko/3dgs-dataset-builder, https://github.com/yumehiko/3dgs-dataset-builder, fetched 2026-10-27 (unofficial)
