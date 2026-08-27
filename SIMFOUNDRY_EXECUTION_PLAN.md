# SimFoundry Execution Plan

This file is the captain-facing execution order, reviewer gate, attempt policy, and acceptance tracker for the selected Pipeline A route.
The attempt manifest's recorded SimFoundry remote URL and exact checked-out commit are the implementation authority, the upstream [`NVlabs/SimFoundry`](https://github.com/NVlabs/SimFoundry) link is for navigation, and the stage contract is in [`SIMFOUNDRY_PIPELINE.md`](SIMFOUNDRY_PIPELINE.md).

## Roles and hard dependency

Luna is the operator agent that runs one installation, verification, smoke, or Pipeline A step at a time.
Sol is the designated reviewer and must run as `gpt-5.6-sol` with `xhigh` reasoning effort.
Firstmate mediates every artifact handoff, reviewer response, correction request, and gate transition.
Sol is a hard dependency after every module or step, including installation rows, verification rows, the bounded Hunyuan smoke, every video stage, and final reload checks.
Luna must not begin the next module or step until Sol has inspected the current outputs, returned PASS or concrete steering, and every requested correction has been rerun and accepted.
Silence, a successful process exit, a green dry-run, or a plausible log is not a reviewer PASS.
Firstmate records the reviewer result against the exact attempt and keeps the next step blocked until the result is complete.
Every Pipeline A stage runs as `scripts/pipeline/A_reconstruction/run.sh --include <single-stage-id> --no-stream`, with exactly one stage identifier in `--include`.
When that command exits, Luna packages immutable versioned evidence and stops while Firstmate routes it to Sol, and only PASS authorizes the next stage command.
Between gates, apply the [risk-proportionate validation rule](AGENTS.md#dispatch-and-supervision-handoff), which preserves every correctness, safety, and reviewer requirement in this tracker.

## Reviewer outcomes

`PASS` means the required artifact set exists, the derived visualization is reviewable, behavioral or structural validation passed, and no requested correction remains open.
`STEER` means Sol found a concrete correction and returned its artifact, rationale, rerun scope, and acceptance condition, so the current gate remains closed.
`FAIL` means the output is unusable or the validation failed, so Luna stops at the current step and Firstmate routes the recovery without advancing.
`BLOCK_INPUT` means Sol cannot judge the requested quality because the supplied input or evidence is invalid for that review, so Luna must create a corrected versioned input, rerun the dependent step, and return to Sol before advancing.
`NEEDS-DECISION` means the evidence leaves a product or destructive choice genuinely ambiguous, so Firstmate escalates it and does not let Luna choose silently.
`BLOCKED` means a dependency or external condition prevents a meaningful rerun, so Firstmate records the blocker and leaves the step held.
Sol must inspect both visual and non-visual evidence before returning PASS.

## Steering loop

1. Luna creates a unique attempt directory and writes the automatic outputs plus a manifest of commands, configuration, versions, and checksums.
2. Luna writes the required derived visualizations and structural or behavioral validation beside that attempt's primary artifacts.
3. Firstmate sends Sol the exact attempt directory, manifest, evidence index, and current step identifier.
4. Sol returns one of the defined outcomes with evidence references and a concrete correction when needed.
5. On `PASS`, Firstmate records the gate and authorizes the next step.
6. On `STEER`, Luna creates a new attempt or refinement suffix, reruns only the requested correction and its dependent checks, and leaves the prior attempt untouched.
7. Firstmate sends the new evidence to Sol for a fresh decision and repeats the loop until PASS, a genuine blocker, or a captain decision is required.
8. Firstmate never converts a steer into a pass, and Luna never closes a reviewer gate from its own logs.

## Versioned attempts

Every run uses a unique resolved scene directory under `data/simfoundry-runs/`, such as `data/simfoundry-runs/official_Fruits/attempt-001`, and never reuses a prior attempt directory for a retry.
The attempt manifest records the SimFoundry remote URL and exact checked-out commit, Codex adapter revision, resolved text-VLM backend, model, and effort, source video checksum, route, configuration overrides, environment matrix, model identifiers, command lines, and output paths.
Stage 5 object iterations remain as `iter_<N>` records, stage 7 generator intermediates remain enabled, and stage 8 automatic poses remain in `info/` while refinements use new `info_interactive*` directories.
An attempt correction receives a new attempt number or an explicit refinement suffix such as `attempt-001-r1`, and no correction may overwrite the automatic estimate or a prior Sol PASS.
Firstmate links each Sol result to one immutable attempt identifier and marks superseded attempts as retained history rather than deleting them.

## Ordered modules and subgoals

The following order is strict, and each row ends at a Sol gate before the next row begins.

### M0 - freeze the route

Luna subgoal: Create the attempt manifest and pin video mode, `SIMFOUNDRY_TEXT_VLM_BACKEND=codex`, the `CodexSubagent` headless Codex CLI route with `gpt-5.6-sol` and `xhigh`, FLUX for stages 5 and 6, Hunyuan3D-2.1 for stage 7 shape and texture, `low_vram=true`, global `visualize=false`, `s8_pose.front_pick_model=gemini-2.5-flash`, `s8_pose.use_foundationpose=true`, no stage 2c, no stage 8b, and `s13_og.include_robot=false`.
Required evidence: The manifest-pinned source and adapter revisions, resolved single-stage `--include <single-stage-id> --no-stream` command template, configuration diff, route summary, and unique attempt directory.
Sol gate: PASS confirms the route and attempt identity, or STEER names the exact route correction.

### M1 - install the seven environments

Luna subgoal: Confirm the seven installed environments are usable without rebuilding a healthy environment.
Required evidence: Environment paths, package import probes, CUDA visibility, and installer logs.
Sol gate: PASS confirms installation evidence, or STEER names the environment and probe to repair.

### M2 - verify the environment matrix and native extension

Luna subgoal: Verify the final environment matrix, rebuild `tiny-cuda-nn` if required, and behavior-test its `sm_86` path.
Required evidence: The green matrix, build output, and a behavior test that exercises the actual extension on `sm_86`.
Sol gate: PASS confirms the matrix and extension behavior, or STEER names the failing environment or test.

### M3 - verify checkpoints and services

Luna subgoal: Confirm the nine checkpoint groups and the selected VLM and model service credentials are available without exposing secrets.
Required evidence: Checkpoint inventory with paths and sizes, non-secret authentication probes, and model load checks.
Sol gate: PASS confirms the inputs required by the selected route, or BLOCKED records the exact missing credential or external dependency.

### M4 - run the bounded Hunyuan smoke

Luna subgoal: Run the smallest shape-and-texture Hunyuan3D-2.1 smoke with `low_vram=true` and a controlled cache location inside an approved disk budget.
Required evidence: The source image, shape output, textured output, retained intermediates, VRAM report, mesh structural report, and smoke log.
Sol gate: Shape and texture execution, manifest and stage status, artifact loadability, and the raw `nvidia-smi` peak passed for one discovered object, but Sol returned `BLOCK_INPUT` for visual quality because the smoke input was a fully opaque full scene rather than an isolated RGBA object.
Luna must rerun the smoke with a versioned isolated crop and obtain Sol PASS for visual quality before any full-video stage 7 run, while retaining the prior home-quota cache-location failure in the logs.

### M5 - run each video through stage 1b

Luna subgoal: Normalize and sample one video into its own versioned attempt directory.
Required evidence: Valid normalized video, full and sampled frame sets, frame contact sheet, and frame-count check.
Sol gate: PASS is required before that video's stage 2 begins.

### M6 - run each video through stage 2

Luna subgoal: Produce finite DepthAnything3 depth and camera evidence for the sampled frames.
Required evidence: Depth arrays, confidence or validity statistics, depth heatmaps, intrinsics, camera data, and point-cloud check.
Sol gate: PASS is required before that video's stage 3 begins.

### M7 - run each video through stage 3

Luna subgoal: Select and record one canonical frame and fit a valid support plane.
Required evidence: Candidate scoring sheet, canonical frame, floor mask overlay, plane fit, and `floor_info.json`.
Sol gate: PASS is required before that video's stage 4 begins.

### M8 - run each video through stage 4

Luna subgoal: Produce a denoised point cloud and a gravity-aligned camera-to-world transform.
Required evidence: Raw, pruned, denoised, and rotated point clouds, axes overlay, and transform validation.
Sol gate: PASS is required before that video's stage 5 begins.

### M9 - run each video through stage 5

Luna subgoal: Decompose the canonical frame into complete per-object crops, removals, metadata, and regenerated depth.
Required evidence: Detection overlays, masks, removal pairs, per-object crops, per-iteration metadata, depth heatmaps, and retained residual scene outputs.
Sol gate: PASS is required before that video's stage 6 begins, and every requested object correction must be rerun before PASS.

### M10 - run each video through stage 6

Luna subgoal: Generate FLUX-refined, alpha-separated object images without changing object identity or affordance.
Required evidence: Source-to-padded-to-FLUX-to-transparent comparisons, output dimensions, alpha validity, and object validity checks.
Sol gate: PASS is required before that video's stage 7 begins.

### M11 - run each video through stage 7

Luna subgoal: Generate Hunyuan3D-2.1 shape and texture assets with `low_vram=true` for every accepted object.
Required evidence: Per-object shape and textured mesh, retained generator intermediates, manifest status, multi-view renders, silhouette comparisons, and mesh structural checks.
Sol gate: PASS is required before that video's stage 8 begins, and a smoke PASS is a prerequisite but does not replace this full stage review.

### M12 - run each video through stage 8

Luna subgoal: Export `SIMFOUNDRY_TEXT_VLM_BACKEND=codex`, pass `s8_pose.front_pick_model=gemini-2.5-flash`, keep global `visualize=false`, set `s8_pose.use_foundationpose=true`, canonicalize each mesh's semantic front through `CodexSubagent`, and fit it through the existing CUDA `RasterizeCudaContext` without enabling Open3D `draw_geometries`.
Headless evidence step: After execution, generate an immutable per-object `s8_pose/fit/<object>_foundationpose_fit.png` from the saved `z_up` transform in `s8_pose/info/<object>.json`, the canonical mesh bounds under `s8_pose/canonical_mesh/`, accepted RGB, and intrinsics with FoundationPose `Utils.draw_posed_3d_box` and `draw_xyz_axis` or equivalent OpenCV projection.
Required evidence: Per-object `s8_pose/front_views/<object>/view_{A-H}_az*.png`, the stage 6 reference photo when available, `s8_pose/canonical_mesh/<object>_orientation.json`, the saved canonical mesh and final pose JSON, the headless box and axis overlay, and any retained refinement directories.
Sol gate: PASS is required before that video's stage 9 begins after Sol confirms the semantic-front and yaw choice, fitted orientation, and per-object box and axis overlay, with an explicitly justified `ambiguous` or `NONE` result and zero yaw accepted for a frontless symmetric fruit or bowl.

### M13 - run each video through stage 9

Luna subgoal: Compile all accepted meshes and poses into one world-frame foreground scene.
Required evidence: Complete scene render or point-cloud overlay, object inventory, transform composition report, and stage metadata.
Sol gate: PASS is required before that video's stage 10 begins.

### M14 - run each video through stage 10

Luna subgoal: Create collision geometry, physical annotations, and loadable URDF assets for every object.
Required evidence: Physical-property record, visual and collision mesh overlay, URDF parse and mesh-reference check, and object inventory.
Sol gate: PASS is required before that video's stage 11 begins.

### M15 - run each video through stage 11

Luna subgoal: Depenetrate and settle all objects in PyBullet and cache the stable poses.
Required evidence: Before-and-after stability renders, collision wireframes, pose and velocity convergence, penetration diagnostics, and `pb_scene_poses.json`.
Sol gate: PASS is required before that video's stage 12 begins.

### M16 - run each video through stage 12

Luna subgoal: Import every URDF into the `real2sim-assets` USD dataset and prove the USD asset structure loads.
Required evidence: Headless importer log, USD prim and mesh-reference report, material report, and rendered asset previews.
Sol gate: PASS is required before that video's stage 13 begins.

### M17 - run each video through stage 13

Luna subgoal: Create and reopen the final OmniGibson scene using the stabilized poses and imported USD assets.
Required evidence: `reconstructed_og_scene.json`, `settled_poses.json`, final `reconstructed_scene.png`, headless reload log, and final object inventory.
Sol gate: PASS closes that video's Pipeline A run, while FAIL or STEER leaves the final scene held for correction.

### M18 - reconcile the three-video package

Luna subgoal: Compare the accepted attempt manifests and reviewer records for all three videos and publish the final evidence index.
Required evidence: Three final previews, three scene reload results, per-stage PASS records, retained-attempt index, and unresolved issue list.
Sol gate: PASS closes the package only when every video and every included stage has its own PASS.

## Accepted evidence records

The tracker links every checked cell to its immutable attempt and exact review or evidence record.

| Record | Exact attempt | Manifest | Review or evidence record |
|---|---|---|---|
| <a id="verification-snapshot-20260826t230030z"></a>Verification snapshot | `data/simfoundry-runs/verification/attempt-20260826T230030Z` | `data/simfoundry-runs/verification/attempt-20260826T230030Z/manifest.json` | `data/simfoundry-runs/verification/attempt-20260826T230030Z/evidence-index.md` |
| <a id="hunyuan-smoke-attempt-001"></a>Bounded Hunyuan smoke | `data/simfoundry-runs/hunyuan-smoke/attempt-001` | `data/simfoundry-runs/hunyuan-smoke/attempt-001/manifest.json` | `data/simfoundry-runs/hunyuan-smoke/attempt-001/sol-review.md` |
| <a id="official-fruits-stage-1b-pass"></a>`official_Fruits` stage 1b PASS | `data/simfoundry-runs/official_Fruits/attempt-001` | `data/simfoundry-runs/official_Fruits/attempt-001/manifest.json` | `data/simfoundry-runs/official_Fruits/attempt-001/reviews/stage-1b-sol.md` |
| <a id="official-fruits-stage-2-pass"></a>`official_Fruits` stage 2 PASS | `data/simfoundry-runs/official_Fruits/attempt-001` | `data/simfoundry-runs/official_Fruits/attempt-001/manifest.json` | `data/simfoundry-runs/official_Fruits/attempt-001/reviews/stage-2-sol.md` |

## Fine-grained tracker

`[x]` means the evidence is current and accepted, while `[ ]` means not yet accepted or still in progress.

### Installation and verification

- [x] [`simfoundry` environment is installed.](#verification-snapshot-20260826t230030z)
- [x] [`hunyuan` environment is installed.](#verification-snapshot-20260826t230030z)
- [x] [`any6d` environment is installed.](#verification-snapshot-20260826t230030z)
- [x] [`da3` environment is installed.](#verification-snapshot-20260826t230030z)
- [x] [`void` environment is installed.](#verification-snapshot-20260826t230030z)
- [x] [`nerfstudio_simfoundry` environment is installed.](#verification-snapshot-20260826t230030z)
- [x] [`3dgrut` environment is installed.](#verification-snapshot-20260826t230030z)
- [x] [Final environment matrix is green.](#verification-snapshot-20260826t230030z)
- [x] [`tiny-cuda-nn` was rebuilt and behavior-tested for `sm_86`.](#verification-snapshot-20260826t230030z)
- [x] [Nine checkpoint groups were downloaded.](#verification-snapshot-20260826t230030z)
- [x] [Pipeline A dry-run passed.](#verification-snapshot-20260826t230030z)
- [x] [Repository tests passed with 243 passed and 29 skipped.](#verification-snapshot-20260826t230030z)
- [ ] Selected Codex text VLM access is verified for the actual run attempt.
- [ ] Selected FLUX access is verified for stages 5 and 6 in the actual run environment.
- [ ] All attempt manifests record the exact source checksum, route, and configuration.

### Bounded Hunyuan smoke

- [x] [Cache-location recovery from the home-quota failure completed for the valid one-object smoke.](#hunyuan-smoke-attempt-001)
- [x] [Hunyuan3D-2.1 shape smoke with `low_vram=true` passed and produced one shape OBJ.](#hunyuan-smoke-attempt-001)
- [x] [Hunyuan3D-2.1 texture smoke with `low_vram=true` passed and produced one textured GLB.](#hunyuan-smoke-attempt-001)
- [x] [The one-object smoke reported successful manifest and stage status and a raw `nvidia-smi` peak of about 17203 MiB.](#hunyuan-smoke-attempt-001)
- [x] [Sol returned `BLOCK_INPUT` for visual-quality validation because the smoke input was a fully opaque full scene rather than an isolated RGBA object.](#hunyuan-smoke-attempt-001)
- [ ] The smoke is rerun with a versioned isolated RGBA crop and receives Sol PASS for visual quality.
- [ ] Sol returned PASS for the bounded Hunyuan smoke.

### Pipeline A per-video gates

| Stage | `Fruits.mp4` | `PutCupInBowl.mp4` | `clipboard_fruit_basket_5s_10fps.mp4` |
|---|---|---|---|
| 1b | [x] [Sol PASS](#official-fruits-stage-1b-pass) | [ ] Sol PASS | [ ] Sol PASS |
| 2 | [x] [Sol PASS](#official-fruits-stage-2-pass) | [ ] Sol PASS | [ ] Sol PASS |
| 3 | [ ] provisional, Sol pending | [ ] Sol PASS | [ ] Sol PASS |
| 4 | [ ] provisional, Sol pending | [ ] Sol PASS | [ ] Sol PASS |
| 5 | [ ] terminated: Flash-Attention 2.8.3 outside 2.7.1-2.7.4 | [ ] Sol PASS | [ ] Sol PASS |
| 6 | [ ] not started | [ ] Sol PASS | [ ] Sol PASS |
| 7 | [ ] Sol PASS | [ ] Sol PASS | [ ] Sol PASS |
| 8 | [ ] Sol PASS | [ ] Sol PASS | [ ] Sol PASS |
| 9 | [ ] Sol PASS | [ ] Sol PASS | [ ] Sol PASS |
| 10 | [ ] Sol PASS | [ ] Sol PASS | [ ] Sol PASS |
| 11 | [ ] Sol PASS | [ ] Sol PASS | [ ] Sol PASS |
| 12 | [ ] Sol PASS | [ ] Sol PASS | [ ] Sol PASS |
| 13 | [ ] Sol PASS | [ ] Sol PASS | [ ] Sol PASS |
| Package reconciliation | [ ] Sol PASS | [ ] Sol PASS | [ ] Sol PASS |

Stage 2c is marked excluded by route for all three videos.
Stage 8b is marked excluded by route for all three videos.
An excluded stage has no execution checkbox and cannot be treated as a failed or passed artifact.
The `official_Fruits` run received Sol PASS for stage 1b and stage 2, while stage 2 began before the new hard gate arrived.
Stages 3 and 4 produced provisional outputs but are not yet Sol-approved, stage 5 terminated before useful decomposition output because Flash-Attention `2.8.3` is outside the required `2.7.1`-`2.7.4` range, and stage 6 did not start.

## Non-image evidence minimums

Stage 1b requires a frame contact sheet and frame-count validation.
Stage 2 requires depth heatmaps, confidence or finite-value checks, and a point-cloud projection.
Stage 3 requires a ground-mask and plane-fit overlay.
Stage 4 requires raw-to-world point-cloud evidence and transform validation.
Stage 5 requires detection, removal, crop, and regenerated-depth evidence for each retained iteration.
Stage 6 requires image comparisons and alpha or background structural checks.
Stage 7 requires mesh renders, silhouette comparisons, and triangle, bounds, material, and finite-vertex checks.
Stage 8 requires front-view and reference-photo evidence, the orientation JSON, saved canonical mesh and pose, a headless per-object box and axis overlay, and transform and scale checks.
Stage 9 requires a full compiled-scene view and object inventory.
Stage 10 requires collision-mesh overlays and URDF structural validation.
Stage 11 requires stability renders and behavioral convergence or penetration checks.
Stage 12 requires USD load evidence, prim hierarchy, mesh-reference, and material checks.
Stage 13 requires the final OmniGibson preview, JSON reload, object inventory, and headless render evidence.

## Current evidence boundary

The installation and repository-test facts above are accepted current evidence as of 2026-08-26.
Current snapshot at `2026-08-26T23:00:30Z` records seven installed environments, a green final environment matrix, an `sm_86`-tested `tiny-cuda-nn` rebuild, nine downloaded checkpoint groups, a passed Pipeline A dry-run, and repository tests with 243 passed and 29 skipped.
The valid bounded Hunyuan stage-7 object smoke passed shape and texture execution, artifact loadability, manifest and stage status, and a raw `nvidia-smi` peak of about 17203 MiB for one discovered object, producing a shape OBJ and textured GLB.
Sol returned `BLOCK_INPUT` for visual-quality validation because the smoke input was a fully opaque full scene rather than an isolated RGBA object.
The quality correction is deferred to a versioned isolated crop before full stage 7, and the earlier home-quota cache-location failure remains in the retained smoke logs as historical evidence.
The `official_Fruits` run received Sol PASS for stage 1b and stage 2.
Stages 3 and 4 produced provisional outputs but are not yet Sol-approved, stage 5 terminated before useful decomposition output because Flash-Attention `2.8.3` is outside the required `2.7.1`-`2.7.4` range, and stage 6 did not start.
The dry-run proves command planning only and does not mark any full-video stage accepted.
No full-video stage is accepted until its own versioned artifacts have a Sol PASS.
