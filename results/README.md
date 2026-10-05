# Results

This directory contains selected figures and numerical result summaries
from the experiments reported in the project.

## Figures

- `framework.png` — Overview of the retrieval-based multispectral localization framework.
- `similarity_distribution.png` — Retrieved-pair and random-pair cosine similarity distributions.
- `retrieval_behavior.png` — Retrieval similarity and target-image coverage as retrieval depth increases.
- `query_perturbation_stability.png` — Query stability under controlled perturbations.
- `inference_sensitivity.png` — Sensitivity to retrieval depth, similarity-threshold offsets, and NMS IoU.

## Tables

- `retrieval_depth.csv` — Retrieval similarity and number of unique target images across retrieval depths.
- `query_perturbation_stability.csv` — Embedding and retrieval stability under query perturbations.
- `multi_query_consistency.csv` — Same-class retrieval consistency measured using Jaccard@100.
- `scalability.csv` — Gallery construction time, query latency, and memory measurements at the evaluated gallery sizes.

The repository does not include the underlying satellite imagery,
challenge annotations, model checkpoints, embeddings, or FAISS indexes.
