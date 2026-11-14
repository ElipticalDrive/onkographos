# LOD Download Manager


## Overview
The LOD Download Manager is a specialized component within the Reall3dViewer ecosystem that orchestrates the asynchronous, priority-based downloading of 3D Gaussian Splat data organized in a Level-of-Detail (LOD) hierarchy [S1, unofficial]. It solves the problem of efficiently streaming large-scale splat clouds by fetching metadata, resolving file URLs, and managing concurrent network requests based on camera proximity and device constraints [S1, unofficial]. It distinguishes itself from general splat cloud loading by specifically handling the hierarchical LOD structure, calculating active LOD targets based on device type (mobile/PC), and enforcing a strict download queue prioritization logic [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** 'TodoDownloadLod' event payload containing splat file identifiers; 'LodDownloadManagerAddLodMeta' event payload containing LOD metadata JSON; camera position for distance calculations.
*   **Output:** Downloaded splat tile data; 'SplatUpdateBoundBox' event with bounding box coordinates; internal state updates to `splatTiles` and download queues.

**Data Structures:**
*   `splatFileSet`: A set of splat file identifiers currently being processed.
*   `splatTiles`: An object storing parsed metadata:
    *   `version`: String.
    *   `magicNumber`: Integer.
    *   `totalCount`: Integer (total point count).
    *   `lodTargets`: Array of integers (LOD targets for PC/Mobile).
    *   `lodDistances`: Array of floats (distances for PC/Mobile).
    *   `files`: Map of file keys to metadata (offset, count, lod level).
    *   `lodLevels`: Integer (total number of LOD levels).
*   `downloadQueue`: Array of file objects, each containing `fileKey`, `url`, `currentDistance` (from camera), and `status`.
*   `fetchingFiles`: Counter for currently active downloads.

**Parameters:**
*   `MaxDownloadCount`: Integer. Set to 3 if `isMobile` is true, otherwise 7 [S1, unofficial].

**Steps:**

1.  **Initialization:**
    *   Listen for the `TodoDownloadLod` event [S1, unofficial].
    *   When triggered, check if the `splatFile` exists in the `splatFileSet` and has no status. If so, add it to the set [S1, unofficial].
    *   Register a frame-by-frame loop to execute the `checkDownload` function. Ensure the loop terminates if the manager is disposed [S1, unofficial].

2.  **Metadata Fetching and Parsing:**
    *   Upon receiving `LodDownloadManagerAddLodMeta`, calculate the URL for the LOD metadata JSON [S1, unofficial].
    *   Fetch the JSON. If the URL ends in `lod-meta.json`, parse it as such; otherwise, parse as standard LOD JSON [S1, unofficial].
    *   Extract the bounding box `min` and `max` from the metadata tree [S1, unofficial].
    *   Fire the `SplatUpdateBoundBox` event with these coordinates [S1, unofficial].
    *   Traverse the LOD tree nodes [S1, unofficial]:
        *   For each node, calculate its center and radius from its bound `min`/`max` [S1, unofficial].
        *   Store these values and remove the raw bound object from the node [S1, unofficial].
        *   If the node has LODs, parse them into a structured array containing `fileKey`, `offset`, `count`, and `lod` level [S1, unofficial].
    *   Populate `splatTiles` with metadata derived from `sceneMeta`: `version`, `magicNumber`, `totalCount`, and initial LOD targets/distances [S1, unofficial].
    *   Compute active LOD targets and distances using `getLodTargets` and `getLodDistances` [S1, unofficial]:
        *   Filter and sort targets/distances based on device type (mobile/PC) [S1, unofficial].
        *   Ensure the first distance is 0 [S1, unofficial].
        *   **Pitfall:** If the count of provided LOD distances does not match the number of LOD targets, reset distances to a default linear progression: $distance_i = i \times 20$ [S1, unofficial].
    *   Resolve all relative URLs in `splatTiles.files` using the base URL of the fetched metadata [S1, unofficial].

3.  **Download Loop (`checkDownload`):**
    *   **Top LOD Check:** Count files at the highest LOD level (`splatTiles.lodLevels - 1`) that have downloaded data. If more than 4 such files are ready, mark `topLodReady` as true [S1, unofficial].
    *   **Capacity Check:** Count currently fetching files (`fetchingCnt`) and total todo files. If no todos remain or `fetchingCnt` exceeds `MaxDownloadCount`, return early [S1, unofficial].
    *   **Priority Check:** If top-level files (highest LOD) were queued but not yet started, return early to allow them to download first [S1, unofficial].
    *   **Queue Management:**
        *   Sort remaining todo files by `currentDistance` from the camera in ascending order [S1, unofficial].
        *   Slice the list to fit within the remaining download capacity: `MaxDownloadCount - fetchingCnt` [S1, unofficial].
        *   Initiate downloads for the sliced files [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class LODDownloadManager:
    def __init__(self, is_mobile: bool):
        self.is_mobile = is_mobile
        self.max_download_count = 3 if is_mobile else 7
        self.splat_file_set = set()
        self.splat_tiles = {}
        self.fetching_count = 0
        self.top_lod_ready = False
        self.download_queue = []
        self.is_disposed = False
        
        # Event listeners
        self.on('TodoDownloadLod', self.handle_todo_download_lod)
        self.on('LodDownloadManagerAddLodMeta', self.handle_add_lod_meta)
        
        # Frame loop
        self.register_frame_loop(self.check_download)

    def handle_todo_download_lod(self, splat_file_id):
        if splat_file_id not in self.splat_file_set:
            self.splat_file_set.add(splat_file_id)

    def handle_add_lod_meta(self, metadata_json, base_url):
        # Parse metadata
        if metadata_json.url.endswith('lod-meta.json'):
            data = parse_lod_meta_json(metadata_json.content)
        else:
            data = parse_standard_lod_json(metadata_json.content)
            
        # Extract bounds
        min_bound = data['bounding_box']['min']
        max_bound = data['bounding_box']['max']
        self.fire_event('SplatUpdateBoundBox', min_bound, max_bound)
        
        # Traverse and parse nodes
        self.splat_tiles = {'version': '', 'magicNumber': 0, 'totalCount': 0, 
                            'lodTargets': [], 'lodDistances': [], 'files': {}, 'lodLevels': 0}
        
        for node in data['nodes']:
            center, radius = self.calculate_center_radius(node['bound_min'], node['bound_max'])
            node['center'] = center
            node['radius'] = radius
            del node['bound_min']
            del node['bound_max']
            
            if 'lods' in node:
                for lod in node['lods']:
                    self.splat_tiles['files'][lod['fileKey']] = {
                        'offset': lod['offset'],
                        'count': lod['count'],
                        'lod': lod['level']
                    }
                    
        self.splat_tiles['lodLevels'] = len(self.splat_tiles['files'])
        
        # Compute LOD targets/distances
        self.splat_tiles['lodTargets'], self.splat_tiles['lodDistances'] = \
            self.compute_lod_params(data['sceneMeta'], self.is_mobile)
            
        # Resolve URLs
        for key in self.splat_tiles['files']:
            self.splat_tiles['files'][key]['url'] = self.resolve_relative_url(
                self.splat_tiles['files'][key]['relative_url'], base_url)

    def compute_lod_params(self, scene_meta, is_mobile):
        targets = scene_meta['lodTargets']
        distances = scene_meta['lodDistances']
        
        # Filter and sort based on device
        if is_mobile:
            targets = [t for t in targets if t in scene_meta['mobileTargets']]
            distances = [d for d in distances if d in scene_meta['mobileDistances']]
        else:
            targets = [t for t in targets if t in scene_meta['pcTargets']]
            distances = [d for d in distances if d in scene_meta['pcDistances']]
            
        # Ensure first distance is 0
        if not distances or distances[0] != 0:
            distances = [0] + distances
            
        # Pitfall handling: mismatched counts
        if len(distances) != len(targets):
            distances = [i * 20 for i in range(len(targets))]
            
        return targets, distances

    def check_download(self, camera_position):
        if self.is_disposed:
            return
            
        # Check top LOD readiness
        top_lod_level = self.splat_tiles['lodLevels'] - 1
        top_lod_ready_count = 0
        for file_key, file_meta in self.splat_tiles['files'].items():
            if file_meta['lod'] == top_lod_level and file_meta['status'] == 'downloaded':
                top_lod_ready_count += 1
                
        if top_lod_ready_count > 4:
            self.top_lod_ready = True
            
        # Capacity check
        if len(self.download_queue) == 0 or self.fetching_count >= self.max_download_count:
            return
            
        # Priority: Top LOD first
        top_lod_files = [f for f in self.download_queue if f['lod'] == top_lod_level and f['status'] == 'queued']
        if top_lod_files:
            return
            
        # Sort by distance
        for file in self.download_queue:
            file['currentDistance'] = self.calculate_distance(camera_position, file['center'])
            
        self.download_queue.sort(key=lambda x: x['currentDistance'])
        
        # Slice to capacity
        remaining_capacity = self.max_download_count - self.fetching_count
        files_to_download = self.download_queue[:remaining_capacity]
        
        for file in files_to_download:
            self.start_download(file)
            self.fetching_count += 1
            file['status'] = 'fetching'
            
        # Remove downloaded from queue
        self.download_queue = [f for f in self.download_queue if f['status'] != 'downloaded']

    def start_download(self, file_meta):
        # Implementation of actual HTTP request
        pass

    def calculate_center_radius(self, min_b, max_b):
        center = [(min_b[i] + max_b[i]) / 2 for i in range(3)]
        radius = sum([(max_b[i] - min_b[i]) ** 2 for i in range(3)]) ** 0.5
        return center, radius

    def calculate_distance(self, pos1, pos2):
        return sum([(pos1[i] - pos2[i]) ** 2 for i in range(3)]) ** 0.5

    def resolve_relative_url(self, relative, base):
        # Standard URL resolution logic
        pass
```

## Operating characteristics
*   **Time Complexity:** Metadata parsing is linear with respect to the number of LOD nodes. Download queue sorting is $O(N \log N)$ where $N$ is the number of pending tiles, but slicing limits actual downloads to a constant `MaxDownloadCount`.
*   **Memory Complexity:** Proportional to the size of the LOD metadata tree and the number of active download queues. The `splatTiles` structure stores all file metadata in memory.
*   **Performance:** Concurrency is limited to 3 (mobile) or 7 (desktop) to prevent network saturation and browser throttling [S1, unofficial].
*   **Strengths:** Prioritizes nearby tiles, ensuring visual continuity as the camera moves. Handles hierarchical LODs efficiently by fetching metadata first.
*   **Limitations:** Relies on a frame-by-frame loop for download checks, which may introduce slight latency compared to event-driven completion callbacks for queue management. The fixed concurrency limits may underutilize high-bandwidth connections on desktop.

## Background
The LOD Download Manager is part of the Reall3dViewer project, an open-source 3D Gaussian Splatting viewer developed by Reall3d [S1, unofficial]. It was designed to handle the specific data structure of Reall3d's splat files, which include hierarchical LODs and specific metadata formats (`lod-meta.json`) [S1, unofficial]. The component addresses the need for smooth streaming of large-scale splat scenes in web browsers, where memory and bandwidth are constrained [S1, unofficial].

## Sources
[S1] reall3d-com/Reall3dViewer, https://github.com/reall3d-com/Reall3dViewer, fetched 2026-10-27 (unofficial)
