# Visual Search and Retrieval for Multispectral Satellite Imagery

A self-supervised visual retrieval framework for candidate localization in four-channel multispectral satellite imagery.

The project formulates localization as **query-driven similarity retrieval** rather than supervised bounding-box prediction. A modified four-channel ResNet-50 is trained with a MoCo-v2-based momentum-contrastive objective to learn compact patch representations. The resulting embeddings are indexed with FAISS, and query exemplars are used to retrieve visually similar regions from a target-image gallery. Retrieved patches are then converted into spatial candidate regions using their stored image coordinates, similarity filtering, and non-maximum suppression.

---

## Overview

Object localization in remote-sensing imagery is often formulated as a supervised detection problem requiring object-level annotations. In the dataset used in this project, independent ground-truth annotations for the target imagery are unavailable.

To operate under this constraint, the project uses an **exemplar-based retrieval formulation**:

```text
Four-Channel Multispectral Imagery
              │
              ▼
      Patch Extraction
         128 × 128
        stride = 64
              │
              ▼
     Four-Channel ResNet-50
              │
              ▼
      128-D Embedding
       L2 Normalized
              │
              ▼
       FAISS IndexFlatIP
              │
              ▲
              │
       Query Exemplar
              │
              ▼
       Top-K Retrieval
              │
              ▼
    Similarity Filtering
              │
              ▼
       Spatial NMS
              │
              ▼
    Candidate Regions
```

The resulting system is intended as a **retrieval-based candidate localization framework**, rather than a ground-truth-verified object detector.

---

## Key Components

- Four-channel multispectral input: **Blue, Green, Red, NIR**
- Modified **ResNet-50** for four-channel input
- 2048-dimensional backbone representation
- Projection head: **2048 → 2048 → 128**
- 128-dimensional L2-normalized embeddings
- **MoCo-v2-based momentum-contrastive learning**
- Contrastive queue size: **65,536**
- Temperature: **0.2**
- **AdamW** optimizer
- Learning rate: **1 × 10⁻⁴**
- Weight decay: **1 × 10⁻⁴**
- Batch size: **32**
- Training: **360 epochs**
- Mixed-precision training
- Cosine learning-rate scheduling
- Exact **FAISS IndexFlatIP** search
- Baseline retrieval depth: **K = 500**
- NMS IoU threshold: **0.40**
- Maximum **50 detections per class per target image**

---

## Dataset

The project uses a four-channel multispectral satellite-imagery challenge dataset.

### Dataset Composition

| Component | Configuration |
|---|---|
| Development images | 150 unlabeled four-band images |
| Target images | 40 four-band images |
| Spectral channels | B, G, R, NIR |
| Query chips | 221 |
| Query classes | 8 |
| Gallery embeddings | 32,542 |
| Patch size | 128 × 128 |
| Target stride | 64 pixels |
| Target overlap | 50% |

### Query Classes

```text
Brick Kiln
Metro Shed
Play Ground
Pond-1
Pond-2
STP
Sheds
Solar Panel
```

The development imagery is used for self-supervised representation learning. Target imagery is tiled to construct the retrieval gallery, while query exemplars are constructed from the available class-wise annotations.

### Data Availability

The original satellite imagery and challenge annotations are **not included in this repository**.

They should only be used in accordance with the applicable dataset and challenge terms.

---

## Multispectral Preprocessing

The input consists of four spectral bands in the following order:

```text
Blue
Green
Red
NIR
```

Raw pixel values are scaled using a divisor of `10,000` and then normalized using channel-wise statistics estimated from the development imagery.

The exact preprocessing configuration is stored in:

```text
preprocess.json
```

This file contains:

- input band order
- scaling divisor
- channel-wise means
- channel-wise standard deviations

The preprocessing configuration should be treated as the source of truth for the normalization stage.

---

## Model

The model is based on ResNet-50 modified to accept four-channel multispectral input.

The classification layer is discarded, producing a **2048-dimensional backbone representation**.

A two-layer projection head maps the representation as:

```text
2048 → 2048 → 128
```

with Batch Normalization and ReLU between the projection layers.

The final 128-dimensional embedding is L2 normalized before similarity search.

---

## Self-Supervised Training

The representation-learning stage uses a **MoCo-v2-based momentum-contrastive framework**.

Two augmented views of each training patch are processed by a query encoder and a momentum-updated key encoder.

