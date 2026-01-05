# Epstein-Maxwell Document Similarity Network

An interactive visualization of document similarity and entity relationships within the Epstein-Maxwell Files corpus. This tool enables researchers, journalists, and legal professionals to explore topological structures in legal document collections using techniques from computational topology and network science. See: https://division-labs.github.io/epstein-maxwell-netviz/.

![Document Network Visualization](https://img.shields.io/badge/D3.js-v7-orange) ![License](https://img.shields.io/badge/License-MIT-blue) ![Static Site](https://img.shields.io/badge/Hosting-GitHub%20Pages-green)

---

## Table of Contents

1. [Overview](#overview)
2. [Features](#features)
3. [Technical Foundation](#technical-foundation)
   - [Document Similarity](#document-similarity)
   - [Persistent Homology](#persistent-homology)
   - [Optimal Threshold Selection](#optimal-threshold-selection)
4. [Data Pipeline](#data-pipeline)
5. [Usage Guide](#usage-guide)
6. [Network Statistics](#network-statistics)
7. [Deployment](#deployment)
8. [References](#references)
9. [License](#license)

---

## Overview

This visualization presents a force-directed graph of **471 legal documents** from the EFTA (Epstein Files Text Archive) corpus, with edges representing semantic similarity computed via TF-IDF cosine similarity on noun term-document matrices. The network reveals document clusters (communities), cross-cluster bridges, and entity co-occurrence patterns that illuminate the structure of this complex legal corpus.

The visualization offers two complementary views:

- **Document Similarity Network**: Documents as nodes, similarity as edges, colored by community, and featuring an entity overlay system with the persons mentioned in each document and their connection to other documents in the corpus
- **Entity Network**: People, organizations, and companies as nodes, with documented relationships as edges

---

## Features

### Interactive Network Exploration

- **Force-directed layout** with zoom, pan, and drag interactions
- **Community detection** using the Louvain algorithm with automatic coloring
- **Node sizing** based on degree centrality (connection count)
- **Real-time filtering** by similarity threshold

### Persistent Homology Integration

- **Topologically-informed threshold selection** using H₀ and H₁ persistent features
- **Visual threshold markers** indicating optimal operating points derived from persistence diagrams
- **Threshold slider** spanning 0.20–0.95 cosine similarity with homology-derived defaults

### Document Details Panel

- **Summary opener** extracted using intelligent text filtering (skipping headers, OCR artifacts)
- **Entity mentions** with frequency counts
- **Known relationships** between mentioned entities
- **Top noun terms** from TF-IDF analysis

### Entity Overlay System

- **Click-to-focus** on any document node
- **Entity halo** showing people/organizations mentioned in that document
- **Cross-document links** revealing which other documents share the same entities
- **Business associations** ranked by role importance (owner > founder > CEO > director > ...)

### Temporal Filtering

- **Six time periods** from Early Period (pre-2000) through Post-Trial (2022+)
- **Period-specific networks** showing document relationships within each era

### Search & Navigation

- **Full-text search** across EFTA IDs, entity names, and noun terms
- **Zoom-to-fit** centering the network view
- **Entity network toggle** to switch between document and entity relationship views

---

## Technical Foundation

### Document Similarity

Document similarity is computed using a standard information retrieval pipeline:

1. **Term-Document Matrix (TDM)**: Noun terms extracted via spaCy NER are organized into a sparse matrix where rows represent terms and columns represent documents.
2. **TF-IDF Weighting**: Raw term counts are transformed using Term Frequency–Inverse Document Frequency weighting to emphasize discriminative terms:

   $$\text{tf-idf}(t,d) = \text{tf}(t,d) \times \log\frac{N}{\text{df}(t)}$$

   where $\text{tf}(t,d)$ is the term frequency in document $d$, $N$ is the total number of documents, and $\text{df}(t)$ is the document frequency of term $t$.
3. **Cosine Similarity**: Pairwise document similarity is computed as the cosine of the angle between TF-IDF vectors:

   $$\text{sim}(d_i, d_j) = \frac{\mathbf{v}_i \cdot \mathbf{v}_j}{\|\mathbf{v}_i\| \|\mathbf{v}_j\|}$$

### Persistent Homology

Persistent homology is a technique from **topological data analysis (TDA)** that identifies topological features—connected components, loops, voids—that persist across multiple scales in data. For document similarity networks, we construct a **Vietoris-Rips filtration** by treating similarity as inverse distance:

1. **Filtration Construction**: Starting from isolated nodes (threshold = 1.0), we progressively add edges as we lower the similarity threshold toward 0.0.
2. **Homology Groups**:

   - **H₀ (0-dimensional)**: Tracks connected components. Features are "born" when a document appears and "die" when it merges with another cluster.
   - **H₁ (1-dimensional)**: Tracks loops/cycles. Features are born when a cycle forms and die when it gets "filled in" by additional edges.
3. **Persistence Diagrams**: Each topological feature is represented as a point $(b, d)$ where $b$ is the birth threshold and $d$ is the death threshold. The **persistence** $d - b$ measures feature significance.
4. **Barcodes**: An equivalent representation shows each feature as a horizontal bar from birth to death, with longer bars indicating more persistent (significant) features.

### Optimal Threshold Selection

The visualization uses persistent homology to automatically recommend similarity thresholds:

- **H₀ Threshold** (green marker): The threshold with the largest persistence gap in connected components, indicating the most stable clustering structure.
- **H₁ Threshold** (red marker): The threshold where the most significant 1-dimensional cycle dies, indicating when the network transitions from containing loops to being more tree-like.

The default threshold is set to the **H₁ optimal** (currently 0.75), as this represents the point where significant topological structure stabilizes.

---

## Data Pipeline

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   PDF Documents │────▶│  Text Extraction│────▶│    PostgreSQL   │
│   (VOL001-008)  │     │  (OCR + Parse)  │     │    Database     │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                                                        │
                        ┌───────────────────────────────┴───────────┐
                        ▼                                           ▼
               ┌─────────────────┐                         ┌─────────────────┐
               │   NER Pipeline  │                         │  TDM Builder    │
               │   (spaCy NLP)   │                         │  (Noun/Verb)    │
               └────────┬────────┘                         └────────┬────────┘
                        │                                           │
                        ▼                                           ▼
               ┌─────────────────┐                         ┌─────────────────┐
               │ Entity Network  │                         │ Document        │
               │ (142 entities,  │                         │ Similarity      │
               │  224 relations) │                         │ (471 docs)      │
               └────────┬────────┘                         └────────┬────────┘
                        │                                           │
                        │     ┌─────────────────────────────┐       │
                        └────▶│   Persistent Homology       │◀──────┘
                              │   (ripser/GUDHI/native)     │
                              └─────────────┬───────────────┘
                                            │
                                            ▼
                              ┌─────────────────────────────┐
                              │   Static JSON Export        │
                              │   (export_static_data.py)   │
                              └─────────────┬───────────────┘
                                            │
                                            ▼
                              ┌─────────────────────────────┐
                              │   D3.js Visualization       │
                              │   (GitHub Pages)            │
                              └─────────────────────────────┘
```

### Key Scripts

| Script                                   | Purpose                                       |
| ---------------------------------------- | --------------------------------------------- |
| `extract_pdfs.py`                      | OCR and text extraction from PDF documents    |
| `load_extracted_text.py`               | Load extracted text into PostgreSQL           |
| `build_noun_tdm_postgres.py`           | Build noun term-document matrix               |
| `build_verb_tdm_postgres.py`           | Build verb term-document matrix               |
| `build_document_similarity_network.py` | Compute similarities and persistent homology  |
| `build_entity_network.py`              | Extract and canonicalize entity relationships |
| `export_static_data.py`                | Generate static JSON files for GitHub Pages   |

---

## Usage Guide

### Navigation

1. **Pan**: Click and drag on empty space
2. **Zoom**: Mouse wheel or pinch gesture
3. **Select Node**: Click on a document node to view details
4. **Entity Overlay**: Click selected node again to show entity halo
5. **Reset**: Click away from nodes or click the selected node a third time

### Controls

- **Similarity Threshold**: Adjust the slider to filter edges by minimum similarity. Higher thresholds show only the strongest connections.
- **Time Period**: Select a temporal period to filter documents by date range.
- **View Toggle**: Switch between Document Similarity Network and Entity Network views.

### Reading the Visualization

- **Node Size**: Larger nodes have more connections (higher degree centrality)
- **Node Color**: Indicates community membership (Louvain clustering)
- **Edge Opacity**: Stronger similarity = more opaque edges
- **Green Marker (H₀)**: Threshold where cluster structure is most stable
- **Red Marker (H₁)**: Threshold where significant loops persist

---

## Network Statistics

| Metric             | Document Network | Entity Network |
| ------------------ | ---------------- | -------------- |
| Nodes              | 471 documents    | 142 entities   |
| Edges (at optimal) | 547              | 224            |
| Communities        | 57               | —             |
| Modularity         | 0.74             | —             |
| Threshold Range    | 0.20 – 0.95     | N/A            |
| H₀ Optimal        | 0.20             | —             |
| H₁ Optimal        | 0.75             | —             |

---

## Deployment

### GitHub Pages

The visualization is designed for static hosting on GitHub Pages:

```bash
# Generate static data from PostgreSQL
cd scripts
python export_static_data.py

# The static-viz directory is ready for deployment
# Push to gh-pages branch or configure GitHub Pages to serve from /static-viz
```

### Local Development

```bash
# Serve locally
cd static-viz
python -m http.server 8080

# Open http://localhost:8080
```

### Directory Structure

```
static-viz/
├── index.html              # Main visualization (single-page app)
├── README.md               # This file
└── data/
    ├── thresholds.json     # Available thresholds with H₀/H₁ markers
    ├── temporal-periods.json
    ├── entity-network.json
    ├── entity-mentions-index.json
    ├── networks/           # Network data at each threshold
    │   ├── network_0.20.json
    │   ├── network_0.25.json
    │   └── ...
    ├── documents/          # Chunked document details
    │   ├── docs_0001.json
    │   └── ...
    ├── overlays/           # Entity overlay data
    │   ├── overlays_0001.json
    │   └── ...
    └── periods/            # Temporal period document lists
        ├── period_*.json
        └── ...
```

---

## References

### Persistent Homology & Topological Data Analysis

1. Edelsbrunner, H., & Harer, J. (2010). *Computational Topology: An Introduction*. American Mathematical Society.
2. Carlsson, G. (2009). Topology and data. *Bulletin of the American Mathematical Society*, 46(2), 255-308. https://doi.org/10.1090/S0273-0979-09-01249-X
3. Ghrist, R. (2008). Barcodes: The persistent topology of data. *Bulletin of the American Mathematical Society*, 45(1), 61-75. https://doi.org/10.1090/S0273-0979-07-01191-3
4. Zomorodian, A., & Carlsson, G. (2005). Computing persistent homology. *Discrete & Computational Geometry*, 33(2), 249-274. https://doi.org/10.1007/s00454-004-1146-y
5. Otter, N., Porter, M. A., Tillmann, U., Grindrod, P., & Harrington, H. A. (2017). A roadmap for the computation of persistent homology. *EPJ Data Science*, 6(1), 17. https://doi.org/10.1140/epjds/s13688-017-0109-5

### Network Analysis

6. Newman, M. E. J. (2010). *Networks: An Introduction*. Oxford University Press.
7. Blondel, V. D., Guillaume, J. L., Lambiotte, R., & Lefebvre, E. (2008). Fast unfolding of communities in large networks. *Journal of Statistical Mechanics: Theory and Experiment*, 2008(10), P10008. https://doi.org/10.1088/1742-5468/2008/10/P10008
8. Wasserman, S., & Faust, K. (1994). *Social Network Analysis: Methods and Applications*. Cambridge University Press.

### Text Analysis & Information Retrieval

9. Manning, C. D., Raghavan, P., & Schütze, H. (2008). *Introduction to Information Retrieval*. Cambridge University Press.
10. Salton, G., & Buckley, C. (1988). Term-weighting approaches in automatic text retrieval. *Information Processing & Management*, 24(5), 513-523. https://doi.org/10.1016/0306-4573(88)90021-0

### Software Libraries

11. **ripser**: Tralie, C., Saul, N., & Bar-On, R. (2018). Ripser.py: A lean persistent homology library for Python. *Journal of Open Source Software*, 3(29), 925. https://doi.org/10.21105/joss.00925
12. **GUDHI**: Maria, C., Boissonnat, J. D., Glisse, M., & Yvinec, M. (2014). The GUDHI library: Simplicial complexes and persistent homology. In *International Congress on Mathematical Software* (pp. 167-174). Springer.
13. **D3.js**: Bostock, M., Ogievetsky, V., & Heer, J. (2011). D3: Data-driven documents. *IEEE Transactions on Visualization and Computer Graphics*, 17(12), 2301-2309. https://doi.org/10.1109/TVCG.2011.185
14. **spaCy**: Honnibal, M., & Montani, I. (2017). spaCy 2: Natural language understanding with Bloom embeddings, convolutional neural networks and incremental parsing. https://spacy.io/

---

## License

This work is released under the **MIT License**.

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files, to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED.

---

## Acknowledgments

This visualization is part of the broader [Epstein-Maxwell Files](https://github.com/yourusername/epstein-maxwell-files) computational analysis project. The underlying document corpus consists of legal documents released through the EFTA (Epstein Files Text Archive) collection.

For questions or contributions, please open an issue on GitHub.
