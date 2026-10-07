## Purpose

The Pose Estimation Model estimates the 6D pose of the target object after it has been detected by the [[Instance Segmentation Model (ISM)]].

A 6D pose contains:

- 3D rotation;
- 3D translation.

The predicted pose transforms points from the object coordinate system into the camera coordinate system:

`x_camera = R × x_object + t`

where:

- `R` is a 3 × 3 rotation matrix;
- `t` is a 3D translation vector, expressed in millimetres in the final pipeline output.

---

## Inputs

The PEM receives:

- an RGB image;
- a depth map;
- camera intrinsic parameters;
- the target CAD model;
- rendered object templates;
- ISM detections from `detection_ism.json`.

Each ISM detection contains:

- a segmentation mask in COCO RLE format;
- a bounding box;
- an ISM confidence score.

The PEM can process every detection above a threshold or only the top `K` candidates.

---

## 1. Detection Selection

The PEM starts from the detections produced by the ISM.

For each detection:

1. The COCO RLE segmentation is decoded into a binary mask.
2. The mask is intersected with valid depth pixels.
3. Detections with too few valid depth points are rejected.
4. Remaining detections can be sorted by their ISM score.
5. Optionally, only the top `K` candidates are passed to the pose model.

This stage prevents pose estimation from running on masks that do not contain enough valid 3D information.

---
## 2. Observed Object Point Cloud

The depth map is converted into a 3D point cloud using the camera intrinsics.

For a pixel `(u, v)` with depth `z`, the camera-frame 3D point is:

x = (u - cx) × z / fx
y = (v - cy) × z / fy
z = z
where fx, fy, cx, and cy are the camera intrinsic parameters.
Only points inside the ISM mask are retained. Therefore, the segmentation result determines which depth pixels are considered part of the object.

---
## 3. Depth Filtering and Point Sampling
The observed point cloud is filtered before being processed by the neural model.
The pipeline:
5. Computes depth percentiles for the masked point cloud.
6. Retains points between the 10th and 70th depth percentiles.
7. Estimates the object centre from the remaining points.
8. Rejects points outside a radius derived from the CAD model size.
9. Randomly samples a fixed number of observed points.

The filtering reduces the influence of depth outliers, background leakage in the mask, and points that are too far from the estimated object centre.
The object RGB crop is resized and masked using the same detection region. Pixel indices are retained to associate visual features with the sampled 3D points.

---
## 4. Feature Extraction
The model extracts joint visual and geometric features from both the CAD templates and the observed object region.
The feature extraction stage receives:
- the masked RGB template crops;
- sampled 3D points from the template CAD surface;
- the masked RGB observation;
- sampled 3D points reconstructed from depth.

A ViT-based encoder extracts image features. These features are associated with the corresponding sampled 3D points, creating a representation that combines appearance and geometry.

---
## 5. Geometric Structure Embedding
The model computes geometric structure embeddings for template and observed point clouds.
These embeddings describe spatial relationships between points, rather than relying only on their absolute coordinates.
This is useful because the observed object can appear at any position and orientation in the camera frame. The relative geometry of the object, however, remains consistent with the CAD model.

---
## 6. Coarse Point Matching
The first matching stage establishes global correspondences between:
- sparse points sampled from the CAD template;
- sparse points sampled from the observed RGB-D object.

The coarse stage identifies which regions of the observed point cloud are likely to correspond to regions of the CAD model.
This provides an initial geometric alignment hypothesis.

---
## 7. Fine Point Matching and Pose Estimation
The second stage refines the correspondences using denser visual and geometric features.

It produces a more accurate alignment between the observed object and the CAD model. From the final correspondences, the pipeline estimates:
- the rotation matrix R;
- the translation vector t;
- a pose confidence score.

The translation is converted to millimetres for the final output.

---
## Outputs
For the best pose candidate, the PEM outputs:
- R: 3 × 3 object-to-camera rotation matrix;
- t: object-to-camera translation vector in millimetres;
- pose_score: confidence score of the pose hypothesis;
- the associated ISM mask and bounding box.