### Training Configuration

| Parameter | Value |
|---|---|
| Method | MoCo-v2 |
| Epochs | 360 |
| Batch size | 32 |
| Queue size | 65,536 |
| Temperature | 0.2 |
| Optimizer | AdamW |
| Learning rate | 1 × 10⁻⁴ |
| Weight decay | 1 × 10⁻⁴ |
| LR schedule | Cosine |
| Mixed precision | Enabled |

The training pipeline uses geometric and radiometric augmentations including crop-and-resize, horizontal and vertical flips, rotations, brightness and contrast variation, blurring, additive noise, NIR-channel dropout, and random erasing.

---

## Gallery Construction

After representation learning, the encoder is frozen and applied to the target imagery.

Each target image is divided into overlapping patches using:

```text
Patch size = 128 × 128
Stride     = 64 pixels
Overlap    = 50%
```

For every gallery patch, the system stores its embedding together with spatial metadata:

```text
image_id
x-coordinate
y-coordinate
patch size
```

The final gallery used in the experiments contains:

```text
40 target images
32,542 indexed patch embeddings
```

The embeddings are stored in an exact FAISS `IndexFlatIP` index.

Because the embeddings are L2 normalized, inner-product ranking is equivalent to cosine-similarity ranking.

---

## Query-Based Localization

A query exemplar is processed using the same preprocessing and embedding pipeline.

The query embedding is compared against the gallery using inner-product similarity:

```text
similarity = query_embedding · gallery_embedding
```

The top-`K` retrieved patches are converted into candidate spatial regions using their stored coordinates.

The baseline inference configuration is:

```text
Retrieval depth K = 500
NMS IoU           = 0.40
Maximum detections = 50 per class per target image
```

Similarity filtering is applied before spatial consolidation, followed by class-wise spatial non-maximum suppression.

The resulting regions are **retrieval-derived candidate locations**, not verified detections against independent target ground truth.

---

## Experimental Evaluation

Independent ground-truth annotations for the target imagery are unavailable. Therefore, the experiments do not use target-box IoU, AP, mAP, precision, or recall as measures of target localization accuracy.

Instead, the project evaluates several properties of the retrieval pipeline:

- Representation similarity
- Retrieval concentration
- Query perturbation stability
- Multi-query consistency
- Inference-parameter sensitivity
- Retrieval computation and scalability

---

## Main Results

### Representation Similarity

Mean retrieval similarity remains very high as retrieval depth increases.

| K | Mean Similarity |
|---:|---:|
| 1 | 0.999799 |
| 10 | 0.999768 |
| 50 | 0.999738 |

Increasing retrieval depth therefore does not immediately introduce substantially low-similarity candidates.

---

### Retrieval Concentration

Increasing retrieval depth causes the retrieved set to cover more target images and broader spatial regions.

| K | Mean Unique Target Images |
|---:|---:|
| 1 | 1.00 |
| 10 | 6.02 |
| 100 | 22.20 |
| 500 | 32.94 / 40 |

At `K = 500`, the mean spatial support is approximately:

```text
x-range ≈ 4296 px
y-range ≈ 3124 px
```

Thus, high embedding similarity does not necessarily imply spatial concentration around a unique target instance.

---

### Query Perturbation Stability

Query stability was evaluated under:

```text
Brightness
Contrast
Horizontal flip
Vertical flip
```

The results show that high embedding cosine similarity does not always correspond to stable retrieval neighborhoods. In particular, the evaluated geometric transformations produced substantially lower retrieval agreement than the photometric transformations.

---

### Multi-Query Consistency

Retrieval neighborhoods were compared between same-class query exemplars using Jaccard overlap at `K = 100`.

The experiment contains:

```text
5,149 same-class query pairs
```

with an overall mean:

```text
Jaccard@100 = 0.0560
```

and median:

```text
Jaccard@100 = 0.0
```

This indicates that different exemplars from the same semantic class can produce substantially different retrieval neighborhoods.

---

### Inference Sensitivity

The inference pipeline was evaluated by varying retrieval depth, similarity-threshold offsets, and NMS IoU.

Increasing retrieval depth from:

```text
K = 50 → 1000
```

increased the final candidate count from:

```text
6,015 → 14,803
```

while mean similarity changed only slightly.

For similarity-threshold offsets:

