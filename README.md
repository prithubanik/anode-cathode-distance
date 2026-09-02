# Anode–Cathode Distance — Computer Vision

This repository contains computer-vision experiments for **anode/cathode segmentation and distance calculation**.

## Structure

```text
anode-cathode-distance/
├── mask2former/          # Mask2Former-based anode/cathode segmentation
├── yolo/                 # YOLO-based detection/segmentation and distance calculation
├── sam/                  # Reserved for SAM-based experiments
├── maskrcnn/             # Reserved for Mask R-CNN experiments
├── unet/                 # Reserved for U-Net experiments
└── other/                # Supporting experiments and error analysis
```

## Main focus

- Anode segmentation
- Cathode segmentation
- Anode/cathode top detection
- Weir detection
- Distance measurement
- Tiled high-resolution image processing
- Model comparison and error analysis

The repository is intentionally separate from the particle-analysis project.

> Before making this repository public, check notebooks for datasets, credentials, API tokens, and machine-specific paths.
