# Kibo-RPC — On-Device Vision for the ISS Astrobee Free-Flyer

Android flight software for the **JAXA/NASA Kibo Robot Programming Challenge**, in which a submitted APK autonomously flies the *Astrobee* free-flying robot around the ISS Kibo module, identifies items posted at each target area from the robot's NavCam, and reports them to the ground system.

The navigation is the easy part. The vision is not: the camera sees each item board from a steep, unpredictable angle, there is no public dataset for the eleven item types, and inference must run on an ARM Android payload with no GPU and no network connection.

This repository contains the full solution — a synthetic data pipeline that produced **40,000 auto-labeled training images**, a YOLOv8n detector fine-tuned on them and exported to TensorFlow Lite for on-device inference, and an ArUco-marker geometry routine that rectifies and crops the item board from any viewing angle so the robot doesn't burn mission time repositioning itself.

---

## Results

Best run (`yolov8n_custom6`), evaluated on a held-out validation split:

| Metric | Score |
| --- | --- |
| Precision | **0.990** |
| Recall | **0.958** |
| mAP@50 | **0.988** |
| mAP@50–95 | **0.973** |

Training config: YOLOv8n fine-tuned from an earlier 10-epoch run, 50 epochs, 320×320 input, batch size 64. Thirteen training runs across YOLOv5n and YOLOv8n are kept in `yolo models (used)/detect/` with their full metric histories, PR/F1 curves and confusion matrices.

![Training curves](yolo%20models%20%28used%29/detect/yolov8n_custom6/results.png)

The deployed model ships as a ~12 MB `.tflite` file in the APK assets and runs entirely on the robot.

**Classes (11):** `coin` · `compass` · `coral` · `crystal` · `diamond` · `emerald` · `fossil` · `key` · `letter` · `shell` · `treasure_box`

---

## How it works

### 1. Synthetic training data — 40,000 images, zero hand-labeling

No dataset exists for these items, and hand-labeling at this scale wasn't realistic, so the training set was generated programmatically (`python helper files/`):

- **Asset prep** — source photos are background-removed (`rembg` + OpenCV/PIL) into transparent PNGs, then rotated through all 360° to give the model full rotation invariance, since items appear at arbitrary orientations relative to the robot.
- **Augmentation** — Gaussian, salt-and-pepper, Poisson and speckle noise are applied to simulate the NavCam's grainy, low-light imagery.
- **Scene composition** — `generate_training_images.py` composites randomly chosen, randomly scaled items onto a 2970×2100 canvas, rejecting placements whose IoU with an already-placed item exceeds 0.16 so objects crowd realistically without occluding each other into uselessness.
- **Labels for free** — because the generator knows where it placed each item, YOLO-format bounding boxes are written alongside every image. Additional scripts handle normalization and the train/validation/test split.

### 2. Detection model

The first attempt used the TensorFlow Object Detection API (kept in `AI model (tensorflow) unused/` for reference) but was dropped for size and latency reasons. The final pipeline trains YOLOv8n with Ultralytics and exports to TensorFlow Lite, which gave a model small and fast enough for the Astrobee payload while holding accuracy. On the robot it is loaded through the TFLite Task Vision `ObjectDetector` with a 0.4 confidence threshold, returning the top 5 detections per frame.

### 3. ArUco-anchored cropping — reading the board from any angle

The single biggest accuracy win, implemented in `YourService.cropImageFull()`:

1. Undistort the NavCam frame using the camera intrinsics and distortion coefficients.
2. Detect the `DICT_5X5_250` ArUco marker printed on the board and read its corner coordinates.
3. Derive the in-plane rotation from the marker's edge vector, rotate the frame both candidate directions, and keep whichever leaves the marker most axis-aligned.
4. Classify the board's orientation into one of four cases from the corner geometry, then crop the item region as a multiple of the marker's own edge length — the marker is a known physical size, so it doubles as a scale reference at any distance.

The result is a tightly framed, upright, scale-normalized crop fed to the detector. This let the robot classify items from its approach pose instead of flying closer and re-aiming at every area, saving mission time that the competition scores directly.

`Java-Crop-Image/` is a small desktop harness for iterating on this geometry against saved frames without running the full simulator.

### 4. Mission logic

`YourService.java` drives the run: start the mission, `moveTo()` each of the four target areas with a tuned pose and quaternion, capture and settle the NavCam, crop and detect, separate landmark items from treasure items, count quantities, and report each area via `setAreaInfo()` before completing the round and returning to the astronaut to notify recognition and snapshot the target item.

---

## Repository layout

| Path | Contents |
| --- | --- |
| `Sample-Apk/` | The Android submission. `YourService.java` (mission + vision pipeline), `YoloObjDetect.java` (TFLite inference wrapper), `Crop.java` / `CropImage.java` (ArUco cropping), `.tflite` models in `app/src/main/assets/`. |
| `python helper files/` | Synthetic dataset generation, background removal, rotation, noise augmentation, label normalization, dataset splitting, plus `data.yaml` / `labels.pbtext`. |
| `yolo models (used)/` | Thirteen YOLOv5n/YOLOv8n training runs with weights, metrics, curves and confusion matrices. |
| `AI model (tensorflow) unused/` | Earlier TensorFlow Object Detection API model, kept for reference. |
| `training_images_examples/` | Sample generated images with their YOLO label files. |
| `Java-Crop-Image/` | Standalone harness for developing the ArUco crop logic off-robot. |

## Tech stack

**Java** (Android, minSdk 24) · **Python** · **YOLOv8n / YOLOv5n** (Ultralytics) · **TensorFlow Lite** 2.10 + Task Vision 0.4 · **OpenCV** 4.5.3 (contrib, for ArUco and `calib3d`) · **rembg**, **PIL**, **NumPy** · **rosjava** via the JAXA Kibo-RPC API · **Gradle**

## Building

The APK builds against the JAXA-supplied Kibo-RPC SDK:

```bash
cd Sample-Apk
./gradlew assembleDebug
```

`guest_science_library`, `kibo_rpc_api` and `opencv-490` are the competition-provided modules; the `.tflite` weights are already in `app/src/main/assets/` and are excluded from compression by the `aaptOptions` block so they can be memory-mapped at runtime.

To retrain:

```bash
yolo detect train model=yolov8n.pt data="python helper files/data.yaml" epochs=50 imgsz=320 batch=64
```

## Known limitations

- The dataset is fully synthetic — composited items on a white canvas — so it trades some real-world background realism for scale and perfect labels.
- The crop geometry assumes exactly one visible ArUco marker per area and falls back to an uncropped frame if detection fails.
- Dataset paths in the Python scripts are absolute Windows paths from the machine they were written on and need updating before rerunning.