```text
-0.10
-0.05
 0.00
+0.05
+0.10
```

the final candidate count remained:

```text
13,291
```

For NMS IoU values from `0.2` to `0.7`, the measured effect on candidate count was limited.

---

## Repository Structure

```text
Visual-Search-and-Retrieval/
│
├── README.md
├── requirements.txt
├── .gitignore
├── preprocess.json
│
├── configs/
│   └── config.yaml
│
├── notebooks/
│   └── visual_search_retrieval.ipynb
│
└── results/
    ├── README.md
    ├── figures/
    └── tables/
```

### Main Files

**`notebooks/visual_search_retrieval.ipynb`**

The main research notebook containing preprocessing, representation learning, gallery construction, retrieval, localization, evaluation, and figure generation.

**`preprocess.json`**

Contains the exact multispectral preprocessing configuration, including band order, scaling, channel-wise means, and standard deviations.

**`configs/config.yaml`**

Contains the main model, training, gallery, retrieval, localization, and experimental configuration.

**`results/`**

Contains selected figures and numerical result summaries.

**`requirements.txt`**

Lists the Python dependencies required to run the notebook.

---

## Installation

Install the required Python packages with:

```bash
pip install -r requirements.txt
```

A CUDA-capable GPU is recommended for the self-supervised training stage.

---

## Running the Project

### 1. Obtain the Dataset

Obtain the required dataset through the appropriate authorized source.

The original challenge data are not distributed with this repository.

### 2. Open the Notebook

Open:

```text
notebooks/visual_search_retrieval.ipynb
```

The notebook contains the main implementation of the project.

### 3. Configure Dataset Paths

Update the dataset paths in the notebook to match your local environment.

Machine-specific paths, including personal Google Drive paths, should not be committed to the repository.

### 4. Run the Pipeline

The notebook covers:

```text
Data loading
    ↓
Four-channel preprocessing
    ↓
Self-supervised representation learning
    ↓
Target gallery construction
    ↓
FAISS index construction
    ↓
Query embedding
    ↓
Top-K retrieval
    ↓
Similarity filtering
    ↓
Spatial NMS
    ↓
Candidate localization
    ↓
Experimental analysis
```

---

## Results Directory

Selected experimental outputs are organized as:

```text
results/
│
├── README.md
│
├── figures/
│   ├── framework.png
│   ├── similarity_distribution.png
│   ├── retrieval_behavior.png
│   ├── query_perturbation_stability.png
│   └── inference_sensitivity.png
│
└── tables/
    ├── retrieval_depth.csv
    ├── query_perturbation_stability.csv
    ├── multi_query_consistency.csv
    └── scalability.csv
```

The repository does not include raw imagery, challenge annotations, model checkpoints, embeddings, or FAISS index files.

---

## Reproducibility

The repository provides the principal components needed to reproduce the implemented experiments:

- `notebooks/visual_search_retrieval.ipynb` contains the executable research pipeline.
- `preprocess.json` records the exact multispectral preprocessing parameters.
- `configs/config.yaml` records the principal model, training, retrieval, localization, and analysis settings.
- `results/` contains selected experimental outputs.

The underlying dataset is not redistributed with the repository.

For reproducibility, the notebook should be executed with the required dataset available and with dataset paths adjusted for the local environment.

---

## Limitations

The conclusions of this project are subject to several limitations.

- Independent target ground-truth annotations are unavailable, so absolute localization accuracy cannot be established.
- The evaluation is restricted to the available challenge imagery and query-exemplar setting.
- The retrieval process is dependent on the selected query exemplar.
- Same-class query exemplars can produce substantially different retrieval neighborhoods.
- The experiments evaluate a single four-channel ResNet-50 representation.
- Generalization across sensors, geographic regions, seasons, spatial resolutions, and public remote-sensing benchmarks is not established.

The system should therefore be interpreted as a **retrieval-based candidate localization framework under the evaluated setting**, rather than a fully validated object detector.

---

## License

**All rights reserved.**

The source code in this repository is provided for viewing and research reference only. No permission is granted to copy, modify, redistribute, publish, or use the code or substantial portions of the code for other projects without prior written permission from the author.

The dataset and challenge materials are not included in this repository and remain subject to their respective terms and restrictions.

---

## Citation

The associated research paper is currently under review.

Citation information will be added here when appropriate.

---
