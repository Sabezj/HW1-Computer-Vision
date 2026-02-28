# HW1 — Optical Flow & Video Dynamics (Variants A + B)

This repository contains a complete implementation of the Optical Flow & Video Dynamics homework. The main code is provided as a reproducible notebook, and all required visualizations (and the stabilized video for A) are generated automatically into the `outputs/` folder.

## Repository structure

- `src/`
  - `HW1.ipynb` (main notebook: Variant A and Variant B pipelines)
- `outputs/`
  - `A/` (Variant A visualizations + stabilized video)
  - `B/` (Variant B visualizations + LK vs Farnebäck comparison plots)


## Requirements

- Python 3.9+
- `opencv-python`
- `numpy`
- `matplotlib`

Running in **Google Colab** is supported.

## How to run

1. Open `src/HW1.ipynb` (locally or in Google Colab).
2. Run all cells from top to bottom.
3. Outputs will be saved into:
   - `outputs/A/` for Variant A
   - `outputs/B/` for Variant B

If the repository does not include input videos, the notebook will download the required sample videos automatically. If downloading is blocked, place your own video into `data/` and update the input path in the notebook.


# Variant A — Video Stabilization (Global Motion Compensation)

## Goal
Estimate global camera motion between consecutive frames using sparse features + tracking, smooth the motion trajectory, and warp frames to produce a stabilized video.

## Pipeline (high level)
1. Convert frames to grayscale.
2. Detect features (`goodFeaturesToTrack`) with balanced spatial sampling.
3. Track features with **Lucas–Kanade (LK)** optical flow.
4. Filter tracks using:
   - forward–backward (FB) consistency
   - robust estimation (**RANSAC**) for the global transform
5. Accumulate per-frame transforms into a camera motion trajectory.
6. Smooth the trajectory (moving average / robust smoothing).
7. Compute stabilization warp and apply `warpAffine` with safe borders.
8. Auto-crop to remove invalid borders, then resize back to original size.
9. Save stabilized video and required figures.

## Key parameters
- Feature detection: number of corners, quality level, min distance, grid sampling
- LK: window size, pyramid levels, termination criteria
- RANSAC: reprojection threshold, confidence
- Smoothing: window length / smoothing strength
- Cropping: robust crop using percentiles to avoid shrinking due to rare bad frames
- Border handling: constant borders (black) to avoid mirrored “tiling” artifacts

## Outputs (saved to `outputs/A/`)
- `stabilized.mp4` — final stabilized video
- `before_after.png` — side-by-side comparison of selected frames (INPUT vs STABILIZED)
- `camera_trajectory.png` — estimated and smoothed camera motion trajectory
- `warping_examples.png` — examples showing the effect of warping
- `error_analysis.png` — diagnostic plot showing residual motion/error before vs after stabilization

## Notes / Observations
- Stabilization is effective when motion is approximately global and planar.
- Failure cases include:
  - large parallax (foreground/background move differently)
  - rolling shutter artifacts
  - severe blur / defocus segments
- Mitigations used:
  - FB-check + RANSAC to reject bad tracks/outliers
  - robust auto-crop (percentiles) to avoid over-cropping due to a few bad frames
  - anchor stabilization logic to reduce slow drift when a stable reference region exists

---

# Variant B — Optical Flow Comparison (LK vs Farnebäck)

## Goal
Compare sparse optical flow (LK trajectories) and dense optical flow (Farnebäck), visualize both, and extract a motion mask. Provide a short comparative report and sensitivity test.

## Pipeline
### 1) Sparse flow (LK)
- Detect feature points with `goodFeaturesToTrack`
- Track with LK across frames
- Apply FB-check to discard unstable tracks
- Visualize short trajectories (fixed trail length) to keep plots readable

### 2) Dense flow (Farnebäck)
- Compute dense optical flow between consecutive frames
- To avoid “everything moving” due to camera motion, the dense flow is computed on motion-compensated frames (global alignment first), producing a **residual flow** that highlights object motion

### 3) Motion mask
- Convert residual flow magnitude to a binary mask using a threshold
- Apply morphological filtering to remove small noise blobs
- Report basic mask statistics (e.g., number of connected components)

### 4) Sensitivity experiment
- Use the **same initial set of feature points** across conditions
- Do **not** reinitialize or add new points
- Compare survival (%) of tracked points over time under:
  - original
  - blur
  - noise
  - shadow/illumination change

## Outputs (saved to `outputs/B/`)
- `lk_trajectories.png` — LK tracks visualization
- `dense_flow.png` — dense residual flow visualization
- `motion_mask.png` — motion segmentation mask example
- `sensitivity.png` — LK tracking survival curves + mask component comparison

---

## LK vs Farnebäck — Comparative report (short)

**Lucas–Kanade (LK)**
- Sparse: tracks only selected feature points (corners/high texture)
- Efficient, interpretable (trajectories)
- Works well when features are stable and motion between frames is moderate
- Can fail when points are lost due to blur/defocus or textureless regions

**Farnebäck**
- Dense: produces flow for (almost) every pixel
- Better for visualizing motion fields and building motion masks
- More sensitive to noise, illumination changes, and camera motion
- Without global motion compensation, dense flow often indicates motion across the entire frame (camera shake), making motion masks noisy

**Practical takeaway**
- LK is best for stable feature tracking and global motion estimation.
- Farnebäck is best for dense motion visualization and segmentation, but should be used with preprocessing (alignment/denoising) when camera motion exists.
