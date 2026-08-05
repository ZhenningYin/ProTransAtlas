# ProTransAtlas

A prokaryotic transposon evidence atlas — database, web application, and ML classifier for transposon protein identification from ESM2 embeddings.

## Overview

ProTransAtlas integrates HMM-based transposon annotation with ESM2 protein language model embeddings to enable large-scale transposon identification, genomic context analysis, and machine learning-based classification.

| Component | Scale |
|-----------|-------|
| HMM hits | 7,069,013 |
| Unique proteins | 7,060,499 |
| Contigs with genomic context | 3,215,316 |
| HMM models | 142 (123 unique descriptions) |
| Training labels | 7,060,499 (protein-label-v1.0) |
| Curated negatives (easy) | 14,267 Swiss-Prot core cellular proteins |
| Hard negatives | 18,360 DNA-interacting proteins |
| ESM2 embeddings | 695,084 (320-dim, 37 families + 2 negative sets) |
| Transposon families | 37 |
| Database | DuckDB, 14 GB |

## Quick Start

### Web Application

```bash
pip install -r requirements.txt
uvicorn backend.app:app --host 0.0.0.0 --port 8000
```

Open `http://localhost:8000`.

### Classification Prediction

```bash
# 1. Download ESM2 model (~32 MB, one-time)
huggingface-cli download facebook/esm2_t6_8M_UR50D --local-dir ./esm2_t6_8M_UR50D

# 2. Predict directly (pre-trained model included)
python classification/predict.py -s "MFLKTVESYRQG..." -m ./esm2_t6_8M_UR50D
python classification/predict.py -f proteins.faa -m ./esm2_t6_8M_UR50D
```

See [`classification/README.md`](classification/README.md) for full classifier documentation.

### Database Build

```bash
python scripts/build_db.py
python scripts/label_training_data.py
python scripts/build_curated_negatives.py --max-per-group 5000
python scripts/build_hard_negatives.py --max-per-group 5000
```

Raw input files are read from `/home/yin/Project/01_Main/0002_TranDB/02_RawData` by default.

## Web Interface

| Route | Description |
|-------|-------------|
| `/` | Dashboard — database stats, top models |
| `/search` | Full-text search across proteins, families, annotations |
| `/hits` | HMM hit browser with evidence/category filters |
| `/protein/{id}` | Protein detail — HMM hits, coordinates, cluster, flanking DNA |
| `/models` | HMM model catalog |
| `/labels` | Training label overview |
| `/clusters` | MMseqs2 cluster browser |
| `/negatives` | Curated negative reference browser |
| `/context` | Genomic context browser — transposon hits with ±1000bp flanking DNA |
| `/blast` | BLAST search against 670K cluster representatives |
| `/predict` | ML classifier — predict transposon family from protein sequence |
| `/summary` | Statistical summaries |
| `/download` | Data downloads (Parquet, TSV, FASTA) |

### API Endpoints

| Route | Content |
|-------|---------|
| `/api/stats` | Database statistics |
| `/api/summary/evidence` | Evidence level summary |
| `/api/summary/category` | Category summary |
| `/api/labels/summary` | Training label summary |
| `/api/negatives/curated/summary` | Curated negative summary |
| `/api/proteins/search?q=` | Protein search |
| `/api/proteins/{id}` | Protein detail |
| `/api/context/{contig_id}` | All transposon hits on a contig |
| `/api/context/search?q=&family=` | Search hits with flanking DNA |
| `POST /api/predict` | Classify protein sequence (form: `sequence`) |

## Database Schema

Core tables:

| Table | Rows | Description |
|-------|------|-------------|
| `hmm_hits` | 7,069,013 | HMM scan results with evidence levels |
| `proteins` | 10,651,187 | Protein coordinates (contig, start, end, strand) |
| `protein_sequences` | 10,651,187 | Protein amino acid sequences |
| `protein_embeddings` | 695,084 | ESM2 t6_8M 320-dim vectors |
| `protein_labels` | 7,060,499 | Training labels (8 roles) |
| `protein_clusters` | 2,784,330 | MMseqs2 cluster assignments |
| `curated_negative_reference` | 14,267 | Easy negatives (core cellular) |
| `hard_negative_reference` | 18,360 | Hard negatives (DNA-interacting) |
| `transposon_hits` | 7,060,499 | Hit proteins with ±1000bp flanking DNA |
| `family_literature` | 70 | Curated literature references per family |

## Classification Results

### Binary Classification (Transposon vs Non-transposon)

| Negative Set | Model | AUROC | AUPRC | F1 |
|-------------|-------|-------|-------|-----|
| Easy (core cellular, n=14,267) | RandomForest | 0.996 | 0.999 | 0.982 |
| Hard (DNA-interacting, n=18,360) | LogisticRegression | 0.945 | 0.980 | 0.910 |

### Multi-class (5 Classes, 10-fold CV)

| Class | F1 | Precision | Recall | Support |
|-------|----|-----------|--------|---------|
| Non-transposon | 0.882 | 0.887 | 0.876 | 18,360 |
| IS Family Transposase | 0.719 | 0.750 | 0.689 | 17,000 |
| REP Tyr Recombinase | 0.757 | 0.803 | 0.716 | 5,000 |
| TnsA/Tn7-like | 0.602 | 0.548 | 0.667 | 4,832 |
| Tn3 Family | 0.490 | 0.446 | 0.543 | 5,000 |

**Best model**: RandomForest (200 trees, max_depth=20), Macro-F1 = 0.690, 10-fold CV.

## Project Structure

```
ProTransAtlas/
├── backend/              # FastAPI web application
│   ├── app.py            # Routes and business logic
│   ├── database.py       # DuckDB connection helper
│   └── templates/        # Jinja2 HTML templates (18 pages)
├── scripts/              # Data pipeline and ETL
│   ├── build_db.py       # Build DuckDB from raw TSV
│   ├── build_curated_negatives.py   # Build easy negative set
│   ├── build_hard_negatives.py      # Build hard negative set
│   ├── build_transposon_context.py  # Extract genomic context
│   ├── encode_curated_negatives.py  # ESM2 encoding script
│   ├── evaluate_classifiers.py      # 10-fold CV comparison
│   ├── extract_flanks_only.py       # Remote flanking DNA extraction
│   ├── label_training_data.py       # Protein-level weak labels
│   └── ...
├── classification/       # ML classifier package
│   ├── predict.py        # Standalone prediction script
│   ├── train_and_save_model.py  # Train classifier from database
│   ├── models/           # Serialized model files
│   └── README.md         # Classifier documentation
├── data/                 # DuckDB, Parquet, FASTA, models (git-ignored)
├── docs/                 # Labeling and negative reference docs
├── knowledge/            # Project state and knowledge base
├── results/              # Experiment results and figures
├── Dockerfile
├── docker-compose.yml
└── requirements.txt
```

## Requirements

```
fastapi
uvicorn
duckdb
jinja2
python-multipart
numpy
scikit-learn
matplotlib
torch
transformers
joblib
biopython
tqdm
```

## References

- Lin Z, Akin H, Rao R, et al. Evolutionary-scale prediction of atomic-level protein structure with a language model. *Science*. 2023;379(6637):1123-1130.
- Siguier P, Gourbeyre E, Chandler M. Bacterial insertion sequences: their genomic impact and diversity. *FEMS Microbiol Rev*. 2014;38(5):865-891.
