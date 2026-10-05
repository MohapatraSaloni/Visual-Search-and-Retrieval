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
