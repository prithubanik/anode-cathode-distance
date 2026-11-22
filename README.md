Anode–Cathode Segmentation and Distance Measurement
Computer-vision experiments for segmenting anode and cathode wires in high-resolution battery microscopy images and estimating their geometric separation. The project combines Mask2Former semantic segmentation with image-processing and path-tracing methods to convert predicted electrode masks into calibrated distance measurements.

The repository is intentionally separate from the battery-particle segmentation and morphology project.
Project objectives
- Segment anode and cathode wires in microscopy images.
- Detect the visible top/apex regions of each electrode wire.
- Identify the relevant anode intersection and trace the electrode geometry.
- Estimate anode–cathode overhang / separation distances in micrometres.
- Process large microscopy images using overlapping tiled inference.
- Compare automated segmentation and distance measurements with annotated and manually measured references.
- Keep experiments with Mask2Former separate from the reserved Mask R-CNN workstream.
Repository structure
anode-cathode-distance/
├── mask2former/          # Mask2Former training, inference and export experiments
├── maskrcnn/             # Reserved for Mask R-CNN experiments
└── README.md
Dataset files, trained weights, prediction outputs and machine-specific configuration are not assumed to be included in this public repository.
Data annotation and training labels
Anode and cathode wire regions were annotated as polygons using DigitalSreeni Image Annotator and exported in COCO annotation format.

The training data loader reads the COCO JSON, maps the two categories to semantic classes, and rasterises polygon coordinates into segmentation masks. Bounding boxes are used only as a fallback when polygon segmentation data is unavailable.

The training workflow includes:
- Separate semantic classes for anode and cathode.
- Polygon-derived pixel masks rather than box-only labels.
- Horizontal flipping, small translations/scaling, and elastic spatial augmentation.
- An 80/20 training/validation split with a fixed random seed (42) in the supplied experiment.
- Class-wise validation IoU and Dice calculations at a probability threshold of 0.50.
Annotation vs. reference measurement: DigitalSreeni Image Annotator was used to create polygon segmentation labels. ZEISS ZEN Core was used separately to obtain manual reference distance measurements for comparison with the automated distance-estimation pipeline. These are two different validation references and should not be described as the same annotation process.

Model and training approach
The main segmentation experiment fine-tunes Mask2Former with a Swin-Base backbone, initialised from the Hugging Face checkpoint:
facebook/mask2former-swin-base-coco-instance
The experiment uses a custom COCO-wire dataset loader and the Hugging Face Mask2FormerImageProcessor / Mask2FormerForUniversalSegmentation workflow. The training configuration in the notebook includes:
- AdamW optimisation with separate learning rates for the backbone (1e-5) and decoder (1e-4).
- Weight decay of 0.05.
- Mixed-precision training (fp16).
- Training and evaluation batch sizes of 24 per device.
- Evaluation and checkpoint saving each epoch.
- Best-checkpoint selection based on evaluation loss.
- Early stopping with a patience of 50 epochs (maximum configured epochs: 400).
These are the settings in the current experiment notebook; adjust them to match the available GPU memory and dataset version when reproducing the run.
Segmentation performance
The project-reported validation results for anode/cathode wire segmentation are:
Metric	Reported result
Intersection over Union (IoU)	~0.92
Dice coefficient	~0.96


The notebook calculates IoU and Dice separately for the anode and cathode classes using thresholded semantic probabilities. The values above are the reported project results; when publishing a formal benchmark, also record the dataset version, number of validation images, per-class scores and the exact aggregation method.

Evaluation implementation note: In the current training notebook, the augmentation transform is attached to the dataset before the 80/20 split. Consequently, the validation subset may also receive stochastic augmentation. For a strictly held-out evaluation, create separate training and validation dataset instances and disable augmentation for validation, then rerun and record the final metrics.
High-resolution tiled inference
To process large microscopy images without resizing the entire image to the model input size, inference uses overlapping sliding windows.
Parameter	Current experiment setting
Tile size	540 × 540 pixels
Tile overlap	150 pixels
Tile stride	390 pixels
Segmentation score threshold	0.49
Physical calibration	1.7943 µm/pixel


