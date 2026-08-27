# SimFoundry Pipeline A

This file is the captain-facing route and artifact contract for Pipeline A reconstruction.
The attempt manifest's recorded SimFoundry remote URL and exact checked-out commit are authoritative for commands, configuration keys, stage ordering, and file layout.
The upstream `main` references are navigation links to [`NVlabs/SimFoundry/scripts/pipeline/A_reconstruction`](https://github.com/NVlabs/SimFoundry/tree/main/scripts/pipeline/A_reconstruction), especially [`run.sh`](https://github.com/NVlabs/SimFoundry/blob/main/scripts/pipeline/A_reconstruction/run.sh), [`run_reconstruction.py`](https://github.com/NVlabs/SimFoundry/blob/main/scripts/pipeline/A_reconstruction/run_reconstruction.py), and [`scripts/pipeline/README.md`](https://github.com/NVlabs/SimFoundry/blob/main/scripts/pipeline/README.md).
The method context is SimFoundry arXiv v4, [`arXiv:2606.28276v4`](https://arxiv.org/abs/2606.28276).

## Route

The selected route is video input through stages 1b, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, and 13.
Stage 2 uses the DepthAnything3 backend in the `da3` environment.
Stages 1b, 3, 4, 5, 6, 8, 9, 10, 11, 12, and 13 use the `simfoundry` environment.
Stage 7 uses the `hunyuan` environment with Hunyuan3D-2.1 for both shape and texture and `s7_mesh.low_vram=true`.
Before any text-VLM stage, Luna runs `export SIMFOUNDRY_TEXT_VLM_BACKEND=codex`.
The installed `simfoundry.models.vlm.Gemini` factory adapter reads that selector and redirects text-capable calls to `CodexSubagent`, which runs the headless Codex CLI with `gpt-5.6-sol` and `xhigh` reasoning effort.
FLUX remains responsible for stage 5 object removal and stage 6 object completion and upsampling and is not replaced by the text-VLM selector.
The route omits stage 2c background Gaussian splat generation and does not pass `--bg-splat`.
The route omits stage 8b articulation decomposition and does not pass `--detect-articulation`.
The route pins `s13_og.include_robot=false` for the scene-reconstruction-only scope.
The route therefore produces a foreground reconstruction with the normal floor and skybox behavior at stage 13 rather than a background splat scene.

Each attempt must use a fresh resolved scene directory under `data/simfoundry-runs/`, such as `data/simfoundry-runs/official_Fruits/attempt-001`, so that a retry cannot overwrite an automatic estimate or a prior review pass.
Stage-local iteration files, manifests, automatic pose files, interactive pose directories, and intermediate model outputs are retained inside that attempt directory.
The automatic stage estimate is immutable evidence, and every correction is written under a new attempt or refinement suffix.

## Method alignment with arXiv v4

Pipeline A follows the paper's Extraction and Generation decomposition from a raw RGB video into per-object RGB-D evidence, visual meshes, aligned poses, physical annotations, and a simulator scene.
Extraction uses a representative RGB-D frame, a lifted point cloud, ground-plane alignment, iterative object segmentation, and image and depth inpainting of removed objects.
Generation uses image refinement, 2D-to-3D mesh generation, pose alignment, collision geometry, physical parameters, and scene compilation.
Object depenetration is treated as a required physical compilation step rather than an optional visual cleanup.
Physical stability is established by stepping the reconstructed objects in PyBullet while zeroing velocities after each step until poses settle, then caching the settled poses.
Intermediate outputs are retained so a reviewer can compare the automatic estimate, each object-removal iteration, each mesh generator intermediate, each pose refinement, and each corrected attempt.
The paper's optional background reconstruction and articulation paths are documented below for completeness but are excluded from this execution route.

## Stage contract

### Stage 1b - process raw video

Purpose: Normalize the source video with FFmpeg, extract all frames, and create a deterministic subsampled frame set for the rest of the route.
Inputs: One of `Fruits.mp4`, `PutCupInBowl.mp4`, or `clipboard_fruit_basket_5s_10fps.mp4` and the selected attempt directory.
Models or environment: The `simfoundry` environment, FFmpeg, OpenCV, Pillow, and the stage configuration under `s1_video`.
Outputs: `s1_video/video/scene.mp4`, the copied source video, `s1_video/frames_all/frame_*.png`, and `s1_video/frames_subsampled_<N>/frame_*.png`.
Failure modes: Missing or unreadable input, missing FFmpeg, no decodable frames, invalid sample count, insufficient source frames, or a quota failure while copying or writing frames.
Visual evidence: A source-frame contact sheet with frame count, dimensions, timestamps or indices, and a clear confirmation that every required object remains visible in the sampled views.
Acceptance: The normalized video opens, the frame count is nonzero, the sampled set is ordered, and the Sol reviewer returns PASS.

### Stage 2c - optional background Gaussian splat

Purpose: Process the configured source video with Nerfstudio, train the optional static background Gaussian splat, and export it as PLY.
Inputs: The source video configured for stage 1b, which stage 2c reuses by default.
Models or environment: The optional `nerfstudio_simfoundry` environment, Nerfstudio `ns-process-data`, COLMAP, `splatfacto`, and `gsplat`.
Outputs: `s2c_gs/processed/`, `s2c_gs/outputs/`, `s2c_gs/export/*.ply`, and `s2c_gs/stage_info.json`.
Failure modes: Missing video or optional environment, insufficient camera registration, CUDA or JIT compilation failure, training failure, excessive memory or disk use, or an export that produces no PLY.
Visual evidence: A rendered splat view and camera-registration coverage for the processed frames.
The paper's foreground-inpainted automatic background add-on uses SAM2, VOID, DepthAnything3, and 3DGRUT separately from this runner stage.
Route status: Excluded from the selected route, so no stage 2c command or review gate is scheduled for these runs.

### Stage 2 - estimate depth

Purpose: Estimate metric depth, camera information, and the RGB-D evidence consumed by ground segmentation and world-frame construction.
Inputs: The stage 1b subsampled frames and the selected `da3` backend.
Models or environment: DepthAnything3 in the `da3` environment with the configured resolution and chunking limits.
Outputs: `s2_da/` backend results including depth arrays, RGB frames, intrinsics, camera poses, and the stage metadata record.
Failure modes: Missing depth checkpoint, unsupported frame resolution, CUDA or VRAM exhaustion, chunk merge failure, invalid metric scale, or missing output arrays.
Visual evidence: Per-frame depth heatmaps, confidence maps when available, and a point-cloud projection colored by RGB.
Acceptance: The selected frame has finite metric depth, intrinsics are present, the point cloud has usable points, and the Sol reviewer returns PASS.

### Stage 3 - segment the ground plane and select the canonical frame

Purpose: Select the frame used by stages 3 through 13, segment the support surface, fit its plane, and define the gravity direction and origin.
Inputs: Stage 1b frames, stage 2 depth and intrinsics, and the frame-selection configuration.
Models or environment: SAM3 and the selected Codex text VLM route in the `simfoundry` environment, with Open3D RANSAC plane fitting.
Outputs: `s3_ground/raw_img.png`, `s3_ground/annotated_img.png`, `s3_ground/image_<idx>_floor_info.json`, `s3_ground/frame_selection.json`, and the optional frame-selection debug sheet.
Failure modes: No floor category or mask, multiple unusable masks, poor plane inlier ratio, excessive tilt, clipped or occluded objects, invalid depth, or a frame-selection ambiguity.
Visual evidence: The canonical frame with the accepted floor mask and box, the scored candidate contact sheet, the fitted plane over the RGB-D point cloud, and the world-up arrow.
Acceptance: The chosen frame is recorded once, the floor plane passes the configured tilt and inlier gates, and the Sol reviewer returns PASS.

### Stage 4 - unify the world frame

Purpose: Transform the camera-native point cloud into a gravity-consistent world frame with the support plane as the reference.
Inputs: Stage 2 RGB-D data, stage 3 floor information, camera intrinsics, and the chosen frame index.
Models or environment: Open3D and SciPy geometry utilities in the `simfoundry` environment.
Outputs: `s4_frame/image_<idx>_pc_raw.ply`, pruned and denoised PLY variants, `image_<idx>_pc_raw_rotated.ply`, and `image_<idx>_cam2world.npy`.
Failure modes: Missing floor information, invalid depth or intrinsics, empty or noisy point cloud, incorrect axis convention, or a transform that does not put the support plane below objects.
Visual evidence: The raw, pruned, denoised, and rotated point clouds with axes and the fitted support plane shown in the same viewer.
Acceptance: The transform is finite and repeatable, the world Z direction is upward, and the Sol reviewer returns PASS.

### Stage 5 - decompose the scene

Purpose: Detect foreground objects, iteratively isolate each object, remove it from the residual scene, and produce per-object RGB-D evidence for mesh generation and pose alignment.
Inputs: The stage 3 canonical RGB frame, stage 2 depth, stage 4 world-frame point cloud and transform, and prior iteration state when resuming.
Models or environment: Codex text VLM for object categories and reasoning, SAM3 for image segmentation, FLUX for object removal, and PriorDepthAnything with the DA3 geometric backend in the `simfoundry` environment.
Outputs: `s5_scene/decomposition_frame.json`, `obj_cat_list/iter_<N>.json`, detected-phrase overlays, removal masks, masked object crops, pre-removal and post-removal images, per-iteration metric depth `.npy` files and heatmap `.png` files, and residual scene images.
Failure modes: VLM timeout or malformed response, invalid bounding box, missed or duplicate mask, occlusion, insufficient valid pixels, rejected FLUX removal, unusable regenerated depth, frame mismatch on resume, an unsupported Flash-Attention version such as `2.8.3` when the required range is `2.7.1`-`2.7.4`, or quota exhaustion from retained iterations.
Visual evidence: Numbered detection overlays, mask and outline overlays, per-object transparent and background crops, before-and-after removal pairs, and a depth heatmap for every accepted iteration.
Acceptance: Every retained object has a valid category record, mask, crop, removal result, and finite metric depth, and the Sol reviewer returns PASS.

### Stage 6 - refine object images

Purpose: Pad and upsample each extracted object image, preserve its visual identity, and emit an alpha-separated input suitable for 3D generation.
Inputs: Stage 5 `masked_object/iter_<N>.png` crops and their object metadata.
Models or environment: FLUX image editing in the `simfoundry` environment, with the configured background-removal model and `infill=false` unless a reviewed correction explicitly enables infill.
Outputs: `s6_upsample/padded/`, optional `infilled/`, `upsampled/`, transparent upsampled PNGs, optional validity JSON, and stage metadata.
Failure modes: Empty or malformed alpha mask, invalid crop ratio, FLUX response failure, background-removal failure, image-size mismatch, invalid object content, or an output that changes the object category or affordance.
Visual evidence: Side-by-side source crop, padded input, FLUX output, and transparent output with a checkerboard background and object bounding box.
Acceptance: Every required object has a reviewable transparent image at the expected resolution and the Sol reviewer returns PASS.

### Stage 7 - generate object meshes

Purpose: Generate a visual mesh and texture for every accepted object image.
Inputs: Stage 6 `upsampled/iter_*_transparent.png` images and the optional object-index filter.
Models or environment: Hunyuan3D-2.1 shape and texture generation in the `hunyuan` environment with `low_vram=true` and `save_intermediates=true`.
Outputs: `s7_mesh/shape/hunyuan/*_shape.obj`, `s7_mesh/textured_mesh/hunyuan/*_mesh.glb`, untextured intermediate GLBs, per-object manifests, and retained generator intermediate outputs.
Failure modes: Missing Hunyuan checkpoint, cache-location or home-quota failure, CUDA or VRAM exhaustion, malformed or fully opaque full-scene input when an isolated RGBA object is required, zero-triangle or non-finite mesh, texture baking failure, or an incomplete per-object manifest.
Visual evidence: Multi-view mesh renders, textured and untextured comparisons, silhouette overlays against the stage 6 image, and a structural report containing triangle counts, bounds, material references, and finite vertex checks.
Acceptance: The bounded Hunyuan smoke has passed execution and artifact-loadability checks with an isolated RGBA object input, every mesh job is finished, every mesh is loadable and non-empty, and Sol accepts the visual-quality evidence.

### Stage 8 - match object poses

Purpose: Canonicalize each generated mesh's semantic front, fit it to its observed object point cloud, and emit a metric 6D transform and scale.
Inputs: Stage 2 depth, RGB, and intrinsics, stage 5 masks and regenerated depth, stage 6 reference photos when available, and stage 7 meshes.
Models or environment: In the `simfoundry` environment, Luna runs `export SIMFOUNDRY_TEXT_VLM_BACKEND=codex`, passes `s8_pose.front_pick_model=gemini-2.5-flash` into the Gemini factory, keeps global `visualize=false`, and sets `s8_pose.use_foundationpose=true` so the Codex front selection is followed by FoundationPose through its CUDA `RasterizeCudaContext` without enabling Open3D `draw_geometries`.
Outputs: `s8_pose/front_views/<object>/view_{A-H}_az*.png`, `s8_pose/canonical_mesh/<object>_orientation.json`, the canonical mesh under `s8_pose/canonical_mesh/`, the final pose in `s8_pose/info/<object>.json`, an immutable headless `s8_pose/fit/<object>_foundationpose_fit.png`, and retained `info_interactive*` refinements when manually corrected.
Failure modes: Missing Codex selection, VLM failure recorded as an unchanged orientation, incorrect semantic-front or yaw selection, bad mesh topology, sparse or occluded mask, registration local minimum, implausible scale, missing pose dependency, CUDA failure, an interactive viewer caused by enabling `visualize`, or a transform that projects outside the observed object.
Visual evidence: After execution, Luna generates the per-object box and axis overlay headlessly from the saved `z_up` transform, canonical mesh bounds, accepted RGB, and intrinsics with FoundationPose `Utils.draw_posed_3d_box` and `draw_xyz_axis` or equivalent OpenCV projection, alongside the eight front views, stage 6 reference photo when available, and orientation JSON.
Acceptance: Sol confirms each semantic-front and yaw choice, the fitted orientation, and the per-object headless box and axis overlay before PASS, while a frontless symmetric fruit or bowl may use an explicitly justified `ambiguous` or `NONE` result with zero yaw.

### Stage 8b - optional articulation

Purpose: Decompose articulated meshes into movable parts and generate joint parameters and physical properties.
Inputs: Stage 7 meshes, multi-view renders, and object classifications.
Models or environment: The optional articulate-anything environments, a text VLM, mesh segmentation, URDF generation, and a critic loop.
Outputs: `s8b_articulate_objects/` classifications, part meshes, URDFs, joint metadata, and motion-review videos.
Failure modes: Missing articulation environments, unsupported object, incorrect part segmentation, implausible joint axis or limits, invalid URDF, or a critic loop that never reaches its threshold.
Visual evidence: Color-coded part segmentation, joint-axis overlays, and simulated motion video judged against the reference object.
Route status: Excluded from the selected route, so no stage 8b command or review gate is scheduled for these runs.

### Stage 9 - compile the scene

Purpose: Apply the world-frame transform and each object pose to compose and validate the foreground scene before physics preparation.
Inputs: Stage 4 world point cloud and transform, stage 8 canonical meshes and pose JSON files, and the selected frame index.
Models or environment: Open3D and SciPy utilities in the `simfoundry` environment.
Outputs: `s9_compile/stage_info.json`; the composed scene exists in memory and Luna derives the retained scene view required below.
Failure modes: Missing pose or mesh, mismatched object index, invalid transform, incorrect camera-to-world multiplication, or an object that is visibly displaced from its source evidence.
Visual evidence: A complete compiled point-cloud and mesh scene with per-object labels and coordinate axes.
Acceptance: All accepted objects are present exactly once with finite poses and the Sol reviewer returns PASS.

### Stage 10 - make objects simulation-ready

Purpose: Assign physical properties, create collision geometry, and emit URDF-backed assets that can be imported by OmniGibson.
Inputs: Stage 5 categories, stage 6 object images, and stage 8 canonical meshes and pose scales.
Models or environment: The `simfoundry` environment, the selected text VLM for physical annotation, and CoACD collision decomposition.
Outputs: `s10_sim/scene_objects_info.json`, per-object visual and collision meshes, URDF files, material and physical property records, and stage metadata.
Failure modes: Missing or invalid mesh, malformed category name, VLM authentication or schema failure, CoACD failure, missing visual or collision reference, non-finite mass or friction, or an invalid URDF.
Visual evidence: Visual-mesh and collision-mesh overlays, per-object bounding boxes, and an URDF load report that resolves every referenced mesh.
Acceptance: Every object has a loadable URDF, collision geometry, finite physical parameters, and a Sol reviewer PASS.

### Stage 11 - depenetrate and stabilize physics

Purpose: Resolve residual interpenetration and settle the reconstructed objects into a physically plausible configuration.
Inputs: Stage 10 URDFs and physical properties, stage 8 or approved refined poses, and the stage 4 gravity-aligned world frame.
Models or environment: PyBullet in the `simfoundry` environment with collision meshes and controlled stepping.
Outputs: `s11_physics/pb_scene_poses.json`, stabilization logs, and retained before-and-after pose evidence.
Failure modes: Missing collision mesh, exploding contact solver, persistent interpenetration, objects falling through the support plane, non-settling velocities, or a pose drift beyond the accepted tolerance.
Visual evidence: Before-and-after stability renders, collision wireframes, contact and penetration diagnostics, and a short settled-motion capture.
Acceptance: The scene settles within the configured step limit, velocities and pose deltas are below the acceptance thresholds, no object remains penetrated, and the Sol reviewer returns PASS.

### Stage 12 - import USD assets

Purpose: Import the simulation-ready URDF assets into the OmniGibson asset dataset and validate the resulting USD structures.
Inputs: Stage 10 URDFs, collision and visual meshes, material properties, and the selected `real2sim-assets` dataset name.
Models or environment: Headless OmniGibson and Isaac Sim tooling in the `simfoundry` environment.
Outputs: Imported USD assets under the dataset path, any required material post-processing, and `s12_usd/` stage metadata.
Failure modes: Headless runtime startup failure, missing mesh reference, malformed URDF, importer error, invalid USD prim hierarchy, or a material opacity setting that makes the asset translucent or invisible.
Visual evidence: A USD load report with prim hierarchy, resolved mesh paths, materials, and a rendered asset preview.
Acceptance: Every imported USD asset loads headlessly, contains the expected visual and collision structure, and the Sol reviewer returns PASS.

### Stage 13 - create the OmniGibson scene

Purpose: Load the imported assets with stabilized poses and export the final reconstructed OmniGibson scene and preview.
Inputs: Stage 10 object metadata, stage 11 settled poses, stage 12 USD assets, and the selected no-background-splat scene configuration.
Models or environment: Headless OmniGibson in the `simfoundry` environment with the configured viewer camera and `s13_og.include_robot=false`.
Outputs: `s13_og/reconstructed_og_scene.json`, `s13_og/settled_poses.json`, `s13_og/reconstructed_scene.png`, and stage metadata.
Failure modes: Missing USD asset, failed scene reset, invalid pose or material, sensor or viewer startup failure, non-finite object state, or an exported scene that cannot be reopened.
Visual evidence: The final OmniGibson preview, a headless reload and render log, object count and names, and a final scene JSON structural check.
Acceptance: The JSON reloads, every expected object is visible and placed plausibly, the preview is captured, and the Sol reviewer returns PASS.

## Artifact review rule

Every included stage produces both its primary artifact and a reviewable derived visualization or structural view.
Image stages must include source-to-output comparisons, and non-image stages must include the relevant depth heatmap, point cloud, pose overlay, collision mesh, stability render, USD load evidence, or final OmniGibson preview.
Between gates, apply the [risk-proportionate validation rule](AGENTS.md#dispatch-and-supervision-handoff), which preserves every correctness, safety, and reviewer requirement in this tracker.
No downstream stage may start until the designated Sol reviewer has inspected the current attempt, returned PASS or concrete steering, and any requested correction has been rerun and accepted.
The detailed reviewer outcomes, mediation loop, attempt naming, and checkbox tracker live in [`SIMFOUNDRY_EXECUTION_PLAN.md`](SIMFOUNDRY_EXECUTION_PLAN.md).

## Current evidence

The single timestamped snapshot owner is [`SIMFOUNDRY_EXECUTION_PLAN.md`](SIMFOUNDRY_EXECUTION_PLAN.md#current-evidence-boundary).
