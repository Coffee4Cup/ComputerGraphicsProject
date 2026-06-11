# Project Context: 2D-3D Pose Estimation (Mini-Project 2025)

## Project Overview
This project involves building a 3D Interactive Graphics application in C++ using OpenGL and OpenCV. The goal is to simulate and solve the Perspective-n-Point (PnP) problem by estimating camera poses based on 2D-to-3D correspondences. The terrain is built using Digital Elevation Maps (DEM).

The final application must support four specific modes (spec: `mini2025.pdf`, pages 46-54):
1.  **A. Navigation Mode:** A dual-viewport system (Global View vs. Camera View) where the user can move the camera, record its path (R), and play it back (Ctrl+R).
2.  **B. Picking Mode:** Manual selection of 3D points via mouse clicks (color picking) to map 2D-3D correspondences and compute the camera pose (C) compared to the "true" pose.
3.  **C. Trackers Mode:** Automated pose estimation using uniquely colored 3D spheres placed in the scene to establish correspondences. On each 'B' press: record true pose + compute pose from automatic correspondences. Global view shows true-vs-computed fly-through paths; camera view toggles between timesteps (N/M) showing the diff.
4.  **D. 2D Feature Matching Mode:** Pre-phase computes 2D features (ORB/SIFT) for n selected views and maps them to 3D; run-phase estimates pose by feature matching (Brute-Force/FLANN + RANSAC). Same true-vs-computed displays as Trackers Mode.

**Experiments required (PDF p.54):** run on at least 2 different DEM terrains; in Feature Matching Mode, change the lighting between the Pre and Run phases (so the app needs a controllable lighting model).

**Course schedule:** Mid-term (Modes A+B) was due 27.5 — passed. Final term covers the full application (Modes C+D + experiments).

## Where Things Stand (updated 11.6.2026)
- **Phases 1-3 of the original pipeline are complete**: environment stabilization, architectural review, and the OpenGL 3.3 core-profile modernization. The modernization was merged upstream (PR #1).
- **The partner (Oren) has since refactored heavily on top of it.** Current upstream architecture (read `docs/architecture-notes.md` first — it is written exactly for this): `Application` composition root owning `Window` → `Simulation` → `Renderer`; a State design pattern in `src/state/` (one `State` subclass per mode: Navigation/Record/Playback/Pick); `src/vision/Pnp.cpp` solving PnP with SQPnP; three shaders (terrain, pick, point); a terminal menu for terrain selection.
- **Modes A and B are done** (Navigation/Record/Playback + Picking with PnP, ghost-terrain diff view).
- **Modes C and D do not exist yet** — no `TrackersState`, no feature-matching code, no lighting model. That is the remaining work, planned below.

## PLAN — Remaining Phases

### Phase 4: Shared pose-comparison infrastructure
Both C and D record `(true pose, computed pose)` pairs per timestep and display them identically. Build once:
- A `PoseLog` (vector of `{Waypoint truePose; std::optional<Waypoint> computedPose}` + current index) owned by the mode states.
- Global view: draw two fly-through paths in distinct colors (true vs computed) — `Renderer::drawPath`/`drawWaypoints` already exist, call them per path.
- Player view: N/M keys step through timesteps; render the scene from the true pose and overlay the computed pose as a translucent ghost (`Renderer::drawGhost` already exists).
- Small carry-forward cleanup: rename `Mesh::width/height` → `cols/rows` (they are vertex counts, not world sizes — confirmed confusing during the migration; still unfixed upstream).

### Phase 5: Mode C — Trackers
- **Tracker representation:** N spheres with unique flat colors, placed on the terrain surface (random placement to start; positions and colors stored in the mode state / Simulation).
- **Rendering:** one shared UV-sphere mesh (VAO/VBO built once), drawn per tracker with a model matrix + color uniform. Rendered in both views.
- **Automatic correspondence (the core of the mode):** on 'B' — read back the player viewport (`glReadPixels` → `cv::Mat`, same mechanism as `Renderer::pickVertex` but full-frame), find each tracker's color blob, take its centroid as the 2D image point; pair with the sphere's known 3D center. With ≥4 visible trackers, solve PnP via the existing `computeCameraPose`; log the timestep in the PoseLog.
- **Edge cases:** fewer than 4 trackers visible (skip + console message), color palette chosen far apart so blob detection is unambiguous, occluded trackers naturally absent from the read-back.

### Phase 6: Lighting (prerequisite for Mode D's experiment)
- Compute per-vertex normals from the height map (central differences in `TerrainLoader` or at upload).
- Extend the terrain shader with a directional light (Lambert diffuse + ambient; uniforms for direction, color, intensity).
- Runtime control to change the light (keybinding or menu option) so Pre and Run lighting can differ in Mode D.

### Phase 7: Mode D — 2D Feature Matching
- **Pre-phase:** select n views (e.g. from recorded waypoints). For each: capture the rendered player view to `cv::Mat`, detect ORB keypoints + descriptors (SIFT as fallback if ORB matches poorly), and map each keypoint to 3D by rendering one id-color pass (existing pick shader) over the full viewport and looking up the terrain vertex under each keypoint pixel. Store as a `FeatureDb` (descriptors + 3D anchors).
- **Run-phase:** on 'B' — record true pose; capture the current view, detect features, match against the `FeatureDb` (Brute-Force Hamming + Lowe ratio test), then `cv::solvePnPRansac` for outlier-robust pose; log the timestep.
- Displays come free from Phase 4.

### Phase 8: Experiments, stability, presentation
- Run all modes on ≥2 terrains (`assets/terrains/` already has terrain1.jpg + terrain2.png; the menu supports any image dropped in).
- Mode D experiment: change lighting between Pre and Run; document how matching degrades/holds up.
- Documentation pass in `docs/`, bug-hunt for stability (grading weighs a stable app and GitHub documentation), demo flow for the final presentation.

## Workflow Notes
- Work happens on feature branches off `main` (kept in sync with `upstream/main` = Oren's repo); PRs go upstream via the `Coffee4Cup` fork. `CLAUDE.md` is gitignored upstream and tracked only on the personal `itay` branch.
- Build & run: WSL2 Ubuntu-22.04, `make && ./bin/drone_sim` (WSLg display). Visual checks need the user.
- Do not modify vendored code: `src/engine/` (BasicOpenGL classes), `src/glad.c`, `include/`.

## AI Assistant Persona & Directives
* **Role:** You are a senior software engineer and mentor.
* **Pedagogy First:** I am doing this project to learn. Please explain the reasoning behind architectural decisions, modern programming principles, and the mathematics of the computer vision operations.
* **Collaborative Pace:** Do not output massive blocks of rewritten code all at once. Propose a structural change, explain its benefits, ask for my thoughts, and then we will implement it together piece by piece.
* **Best Practices:** Guide me toward modern C++ standards, smart memory management, and clear separation of concerns (e.g., decoupling rendering logic from OpenCV logic).