Each tile is processed by Mask2Former. The class probability maps are resized to the tile dimensions and accumulated into full-resolution anode and cathode masks. The overlapping tiles help reduce missed regions at tile boundaries. A mask-cleaning step removes pixels assigned to both wire classes before distance tracing.

The current inference configuration uses the following semantic mapping:
- Anode wire: class ID 1
- Cathode wire: class ID 3
Distance measurement pipeline
The distance-estimation stage combines the segmentation masks with classical image processing and path analysis:
1. Wire component detection: connected components are identified independently for anode and cathode masks; very small cathode components are filtered.
2. Physical top/apex refinement: grayscale thresholding combines Otsu and adaptive Gaussian thresholding, followed by morphological closing. This helps refine the visible physical apex within each segmented component.
3. Intersection detection: a horizontal scan finds the anode-mask intersection associated with a cathode top, with a limited vertical search tolerance for small mask gaps.
4. Hybrid skeleton construction: thresholded bright image pixels are intersected with a dilated anode mask, morphologically closed and skeletonised.
5. Path tracing: skimage.graph.route_through_array routes from the anode intersection towards the refined apex. The cost map favours the skeleton, permits movement within the segmented wire at a higher cost, and penalises leaving the wire region.
6. Path smoothing and calibration: the traced path is simplified with OpenCV polygon approximation. Segment lengths are summed in pixels and multiplied by the physical pixel calibration to obtain micrometres.
The output overlay visualises the predicted wire masks, horizontal intersection segment, traced path, detected points and the estimated distance in micrometres.
Comparison with ZEISS ZEN Core reference measurements
The automated distance-measurement pipeline was compared with manual reference distances measured in ZEISS ZEN Core on original-resolution microscopy images. The project reports approximately 98% measurement accuracy/agreement against those human reference measurements.

This ~98% figure refers to the distance-measurement comparison, not to segmentation IoU, Dice, or the accuracy of the DigitalSreeni polygon annotations. For a fully reproducible report, document the number of compared measurements and the precise calculation used for the percentage (for example, relative error or agreement within a tolerance).
ONNX export experiment
An ONNX export workflow is included as an experiment for later deployment. The notebook currently demonstrates successful ONNX graph checking. However, the recorded run did not complete ONNX Runtime inference because the exported input signature and runtime feed differed (pixel_mask was reported as an invalid input name). Therefore, ONNX Runtime execution and numerical parity are not yet claimed as validated.
Main tools and libraries
- Deep learning: PyTorch, Hugging Face Transformers, Mask2Former (Swin-Base)
- Annotation and data: DigitalSreeni Image Annotator, COCO JSON
- Image processing: OpenCV, NumPy, Albumentations
- Morphology and path tracing: scikit-image (skeletonize, route_through_array)
- Model export experiment: ONNX, ONNX Runtime
- Reference measurement software: ZEISS ZEN Core
Reproducibility and data handling
The notebooks currently contain local paths and assume access to the relevant microscopy datasets and trained checkpoints. Before reproducing an experiment, update the configuration paths and confirm that the pixel-to-micrometre calibration matches the image acquisition settings.

Before making this repository public:
- Remove or replace machine-specific paths, usernames, network-share locations and private file names.
- Do not commit proprietary microscopy images, confidential research data, COCO annotations derived from restricted data, or trained checkpoints unless permission has been granted.
- Check notebooks and exported configuration files for credentials, tokens and internal URLs.
- Record the dataset version, split, model checkpoint, class mapping, calibration and evaluation protocol for each reported result.
Current scope and next steps
- Complete and document the ONNX Runtime input-signature validation and PyTorch/ONNX numerical comparison.
- Add a reproducible evaluation report with per-class IoU/Dice, validation-set size and the exact calculation behind the ZEISS ZEN Core distance-agreement figure.
- Add Mask R-CNN experiments under maskrcnn/ when that workstream is implemented.
- Add anonymised example images and prediction overlays only where data-sharing permissions allow.
