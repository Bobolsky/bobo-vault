## Purpose
The Instance Segmentation Model identifies pixels belonging to the target object in an RGB-D image. Its output is a set of ranked object-mask candidates, which are passed to the Pose Estimation Model.

## Inputs
- [^3]RGB image
- Depth map, expressed in millimetres
- Camera intrinsic parameters
- CAD model of the target object
- Rendered CAD templates from multiple viewpoints

---

## 1. Template Preparation

Before inference, the CAD model is rendered from multiple viewpoints.

For each viewpoint, the pipeline stores:

- an RGB template;
- a binary object mask;
- [^1]NOCS coordinates, which associate pixels with object-centred 3D coordinates.

These templates provide the visual reference used to recognise the object in the input image.

---

## 2. Mask Proposal Generation

The RGB image is processed by a foundation segmentation model:

- **SAM**, or
- **[[FastSAM]]**.

The selected model generates class-agnostic mask proposals. At this stage, the model does not know the target object class and does not use depth information.

A proposal may correspond to the target object, a background region, another object, or a fragment of the target object.

![[Pasted image 20261007102823.png|270]]

---
## 3. Visual Matching with CAD Templates

Each proposed mask is used to crop an image region from the input RGB frame.

DINOv2 extracts visual descriptors from:

- the cropped proposal;
- the rendered CAD templates.

The descriptors are compared to estimate whether the proposal has an appearance consistent with the target object.

This stage provides semantic and appearance similarity scores.
The goal here is to determine which mask is likely to contain the target object. Every mask is compared with every template of the CAD.

---
## 4. Use of Depth Information

Depth is not used to generate the masks. It is used after mask generation to verify whether a proposal is geometrically plausible.

For each candidate mask, valid depth pixels are combined with the camera intrinsics to recover 3D information in the camera frame.

Depth supports the following checks:

- estimating the physical size of the segmented region;
- comparing its observed 3D extent with the CAD model dimensions;
- evaluating geometric consistency between the proposal and the expected object shape;
- detecting masks that include background pixels at a substantially different depth;
- penalising candidates whose visible object area is inconsistent with the rendered templates.

An optional size filter rejects masks whose depth-derived extent is incompatible with the CAD model diameter.

For this comparison to be meaningful, the CAD model and depth map must use consistent real-world units, typically millimetres.

---

## 5. Candidate Scoring and Selection

Each mask proposal receives several scores:

- **Semantic score**: similarity between the proposal and the template features;
- **Appearance score**: visual consistency with the target object;
- **Geometric score**: compatibility with depth and CAD geometry;
- **Visible-ratio score**: estimated visible fraction of the object.

The final score combines visual similarity with geometric and visibility weights:

`final_score = mean(semantic, appearance) × geometry_weight × visibility_weight`

The highest-ranked candidates are retained as ISM detections.

---
## Output
The model produces `detection_ism.json`, containing for each accepted candidate:
- a segmentation mask encoded as [^2]COCO RLE;
- a 2D bounding box;
- an ISM confidence score;
- the object category identifier.

## Scoring
The final score combines semantic and appearance similarity, then weights them using geometric consistency and the visible ratio:

==🟡`final_score = mean(semantic, appearance) × geometry_weight × visibility_weight`==

This ranking step is important because generic segmentation models can propose many masks, while only some correspond to the CAD object.






[^1]: NOCS (Normalized Object Coordinate Space) represents 3D points in an object-centred coordinate system. Each visible pixel in a rendered template can be associated with a point on the CAD model surface.

[^2]: COCO (Common Objects in Context) is a standard format for dataset and computer vision results. COCO RLE (Run Lenght Encoding) is an encoding format, instead of saving a full binary image, RLE stores consecutives runs of background and foreground pixels. This makes segmentation masks compact while preserving the original image resolution. Pem afterwards decodes it.

[^3]: RGB image gets also preprocessed through Morphological Closing that is used to close any small gap or discontinuity. Sobel Border to better highlight the objects broder and produce more distinct masks.
